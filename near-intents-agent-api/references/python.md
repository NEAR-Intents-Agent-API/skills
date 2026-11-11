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
