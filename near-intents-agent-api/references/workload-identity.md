# Workload identity federation

A grant token (`ngt_…`) is a long-lived bearer secret: whoever holds it acts for up to 365 days. A
workload that already has a platform identity (a CI job, a cloud VM, a pod) or can hold a private
key that never leaves it does not need one. The owner binds the grant to that identity or key
instead, and the workload proves itself with a JWT that lives minutes. Only the grant's `credential`
changes: the policy, timelock, approvals, budget, idempotency and receipts see the grant, never the
credential.

> Support is in the API and the TypeScript SDK (`@near-intents-agent-api/sdk`, methods named below).
> **No platform here has been tested live.** The issuer settings (URLs, key hosts, pinned claims,
> lifetimes) follow each vendor's documentation, and the recipes follow the vendors' documented token
> endpoints. Known unconfirmed: the EKS dual-stack JWKS host, the GKE JWKS host, GitHub's ID token
> lifetime and the Entra managed-identity token lifetime. A wrong host fails closed with
> `issuer_unavailable`. Smoke-test each platform with a dry call before relying on it.

## Choose the credential

`grant_issue.credential` is one of three forms. The owner signs it; the grant view then shows
`credential_type: "token" | "oidc" | "public_key"`.

| `credential` | The workload holds | It sends | Choose it for |
|---|---|---|---|
| SHA-256 hex of an `ngt_…` token | The token, for the grant's life | The token as `X-Grant-Token` | Your own backend, which already keeps secrets: one grant per chat session or assistant |
| `{ type: "oidc", issuer, claims, audience?, jwks? }` | Nothing: its platform issues identity tokens | An `ngs_…` session token as `X-Grant-Token` | **CI and cloud workloads**: GitHub Actions, GitLab CI, Google Cloud, AWS, Kubernetes, Entra ID, Vercel |
| `{ type: "public_key", jwk }` | A private key (file, TPM, HSM, cloud KMS) | An `ngs_…` session as `X-Grant-Token`, **or** a proof it signs per request as `X-Grant-Proof` | Self-hosted servers, VPS, bare metal, laptops; any place with no platform identity |

Prefer a federated grant for anything that runs in CI or a cloud: no long-lived secret sits on the
workload or crosses the wire. Keep `token` for the cases above. Within `public_key`, send a **proof**
when no bearer token should exist anywhere (not in memory, logs or a proxy), and a **session** when
the key is slow or billed per signature (a cloud KMS) and the workload makes many calls: one
signature then covers 15 minutes.

The API key (`X-API-Key`) is still required on every call, the exchange included. Federation
replaces only the grant token. Give each workload its own partner API key so its limits and
revocation are its own.

## What the owner signs

| Form | Signed content | Notes |
|---|---|---|
| `oidc` | `issuer`, the audience, 1 to 16 pinned `claims` | **Leave `audience` out**: the API fills in `{API url}/v1/agents/{agent_id}` and refuses an explicit one ("leave audience out"). Only Microsoft Entra ID names it, and there it is **required** |
| `oidc` + `jwks` | the above plus up to 4 issuer public keys | For an issuer the API does not know (self-hosted Kubernetes). Keys are frozen into the grant |
| `public_key` | `jwk` | Ed25519 (`{kty:"OKP",crv:"Ed25519",x}`) or P-256 (`{kty:"EC",crv:"P-256",x,y}`). Public members only: a `d` is refused |

**Two audiences, do not mix them up.** The `audience` field of the grant credential is the audience
the owner signs: omitted for every issuer but Entra ID (the API sets it), required for Entra. The
audience a workload puts in its identity-token request is `oidcGrantAudience(baseUrl, agentId)`
(`{API url}/v1/agents/{agent_id}`, `$AUD` in the recipes below); for Entra it is the registered
audience the grant names instead.

`claims` values are printable-ASCII strings compared exactly with the token's claim (a number or
boolean compares as its JSON text: `"4242"`, `"true"`). A name pins a top-level claim; an RFC 6901
pointer pins a nested one (`/kubernetes.io/serviceaccount/uid`). Pin **immutable ids**, never names:
a repository, organization or service account can be renamed or recreated, and its old name taken by
someone else. You may pin more claims than required; every one must match.

The intent's `preview.summary` names everything the grant trusts (issuer URL, pinned claims,
audience, key thumbprints). Show it to the owner. A credential the exchange could never accept is
refused at `generate-intent` with `grant_credential_invalid` (400), before anything is signed.

Issue it like any grant:

```ts
await api.generateIntent({
  type: "grant_issue",
  agent_id: agentId,
  label: "CI deploy bot (production)",
  credential: {
    type: "oidc",
    issuer: "https://token.actions.githubusercontent.com",
    claims: { repository_owner_id: "123", repository_id: "456", environment: "production" },
  },
  expires_at: new Date(Date.now() + 30 * 86_400_000).toISOString(),
});
// owner signs, submit, wait: status.details.grant.grant_id
```

Renewing: issue the **same** credential again before the old grant ends. Grants binding the same
credential are one identity, and an exchange opens its session on the one that lives longest. Other
overlapping bindings on one agent are allowed, and a token that several of them accept needs
`grant_id` (`grant_ambiguous`). Revoke the grant when you delete the role, service account or
cluster it names (a recreated one with the same name can match it).

### Finding the `grant_id`

An exchange finds the grant itself; only a proof (and an ambiguous exchange) needs the id.

- From the submit result: `GET /v1/status` for the `grant_issue` correlation id, `details.grant.grant_id`.
  Store it with the grant.
- Later: `GET /v1/agents/{agent_id}/grants` (`api.listGrants(agentId)`): pick the row with
  `credential_type: "public_key"`, `revoked_at: null`, not expired, whose `owner_message.credential`
  holds your public JWK.
- Keep the **original** id fixed on retries and recovery, also after a same-key renewal.

## Exchange and sessions

`POST /v1/agents/{agent_id}/grants/token` (RFC 8693 token exchange, JSON body, `X-API-Key` required)
trades a JWT for a session token. Wire format and a curl example: [http-api.md](http-api.md#workload-identity-exchange-and-proofs).

- **Session token** `ngs_…`: 15 minutes by default (up to 60), never past the grant. It goes in
  `X-Grant-Token`, exactly where an `ngt_…` goes. Revoking the grant, changing its owner or letting
  it expire ends every session of it at once.
- **Reuse it.** Exchange again from about a minute before `expires_at` (at most half the session's
  life), but keep using the current token until 5 s before its end, so a short issuer outage costs
  nothing. After a failed exchange wait about 5 s before the next. A grant holds at most 100 live
  sessions (`grant_session_limit_reached`), so never exchange per request, and keep one client per grant.
- **A refused session is dropped, not retried.** If a delegated call answers `agent_grant_required`
  (401/403) the grant behind the session was revoked, replaced or expired: forget the session, do
  not repeat that call automatically, exchange afresh on the next call, and ask the owner if that
  fails too.
- An OIDC identity token may be exchanged more than once while it lives (platforms cache them). A
  key assertion is single-use: sign a new one for every exchange.
- Never both headers: `X-Grant-Token` and `X-Grant-Proof` together are `validation_failed` (400).

TypeScript does all of that for you:

```ts
import { DEFAULT_BASE_URL, oidcGrantAudience } from "@near-intents-agent-api/sdk";

const audience = oidcGrantAudience(DEFAULT_BASE_URL, agentId);   // for the workload's token request, not for the grant

const bot = api.forGrantSession(agentId, {
  subjectToken: () => requestPlatformToken(audience),            // called per exchange: read it fresh
  // grantId: "<64 hex>",                                         // only when several grants accept it
});
await bot.transfer(agentId, { asset: "nep141:wrap.near", amount: "1", recipient: "bob.near" }, { idempotencyKey });
```

`bot` exchanges on its first delegated call and keeps the session. From 60 s before its end (never
more than half its life) it renews in the background while the current token keeps serving until 5 s
before the end; a failed exchange is not repeated for 5 s. Concurrent calls share one exchange. A call
the API refuses with `agent_grant_required` throws, is never repeated, and drops the cached session so
the next call exchanges again. Keep one client per grant (a client per invocation burns the grant's 100 sessions). `baseUrl` must be the URL the
API publishes as `servers[0]` in `/openapi.json` (`https://api.agentsonintents.com`, or
`https://api.demo.agentsonintents.com` on the demo): audiences are built from it, and an alias that
reaches the same host does not match. In other languages, exchange yourself, cache the token until
shortly before `expires_at`, and send it as `X-Grant-Token`
([python.md](python.md#workload-identity), [rust.md](rust.md#workload-identity),
[other-languages.md](other-languages.md#workload-identity)).

## Platform recipes

In every recipe the grant credential has **no `audience`** (except Entra ID). The audience the
*workload* requests its token for is `AUD="$API_URL/v1/agents/$AGENT_ID"` (`API_URL` as above, no
trailing slash; for Entra, the audience the grant names). Then exchange the token.

| Platform | `issuer` | Must pin | Plus one of |
|---|---|---|---|
| GitHub Actions | `https://token.actions.githubusercontent.com` | `repository_owner_id`, `repository_id` | `ref` (full: `refs/heads/main`), `environment`, `job_workflow_ref`, `workflow_ref` |
| GitLab CI (gitlab.com) | `https://gitlab.com` | `namespace_id`, `project_id`; `ref_type` whenever `ref` is pinned | `ref`, `environment`, `ci_config_ref_uri` |
| Google Cloud | `https://accounts.google.com` | `sub` (the service account's unique id) | |
| AWS IAM outbound federation | `https://<account-uuid>.tokens.sts.global.api.aws` | `sub` (the role ARN) | |
| Amazon EKS | the cluster's `oidc.eks…` issuer URL | `sub`, `/kubernetes.io/serviceaccount/uid` | |
| Google Kubernetes Engine | `https://container.googleapis.com/v1/projects/<p>/locations/<l>/clusters/<c>` | `sub`, `/kubernetes.io/serviceaccount/uid` | |
| Azure Kubernetes Service | the cluster's `…oic.prod-aks.azure.com/…/` issuer URL | `sub`, `/kubernetes.io/serviceaccount/uid` | |
| Microsoft Entra ID | `https://sts.windows.net/<tenant>/` (v1.0) or `https://login.microsoftonline.com/<tenant>/v2.0` | `oid` (`idtyp` = `app` is recommended, not required) | |
| Vercel | `https://oidc.vercel.com/<team_slug>` | `owner_id`, `project_id` | `environment` |
| Self-hosted Kubernetes, any other `https` issuer | the issuer as its tokens state it | `sub` (+ the service account uid for a cluster) | plus `jwks` signed by the owner |

A shared issuer (GitHub, GitLab) signs for every customer, so a binding must also pin a branch,
environment or workflow. Prefer `environment` with protection rules for production: reviewers gate it.

**GitHub Actions** (read the ids from `GET https://api.github.com/repos/{owner}/{repo}`: `owner.id`, `id`)

```yaml
permissions:
  id-token: write          # lets the job request an OIDC token
jobs:
  act:
    environment: production
    steps:
      - run: |
          AUD="$API_URL/v1/agents/$AGENT_ID"
          JWT=$(curl -sSf -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
            "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=$AUD" | jq -r .value)
          echo "::add-mask::$JWT"
          # POST $JWT to /v1/agents/$AGENT_ID/grants/token, then use the ngs_ token
```

**GitLab CI**: declare an ID token with the audience; GitLab exposes it as an environment variable.

```yaml
job:
  id_tokens:
    AGENT_ID_TOKEN:
      aud: https://api.agentsonintents.com/v1/agents/<agent_id>
```

A GitLab token lives for the job's timeout (5 minutes when none is set); a job whose timeout exceeds
a day cannot exchange.

**Google Cloud** (Compute Engine, Cloud Run, GKE Workload Identity, any impersonated service account)

```sh
curl -sSf -H "Metadata-Flavor: Google" \
  "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=$AUD&format=full"
```

Pin the service account's unique id (`gcloud iam service-accounts describe … --format='value(uniqueId)'`).
Pinning `email` alone is refused: a deleted account's email can be reused, its id cannot.

**AWS** (Lambda, EC2, ECS, EKS with AWS credentials). Turn on outbound identity federation in IAM
(Account settings): it shows the account's issuer URL (its host is the account's lowercase UUID). The role needs
`sts:GetWebIdentityToken`:

```sh
aws sts get-web-identity-token --region us-east-1 --audience "$AUD" \
  --signing-algorithm RS256 --duration-seconds 900
```

Grant: `{ type: "oidc", issuer: "https://<uuid>.tokens.sts.global.api.aws", claims: { sub: "arn:aws:iam::123456789012:role/agent" } }`.

**Kubernetes** (EKS, GKE, AKS, self-hosted). Project a service account token with the audience, and
read the file before each exchange (the kubelet rotates it). Keep `expirationSeconds` at 86400 or less.

```yaml
volumes:
  - name: agent-token
    projected:
      sources:
        - serviceAccountToken:
            audience: https://api.agentsonintents.com/v1/agents/<agent_id>
            expirationSeconds: 3600
            path: token
```

Pin `sub` (`system:serviceaccount:<namespace>:<name>`) and `/kubernetes.io/serviceaccount/uid`
(`kubectl get serviceaccount <name> -o jsonpath='{.metadata.uid}'`), so a recreated account does not
inherit the grant. EKS, GKE and AKS: use the cluster's issuer URL and the API discovers the keys.
**Self-hosted** (k3s, kubeadm, kind, on-prem): the issuer is often not reachable, so the owner signs
the cluster's public keys into the grant (`kubectl get --raw /openid/v1/jwks`) as `jwks`: up to 4 keys
(RSA 2048+ bits, P-256, P-384 or Ed25519), public members only, each with the `kid` the tokens name.
Nothing is fetched, and the keys are frozen: when the cluster rotates its key, revoke and issue anew.
Signed keys are refused for issuers on a vendor's domain.

**Microsoft Entra ID** (VM, App Service, Functions, Container Apps managed identity; AKS Workload ID).
Create an app registration that stands for this API alone and give it an Application ID URI
(`api://near-intents-agents`). Every principal in the tenant, users included, can get a token for it,
so pin `oid` (the identity's object id; the API requires only this). Add the `idtyp` optional claim
and also pin `idtyp` = `app` so a user's token can never match: recommended, because Entra emits that
claim only when the app registration configures it. The audience depends on the token version:

| Token | `issuer` | `audience` the owner signs |
|---|---|---|
| v1.0 (default for an app registration; `aud` is the resource asked for) | `https://sts.windows.net/<tenant>/` | **required**: the Application ID URI (`api://…`) or the client id |
| v2.0 (`accessTokenAcceptedVersion` = 2; `aud` is always the client id) | `https://login.microsoftonline.com/<tenant>/v2.0` | **required**: the app registration's **client id** |

```sh
curl -sSf -H Metadata:true \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=api://near-intents-agents"
```

A grant whose audience its tokens can never carry is refused at generate time.

**Vercel**: the token's issuer is the team that owns the deployment, so use the team-scoped issuer.

```ts
import { getVercelOidcToken } from "@vercel/oidc";
const jwt = await getVercelOidcToken({ audience: `${apiUrl}/v1/agents/${agentId}` });
```

## Keys the workload holds

Generate an Ed25519 or P-256 key where it will live, keep the private half there, and have the owner
sign a `public_key` grant over the public JWK (P-256 suits a TPM, HSM, AWS KMS and Google Cloud KMS;
Ed25519 suits a key file).

```ts
import { createWorkloadKey, localJwkSigner } from "@near-intents-agent-api/sdk";

const { privateJwk, publicJwk } = createWorkloadKey("Ed25519");   // or "P-256"
// grant_issue with credential: { type: "public_key", jwk: publicJwk }; owner signs; keep grant_id
const signer = localJwkSigner(privateJwk);                        // load privateJwk from your secret store

const viaSession = api.forGrantSession(agentId, { signer });      // signs exchange assertions, refreshes
const viaProof   = api.forGrantProof({ grantId, signer });        // signs every request, no bearer token
```

`forGrantProof` sends `X-Grant-Proof` instead of `X-Grant-Token`. Each attempt, a retry included,
signs a new proof (a proof is spent on first use); a retry keeps the same `Idempotency-Key` and gets
the stored result. The proof binds one grant, method, URL (no query), exact body bytes and
`Idempotency-Key`. The claim list for hand-built proofs is in
[http-api.md](http-api.md#signing-every-request-x-grant-proof).

### A key in a KMS or HSM

The private key never leaves the KMS. Give the SDK your own `WorkloadSigner`:
`{ publicJwk, sign(signingInput: Uint8Array): Uint8Array | Promise<Uint8Array> }`. `sign` receives the
JWS signing input (`base64url(header).base64url(claims)` as UTF-8 bytes) and returns the **raw
64-byte** JWS signature. Ed25519: the signature over those bytes. P-256: ECDSA over their SHA-256 as
`r || s`, **not the ASN.1 DER a KMS returns** (the SDK throws on anything but 64 bytes). Convert it:

```ts
import { createHash, createPublicKey } from "node:crypto";
import { KeyManagementServiceClient } from "@google-cloud/kms";
import type { GrantPublicKeyJwk, WorkloadSigner } from "@near-intents-agent-api/sdk";

const kms = new KeyManagementServiceClient();
const name = "projects/<p>/locations/<l>/keyRings/<r>/cryptoKeys/<k>/cryptoKeyVersions/1"; // EC_SIGN_P256_SHA256

/** DER ECDSA signature -> the raw r || s a JWS carries (P-256: two 32-byte halves). */
function derToRaw(der: Uint8Array): Uint8Array {
  if (der[0] !== 0x30) throw new Error("not a DER ECDSA signature");
  const raw = new Uint8Array(64);
  let offset = 2;
  for (const half of [0, 32]) {
    if (der[offset] !== 0x02) throw new Error("not a DER ECDSA signature");
    const length = der[offset + 1] ?? 0;
    let value = der.subarray(offset + 2, offset + 2 + length);
    offset += 2 + length;
    while (value.length > 32 && value[0] === 0) value = value.subarray(1); // DER sign padding
    if (value.length > 32) throw new Error("not a P-256 signature");
    raw.set(value, half + 32 - value.length);
  }
  return raw;
}

const [{ pem }] = await kms.getPublicKey({ name });             // once: this JWK is what the owner signs
const publicJwk = createPublicKey(pem!).export({ format: "jwk" }) as GrantPublicKeyJwk;

const signer: WorkloadSigner = {
  publicJwk,
  sign: async (input) => {
    const [{ signature }] = await kms.asymmetricSign({
      name,
      digest: { sha256: createHash("sha256").update(input).digest() },   // this KMS algorithm takes the digest
    });
    return derToRaw(signature as Uint8Array);
  },
};
const api2 = api.forGrantSession(agentId, { signer });        // one KMS call per exchange, not per request
```

Other KMSs differ in what they sign (digest or data) and in how they return the signature; the
contract with the SDK is only "raw `r || s`" (P-256) or the raw Ed25519 signature. Each KMS signature
is a network call and a billed operation: use `forGrantSession(agentId, { signer })` unless no bearer token
may exist.

## Security

- **Prefer federated grants for CI and cloud workloads.** Use a token grant only where the backend
  already keeps secrets.
- **Never log, print, store in a repo, put in a prompt, a URL or a client bundle**: `ngt_…` and
  `ngs_…` tokens, platform identity JWTs, proofs, private JWKs and the API key. In CI mask the JWT
  (`::add-mask::`). A short-lived token leaked in a log is still usable for its lifetime.
- **KMS keys stay in the KMS.** Never export a key to make signing easier; wrap the KMS call in a
  `WorkloadSigner`. Restrict who can call `asymmetricSign` like you would who holds the key.
- **Pin immutable ids** (`repository_id`, project and namespace ids, service account unique id and
  uid, `oid`), and the narrowest of branch, environment or workflow. A compromised issuer or pinned
  workload (a malicious commit on the pinned branch) can act within the grant until it is revoked;
  the policy, timelock, approvals and budget still bound it.
- **One grant per workload** and per environment, labelled so the owner can revoke one alone.
- Keep the Entra app registration dedicated to this API. Revoke and reissue when an owner-signed
  `jwks` issuer rotates its keys.
- A federated grant is still the owner's decision: you cannot create or widen it, and the owner
  signs exactly what the workload may present.

Errors for every step are in [errors.md](errors.md#workload-identity).
