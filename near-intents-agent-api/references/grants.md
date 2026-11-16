# Grants

A grant is the owner's signed permission for **one actor** (a chat session, an assistant, a bot,
a device) to run money actions on the account. It says *who*; the policy says *what*. All grants
of an account share the same policy, limits and USD budget.

## Credential kinds

The owner signs the exact **credential** that may use the grant. Three kinds; the grant view
reports `credential_type: "token" | "oidc" | "public_key"`.

| `credential_type` | `grant_issue.credential` | The actor holds | It sends |
|---|---|---|---|
| `token` | SHA-256 hex of an `ngt_…` token (below) | the token, up to 365 days | `X-Grant-Token: ngt_…` |
| `oidc` | `{ type: "oidc", issuer, claims, jwks? }` (`audience` only for Microsoft Entra ID) | nothing: its platform (GitHub Actions, GitLab, Google Cloud, AWS, Kubernetes, Entra ID, Vercel) issues short-lived identity tokens | `X-Grant-Token: ngs_…`, a 15-minute session from `POST …/grants/token` |
| `public_key` | `{ type: "public_key", jwk }` (Ed25519 or P-256) | a private key that never leaves it (file, TPM, HSM, KMS) | `X-Grant-Token: ngs_…` from a signed assertion, **or** `X-Grant-Proof` signed per request |

**Choosing.** CI jobs and cloud workloads: `oidc`, so no long-lived secret exists. A server with no
platform identity, or one where no bearer token may exist: `public_key` (a proof per request; a
session when the key is a billed KMS and calls are many). Your own backend that already keeps
secrets, one grant per chat session or assistant: `token`. Prefer a federated grant over a token
wherever the workload can have one.

**What the owner signs.** For `oidc`: the issuer, the audience and 1 to 16 pinned claims (immutable
ids, never names). Leave `audience` out: the API fills in `{API url}/v1/agents/{agent_id}`, and
refuses an explicit one for every issuer except Microsoft Entra ID, where it is required. For
`public_key`: the public JWK (no private member). `generate-intent` refuses an unusable credential
with `grant_credential_invalid` before anything is signed. `owner_message.credential` in the grant
view is then the object the owner signed.

**`grant_id`.** An OIDC exchange finds its grant; a **proof** must name it: take `grant_id` from the
`grant_issue` result (`details.grant.grant_id`) or `GET /v1/agents/{id}/grants` (`credential_type:
"public_key"`, not revoked or expired), store it, and keep it fixed on retries. Send `grant_id` to
the exchange only on `grant_ambiguous`.

**Sessions and renewal.** A session (`ngs_…`) lives 15 minutes by default (60 at most), never past
the grant; revoking the grant, changing the owner or expiry ends all of them. Reuse one until about
a minute before `expires_at`. To renew, issue the same credential again before the old grant ends.
Never log or store `ngs_`/`ngt_` tokens, identity JWTs or proofs. Everything about exchange, platform
recipes and KMS signers: [workload-identity.md](workload-identity.md).

## Token and commitment

Your backend creates the token; the API only ever sees its hash.

```text
token      = "ngt_" + base64url_no_padding(32 cryptographically random bytes)   # 47 chars
credential = lowercase_hex(sha256(utf8(token)))                                  # hash the STRING, prefix included
```

| Language | Code |
|---|---|
| TypeScript | `const { token, commitment } = createGrantCredential()` (SDK) |
| Python | `"ngt_" + base64.urlsafe_b64encode(secrets.token_bytes(32)).rstrip(b"=").decode()`; `hashlib.sha256(token.encode()).hexdigest()` |
| Rust | `create_grant_credential()` → `(token, commitment)` in the asset crate |
| Go | `"ngt_" + base64.RawURLEncoding.EncodeToString(b)`; `hex.EncodeToString(sha256.Sum256([]byte(token)))` |

Use a **new token for every grant** (`grant_credential_reused` otherwise). Check your
implementation with `grant.token` / `grant.commitment` in `assets/test-vectors.json`.

## Issue

```json
POST /v1/generate-intent
{ "type": "grant_issue", "agent_id": "…", "label": "telegram-bot",
  "credential": "<sha256 hex>", "expires_at": "2027-04-01T00:00:00Z" }
```

- `credential`: the commitment above, or an `oidc` / `public_key` object ([Credential kinds](#credential-kinds)).
- `label`: what the owner sees in lists and previews; name the actor ("Claude Desktop on
  Alice's laptop", "support-bot prod").
- `expires_at`: required, at most 365 days ahead. Prefer short lives for sessions (hours/days)
  and renew.
- Owner signs → submit → `SUCCESS` → `details.grant.grant_id`.
- Limit: 50 live (unrevoked, unexpired) grants per agent → `grant_limit_reached`.

**Store the token before you ask the owner to sign**, encrypted (KMS envelope or app-level
AES-GCM), keyed by `grant_id` once known. If you lose it, the grant is useless: revoke it and
issue another. Never log it, never put it in an LLM prompt, URL or client bundle.

## Use

Send it as `X-Grant-Token` (or, for a `public_key` grant, `X-Grant-Proof` instead, never both) on swap, limit orders (place and cancel), withdraw, transfer, shield, recover (and
sign). One client per grant, so concurrent sessions never borrow each other's token:

```ts
const agentClient = api.forGrant(token);            // TS SDK; also takes an ngs_ session token
// federated: api.forGrantSession(agentId, { subjectToken | signer }) or api.forGrantProof({ grantId, signer })
```
```python
agent_client = api.for_grant(token)                  # Python asset
```

Every execution's `StatusResponse.grant` = `{ id, label }` shows which grant acted; use it in
activity feeds and audits.

## Revoke

```json
{ "type": "grant_revoke", "agent_id": "…", "grant_id": "<64 hex>" }
```

Takes effect for anything not yet dispatched. Operations already past the dispatch commit finish;
`details.committed_correlation_ids` lists them (`committed_truncated` if the list was cut).
`details.committed_cancellations` lists limit-order cancels made with the grant that may still reach
the provider (`{ order_id, committed_at }`, `order_id` null for cancel-all): read those orders. A limit
order already resting stays on the book: revoking the placing grant does not cancel it, and
cancelling needs a live grant (any of the account's), so keep one that can.
Expired grants stop working by themselves.

Audit: `GET /v1/agents/{id}/grants` (revoked included, each with the canonical `owner_message`
the owner signed) and `GET /v1/agents/{id}/containment` (live grants + unfinished operations).

## Patterns

| Situation | Do |
|---|---|
| Per chat/assistant session | one grant per session, label with the session, expire in hours/days |
| Long-running bot | one grant per bot deployment, rotate (issue new, then revoke old) every N months |
| CI job or cloud workload | `oidc` grant pinning its immutable ids; no token to store ([workload-identity.md](workload-identity.md)) |
| Third-party integration (MCP client, partner tool) | its own grant so it can be cut off alone |
| User says "stop that assistant" | `grant_revoke` that one grant |
| User says "stop everything" | `agent_freeze` |
| Agent needs a new capability | `policy_update`; do **not** issue a new grant (grants carry no permissions) |

Errors: `agent_grant_required` (403: missing, unknown, revoked, expired or other agent's token),
`grant_limit_reached`, `grant_credential_reused`, `grant_not_found`; federated grants add
`grant_credential_invalid`, `subject_token_invalid`, `grant_proof_invalid`, `grant_ambiguous`,
`grant_session_limit_reached` and `issuer_unavailable` ([errors.md](errors.md#workload-identity)).
