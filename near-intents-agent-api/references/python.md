# Python

There is no published Python package. Copy
[../assets/python/near_intents_agent_api.py](../assets/python/near_intents_agent_api.py) into
your project (one file, mirrors the TypeScript SDK).

```bash
pip install httpx cryptography        # client + NEAR signing
pip install eth-account               # only for EVM (EIP-712) owners
python assets/python/test_vectors.py  # signer self-check against assets/test-vectors.json
```

Python 3.10+.

## Client

```python
from near_intents_agent_api import AgentApi, AgentApiError, AgentApiRequestError, create_idempotency_key

api = AgentApi.from_env()                 # NEAR_INTENTS_AGENT_API_KEY, NEAR_INTENTS_AGENT_API_URL (default hosted)
# or AgentApi(api_key="<partner API key>", base_url="https://api.agentsonintents.com")

api.get_network(); api.whoami(); api.get_quotas()
tokens = {t["asset_id"]: t for t in api.get_tokens()}
```

Methods: `generate_intent`, `submit_intent`, `get_status(cid, wait_ms, refresh)`,
`wait_for_status(cid, deadline_s)`, `list_agents`, `get_agent`, `get_balances(agent_id, source="confidential", asset)`,
`get_policy`, `list_grants`, `get_history`, `list_scheduled`, `list_approvals`,
`get_containment`, `quote_swap`, `quote_withdraw`, `swap`, `withdraw`, `transfer`, `shield`,
`unshield` (always refused), `deposit`, `recover`, the limit-order methods (`quote_limit_order`, `place_limit_order`,
`list_limit_orders`, `get_limit_order`, `cancel_limit_order`, `cancel_all_limit_orders`), plus the generic `request(method, path, query=, body=, idempotency_key=, grant=)`
for anything else. Responses are plain dicts with the API's snake_case fields.

Helpers: `create_idempotency_key()`, `create_grant_credential()` → `{token, commitment}`,
`grant_commitment(token)`, `to_atomic("1.5", 24)`, `to_decimal(raw, decimals)`,
`iterate_pages(lambda c: api.list_agents(cursor=c))`.

## Owner flow with a NEAR key (script / company-held key)

```python
from near_intents_agent_api import NearKey, near_owner, sign_intent, run_owner_intent

key = NearKey(os.environ["AGENT_OWNER_PRIVATE_KEY"])            # ed25519:<base58 seed+pub>
owner = near_owner(os.environ["AGENT_OWNER_ACCOUNT_ID"], key)

status = run_owner_intent(
    api,
    {"type": "agent_create", "name": "Treasury bot", "owner": owner, "policy": policy},
    lambda generated: sign_intent(generated, near_key=key, rpc_url="https://free.rpc.fastnear.com"),
)
agent_id = status["details"]["agent_id"]
```

EVM owner:

```python
from near_intents_agent_api import evm_owner, sign_intent
owner = evm_owner(os.environ["AGENT_OWNER_EVM_PRIVATE_KEY"])
sign = lambda g: sign_intent(g, evm_private_key=os.environ["AGENT_OWNER_EVM_PRIVATE_KEY"])
```

In a web product, skip the signers: return `generated["intent"]` and `generated["preview"]` to
the browser and pass the wallet's output to `api.submit_intent(type, correlation_id, signed_data)`.

## Grant and execute

```python
from near_intents_agent_api import create_grant_credential
from datetime import datetime, timedelta, timezone

cred = create_grant_credential()                                 # store cred["token"] encrypted
expires = (datetime.now(timezone.utc) + timedelta(days=7)).isoformat().replace("+00:00", "Z")
done = run_owner_intent(api, {"type": "grant_issue", "agent_id": agent_id, "label": "bot",
                              "credential": cred["commitment"], "expires_at": expires}, sign)
grant_id = done["details"]["grant"]["grant_id"]

agent = api.for_grant(cred["token"])
quote = agent.quote_swap(agent_id, {"origin_asset": "nep141:wrap.near", "destination_asset": usdc,
                                    "amount": to_atomic("0.1", 24)})
key = create_idempotency_key()                                   # persist with the body first
op = agent.swap(agent_id, {"origin_asset": "nep141:wrap.near", "destination_asset": usdc,
                           "amount": to_atomic("0.1", 24)}, idempotency_key=key)
final = api.wait_for_status(op["correlation_id"])
```

Limit orders (see [limit-orders.md](limit-orders.md)):

```python
order = {"base_asset": "nep141:wrap.near", "quote_asset": usdt, "side": "sell",
         "quantity": to_atomic("1", 24), "price": "5.2"}               # price: quote units per whole base
agent.quote_limit_order(agent_id, order)                              # dry: no grant, no key
key = create_idempotency_key()                                        # persist with the body first
op = agent.place_limit_order(agent_id, order, idempotency_key=key)
done = api.wait_for_status(op["correlation_id"])                      # SUCCESS = funded and resting, not filled
order_id = done["details"]["order_id"]
live = api.get_limit_order(agent_id, order_id)                        # fill_status, partial_fills, payouts
agent.cancel_limit_order(agent_id, order_id)                          # grant, no key
while not agent.cancel_all_limit_orders(agent_id)["complete"]: pass   # 50 a call
```

## Workload identity

For a federated grant (`oidc` or `public_key`; which one and the platform recipes:
[workload-identity.md](workload-identity.md)) the asset has no dedicated methods. A session token
`ngs_` is accepted by `for_grant` like an `ngt_`, and the exchange is one `request`. A session expires
in 15 minutes by default, so keep it and renew it shortly before (the helper below also keeps serving
the old token until 5 s before its end, backs off 5 s after a failed exchange and forgets a refused session):

```python
import threading
import time


class GrantSession:
    """Exchanges lazily and keeps the ngs_ token. From min(60 s, life/2) before its end the current token
    keeps serving while one exchange renews it; a failure there costs nothing until the last 5 s. A failed
    exchange is not repeated for 5 s. One per grant: a grant holds at most 100 live sessions."""

    def __init__(self, api, agent_id, subject_token, grant_id=None):
        self._api, self._agent_id, self._subject_token, self._grant_id = api, agent_id, subject_token, grant_id
        self._client = self._error = None
        self._refresh_at = self._expires_at = self._retry_at = 0.0
        self._lock = threading.Lock()

    def client(self):
        """The AgentApi to call delegated endpoints with."""
        with self._lock:
            now = time.monotonic()
            if self._client and now < self._refresh_at:
                return self._client
            usable = self._client is not None and now < self._expires_at - 5
            if now < self._retry_at:  # backing off after a failed exchange
                if usable:
                    return self._client
                raise self._error
            try:
                self._exchange()
            except Exception as error:
                self._error, self._retry_at = error, time.monotonic() + 5
                if not usable:
                    raise
            return self._client

    def invalidate(self, client):
        """Forget a session the API refused (agent_grant_required): the next call() exchanges again."""
        with self._lock:
            if self._client is client:
                self._client = None

    def _exchange(self):
        body = {
            "grant_type": "urn:ietf:params:oauth:grant-type:token-exchange",
            "subject_token": self._subject_token(),  # a fresh platform JWT, or a signed key assertion
            "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
        }
        if self._grant_id:
            body["grant_id"] = self._grant_id
        started = time.monotonic()
        session = self._api.request("POST", f"/v1/agents/{self._agent_id}/grants/token", body=body)
        life = session["expires_in"]
        self._client = self._api.for_grant(session["access_token"])  # ngs_ is accepted like ngt_
        self._refresh_at = started + life - min(60, life / 2)
        self._expires_at = started + life
        self._error = None

# OIDC grant: the platform's token for audience f"{base_url}/v1/agents/{agent_id}", read fresh each time
session = GrantSession(api, agent_id, subject_token=lambda: read_platform_token(f"{api.base_url}/v1/agents/{agent_id}"))
agent = session.client()
try:
    op = agent.transfer(agent_id, {"asset": "nep141:wrap.near", "amount": "1", "recipient": "bob.near"},
                        idempotency_key=create_idempotency_key())
except AgentApiError as e:
    if e.code == "agent_grant_required":  # the grant behind the session is gone: drop it, do not auto-retry
        session.invalidate(agent)
    raise
```

Key-bound grant, **a proof for every request** (no bearer token). `private_key` is a
`cryptography` Ed25519 or P-256 key; `public_jwk` is the JWK the owner signed; `grant_id` comes from the
`grant_issue` result or `list_grants`. For a key assertion (exchange), sign
`{iss, sub: thumbprint_uri(public_jwk), aud: f"{base_url}/v1/agents/{agent_id}/grants/token", iat, exp, jti}`
with `sign_jwt` and pass it as `subject_token`. Claims and rules: [http-api.md](http-api.md#signing-every-request-x-grant-proof).

```python
import base64, hashlib, json, time, uuid
from urllib.parse import urlsplit

from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives.asymmetric.utils import decode_dss_signature

THUMBPRINT_PREFIX = "urn:ietf:params:oauth:jwk-thumbprint:sha-256:"


def b64u(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()


def thumbprint_uri(jwk: dict) -> str:
    """RFC 7638 over the required public members only (crv, kty, x[, y]), sorted, no spaces."""
    members = {k: jwk[k] for k in ("crv", "kty", "x", "y") if k in jwk}
    return THUMBPRINT_PREFIX + b64u(hashlib.sha256(json.dumps(members, separators=(",", ":")).encode()).digest())


def sign_jwt(private_key, public_jwk: dict, claims: dict) -> str:
    """`private_key` is a cryptography Ed25519PrivateKey or P-256 EllipticCurvePrivateKey."""
    alg = "EdDSA" if public_jwk["kty"] == "OKP" else "ES256"
    compact = lambda value: b64u(json.dumps(value, separators=(",", ":")).encode())
    signing_input = f"{compact({'alg': alg})}.{compact(claims)}".encode()
    if alg == "EdDSA":
        signature = private_key.sign(signing_input)
    else:  # P-256: JWS wants raw r || s, not the DER that cryptography (and a KMS) returns
        r, s = decode_dss_signature(private_key.sign(signing_input, ec.ECDSA(hashes.SHA256())))
        signature = r.to_bytes(32, "big") + s.to_bytes(32, "big")
    return f"{signing_input.decode()}.{b64u(signature)}"


def grant_proof(private_key, public_jwk, grant_id, method, url, body: bytes = b"", idempotency_key=None) -> str:
    """One X-Grant-Proof: single use, so call it again (new jti) for every attempt, retries included."""
    parts = urlsplit(url)
    iss = thumbprint_uri(public_jwk)
    now = int(time.time())
    claims = {
        "iss": iss, "sub": iss, "grant_id": grant_id,
        "aud": f"{parts.scheme}://{parts.netloc}{parts.path}",     # no query, no fragment
        "htm": method.upper(),
        "body_sha256": hashlib.sha256(body).hexdigest(),           # of the exact bytes you send
        "iat": now, "exp": now + 60, "jti": str(uuid.uuid4()),
    }
    if idempotency_key is not None:                                 # omit the claim when no header is sent
        claims["idempotency_key"] = idempotency_key
    return sign_jwt(private_key, public_jwk, claims)
```

```python
import httpx

body = json.dumps({"asset": "nep141:wrap.near", "amount": "1", "recipient": "bob.near"}).encode()  # exact bytes, once
key = create_idempotency_key()                       # persist with the body first; the retry reuses it
url = f"{api.base_url}/v1/agents/{agent_id}/transfer"
response = httpx.post(url, content=body, headers={
    "x-api-key": api.api_key,
    "x-grant-proof": grant_proof(private_key, public_jwk, grant_id, "POST", url, body, key),
    "idempotency-key": key,
    "content-type": "application/json",
})
# On a timeout or 5xx: same body, same key, same grant_id, and a NEW grant_proof(...) call.
```

A key in a KMS or HSM: replace the `private_key.sign(...)` calls in `sign_jwt` with the KMS call (for
P-256 most KMSs sign the SHA-256 digest and return DER: `decode_dss_signature` converts it, as above).
The key itself never leaves the KMS. Each signature is a billed network call: prefer a session
(`GrantSession` with a signed assertion) unless no bearer token may exist. Never log `ngs_`/`ngt_`
tokens, platform JWTs, proofs or private keys.

## Errors

```python
try:
    agent.withdraw(agent_id, body, idempotency_key=key)
except AgentApiError as e:
    if e.code == "policy_destination_denied": ...
    elif e.code in ("policy_change_throttled", "rate_limited"): retry_at = e.available_at
    else: raise
except AgentApiRequestError as e:
    # network/timeout: retry with e.idempotency_key and the same body
    ...
```

`AgentApiError` has `status`, `code`, `title`, `detail`, `retryable`, `available_at`,
`request_id`, `errors`, `idempotency_key`.

The native routes (`sign-intent`, `sign-in`, `sign-transfer`) have no wrapper in the asset: call
`api.for_grant(token).request("POST", f"/v1/agents/{agent_id}/native/evm/sign-intent", body={"payload": payload}, grant=True)`;
when the payload pays a 1Click deposit address, add `"one_click_quotes": [quote]` (1Click's live quote answer, unchanged)
([sign-intent.md](sign-intent.md)).

## Async / frameworks

The asset uses a sync `httpx.Client`. For asyncio (FastAPI, aiohttp), switch to
`httpx.AsyncClient` and `await` in `request()` — the rest is unchanged. Run long-polls in
background tasks (Celery, RQ, arq) rather than request handlers.

## Pitfalls

- Use `Decimal`/strings for amounts, never floats (`to_atomic` refuses excess precision).
- Don't `json.dumps(..., sort_keys=True)` anything you sign; `js_json` reproduces
  `JSON.stringify` for NEP-366 args.
- EIP-712 needs `eth-account>=0.13` (`encode_typed_data(full_message=payload)`).
