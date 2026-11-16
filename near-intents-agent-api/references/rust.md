# Rust

No crate is published. Copy [../assets/rust/](../assets/rust/) into your workspace (or vendor
`src/lib.rs` into an existing crate and merge the dependencies from `Cargo.toml`).

```bash
cd assets/rust
cargo test                    # NEAR + grant vectors
cargo test --features evm     # + EIP-712 vector (pulls in alloy)
```

Key dependencies: `reqwest` (rustls), `serde_json` **with `preserve_order`** (required: NEP-366
args must be serialized in the API's key order), `ed25519-dalek`, `sha2`, `bs58`, `base64`;
optional `alloy-*` behind feature `evm`.

## Client

```rust
use near_intents_agent_api::{AgentApi, Error, create_idempotency_key};
use serde_json::json;

let api = AgentApi::from_env()?;                    // NEAR_INTENTS_AGENT_API_KEY, NEAR_INTENTS_AGENT_API_URL (default hosted)
// or AgentApi::new("<partner API key>", None)?

let network = api.get_network().await?;
let tokens = api.get_tokens().await?;
let agent = api.get_agent(&agent_id).await?;
let policy = api.get_policy(&agent_id).await?;
let balances = api.get_balances(&agent_id, "confidential").await?;   // "public" reads the legacy balance
```

Bodies and responses are `serde_json::Value`. For anything without a helper use
`api.request(Method, path, query, body, idempotency_key, grant)`. Generate typed models from
`/openapi.json` (e.g. `openapi-generator -g rust` or `progenitor`) if you want structs; keep
`intent.payload` as `Value` either way so you never re-order it.

## Owner flow

```rust
use near_intents_agent_api::{NearKey, sign_intent};

let key = NearKey::from_secret(&std::env::var("AGENT_OWNER_PRIVATE_KEY")?)?;
let owner = json!({ "type": "near", "account_id": account_id, "public_key": key.public_key() });

let generated = api.generate_intent(&json!({
    "type": "agent_create", "name": "Treasury bot", "owner": owner, "policy": policy
}), &create_idempotency_key()).await?;

let signed = sign_intent(&generated, Some(&key), "https://free.rpc.fastnear.com", None).await?;
let cid = generated["correlation_id"].as_str().unwrap();
api.submit_intent(generated["type"].as_str().unwrap(), cid, signed).await?;
let status = api.wait_for_status(cid, std::time::Duration::from_secs(300)).await?;
```

EVM owner (feature `evm`): `sign_intent(&generated, None, "", Some(&evm_private_key_hex))`.
Passkeys can only be signed in a browser.

## Grant and execute

```rust
use near_intents_agent_api::create_grant_credential;

let (token, commitment) = create_grant_credential();      // store token encrypted
// grant_issue with "credential": commitment … owner signs … wait SUCCESS

let agent = api.for_grant(token)?;
let quote = agent.quote_swap(&agent_id, json!({
    "origin_asset": "nep141:wrap.near", "destination_asset": usdc, "amount": "100000000000000000000000"
})).await?;

let key = create_idempotency_key();                        // persist with the body first
let op = agent.execute(&agent_id, "swap", &json!({
    "origin_asset": "nep141:wrap.near", "destination_asset": usdc, "amount": "100000000000000000000000"
}), &key).await?;
let final_status = api.wait_for_status(op["correlation_id"].as_str().unwrap(), std::time::Duration::from_secs(300)).await?;
```

`execute` covers `swap`, `limit-orders` (a live order), `withdraw`, `transfer`, `shield` (and `unshield`, which the API always refuses);
`deposit` has its own method (no grant). Limit orders also have `quote_limit_order` (dry, no
grant), `list_limit_orders`, `get_limit_order`, `cancel_limit_order` and `cancel_all_limit_orders`
(grant, no key; repeat until `complete`); see [limit-orders.md](limit-orders.md).

## Workload identity

For a federated grant (`oidc` or `public_key`; which one and the platform recipes:
[workload-identity.md](workload-identity.md)) the asset has no dedicated methods. `for_grant` accepts an
`ngs_` session token like an `ngt_`, and the exchange is one `request`. The session lasts 15 minutes by
default (`expires_in` seconds): keep it and exchange again from about a minute before it ends (the old token still works until a few
seconds before its end, so a failed exchange there can wait; back off about 5 s after one; drop the
session on `agent_grant_required` and do not auto-retry), not per request.

```rust
use reqwest::Method;

let body = json!({
    "grant_type": "urn:ietf:params:oauth:grant-type:token-exchange",
    "subject_token": platform_jwt,        // a fresh identity token for {api url}/v1/agents/{agent_id}, or a signed key assertion
    "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
    // "grant_id": "<64 hex>",            // only when several live grants accept the token (grant_ambiguous)
});
let session = api.request(Method::POST, &format!("/v1/agents/{agent_id}/grants/token"), &[], Some(&body), None, false).await?;
let agent = api.for_grant(session["access_token"].as_str().unwrap())?;   // ngs_…, until session["expires_at"]
```

A key-bound grant can sign **a proof for every request** instead (no bearer token). The asset's
`request` sends no custom header, so call the endpoint with `reqwest` yourself and send the exact bytes
you hashed. Claims and rules: [http-api.md](http-api.md#signing-every-request-x-grant-proof). This is
Ed25519; for P-256 use the `p256` crate (`SigningKey::sign` → `Signature::to_bytes()` is the raw `r || s`
a JWS needs, not DER) with `"alg":"ES256"` and the `x`/`y` thumbprint members.

```rust
use std::time::{SystemTime, UNIX_EPOCH};

use base64::{engine::general_purpose::URL_SAFE_NO_PAD as B64URL, Engine};
use ed25519_dalek::{Signer, SigningKey};
use serde_json::{json, Value};
use sha2::{Digest, Sha256};

/// An Ed25519 workload key and the grant it is bound to. Cargo: `ed25519-dalek`, `sha2`, `base64`,
/// `hex`, `serde_json`, `uuid` (all already used by the asset crate).
pub struct ProofKey {
    signing: SigningKey,
    x: String, // the public JWK member the owner signed: base64url of the 32 public bytes
}

impl ProofKey {
    pub fn new(signing: SigningKey) -> Self {
        let x = B64URL.encode(signing.verifying_key().to_bytes());
        Self { signing, x }
    }

    /// The `iss` and `sub` of everything this key signs (RFC 9278 over the RFC 7638 thumbprint).
    pub fn thumbprint_uri(&self) -> String {
        let canonical = format!(r#"{{"crv":"Ed25519","kty":"OKP","x":"{}"}}"#, self.x);
        format!("urn:ietf:params:oauth:jwk-thumbprint:sha-256:{}", B64URL.encode(Sha256::digest(canonical)))
    }

    fn sign_jwt(&self, claims: &Value) -> String {
        let input = format!("{}.{}", B64URL.encode(br#"{"alg":"EdDSA"}"#), B64URL.encode(claims.to_string()));
        format!("{input}.{}", B64URL.encode(self.signing.sign(input.as_bytes()).to_bytes()))
    }

    /// One `X-Grant-Proof`. Single use: call it again (new `jti`) for every attempt, retries included.
    /// `url` is the request URL (a query or fragment is not signed); `body` the exact bytes you send.
    pub fn grant_proof(&self, grant_id: &str, method: &str, url: &str, body: &[u8], idempotency_key: Option<&str>) -> String {
        let iss = self.thumbprint_uri();
        let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
        let mut claims = json!({
            "iss": iss, "sub": iss, "grant_id": grant_id,
            "aud": url.split(['?', '#']).next().unwrap(),
            "htm": method.to_uppercase(),
            "body_sha256": hex::encode(Sha256::digest(body)),
            "iat": now, "exp": now + 60, "jti": uuid::Uuid::new_v4().to_string(),
        });
        if let Some(key) = idempotency_key {
            claims["idempotency_key"] = json!(key); // omit the claim when no header is sent
        }
        self.sign_jwt(&claims)
    }
}
```

```rust
let body = serde_json::to_vec(&json!({ "asset": "nep141:wrap.near", "amount": "1", "recipient": "bob.near" }))?; // once
let key = create_idempotency_key();                     // persist with the body first; the retry reuses it
let url = format!("{base_url}/v1/agents/{agent_id}/transfer");
let proof = proof_key.grant_proof(&grant_id, "POST", &url, &body, Some(&key));
let response = reqwest::Client::new().post(&url)
    .header("x-api-key", api_key).header("x-grant-proof", proof).header("idempotency-key", &key)
    .header("content-type", "application/json").body(body.clone()).send().await?;
// On a timeout or 5xx: same body, same key, same grant_id, and a NEW grant_proof(...) call.
```

A key in a KMS or HSM: replace `self.signing.sign(...)` with the KMS call and return the raw signature
(for P-256, convert the DER a KMS returns to `r || s`). The key never leaves the KMS; each signature
is a billed call, so prefer a session unless no bearer token may exist. Never log `ngs_`/`ngt_` tokens,
platform JWTs, proofs or private keys.

## Errors

```rust
match agent.execute(&agent_id, "withdraw", &body, &key).await {
    Ok(op) => track(op),
    Err(Error::Api { code, .. }) if code == "spend_budget_exceeded" => {
        /* show usage.budget (remaining_usd, resets_at); retry with a NEW key after that cap's
           resets_at, or once the owner raises it */
    }
    Err(e) if e.code() == Some("policy_destination_denied") => { /* ask the owner to allow it */ }
    Err(Error::Transport { idempotency_key, .. }) => {
        /* outcome unknown: retry with the same key and body */
    }
    Err(e) => return Err(e.into()),
}
```

`Error::Api { status, code, title, detail, retryable, available_at, request_id, errors, idempotency_key }`,
`Error::Transport { source, idempotency_key }`, `Error::Invalid(String)`.

## Pitfalls

- Without `preserve_order`, `serde_json::Map` is a `BTreeMap` and sorts keys: NEP-366 args
  bytes change and the API rejects the signature.
- Use `u128` (or strings) for amounts; 24-decimal NEAR amounts overflow `u64`.
- The crate allows `clippy::result_large_err` on purpose (errors carry the full document).
