# Status, idempotency and recovery

## Statuses

| Status | Meaning | Your move |
|---|---|---|
| `PENDING_SIGNATURE` | owner intent generated, not yet signed | get the signature or let it expire |
| `QUEUED` | held by the policy timelock until `details.execute_after` | wait; owner may `execution_cancel` |
| `PENDING_APPROVAL` | waiting for the owner's `approval_vote` (`details.approval_id`) | ask the owner; poll slowly |
| `PENDING_DEPOSIT` | deposit address issued, waiting for the payer's funds. **Only deposits report it** | show `deposit_address`; poll until `expires_at` (a payment after it reopens the deposit) |
| `PROCESSING` | submitted, settlement not yet observed. Includes a withdrawal's or swap's bridge leg waiting on 1Click (`details.provider_status` keeps the provider's own state) and a provider `success` whose settlement it still reports pending | keep polling |
| `SUCCESS` | settled with evidence | done; `details.amount_out` (delivered) is present only now, or at `REFUNDED` after a partial fill. Before that read `expected_amount_out` and treat it as an estimate |
| `REFUNDED` | route refunded the input (`refund_tx_hash`) | done; tell the user |
| `FAILED` | never executed, or failed with `failure_code` (an unpaid deposit: `deposit_expired`, a day after `expires_at`; a native NEAR transaction NEAR finalized as failed: `native_transaction_failed`) | done; new request needs a new key |
| `UNCERTAIN` | outcome not yet proven, **may have executed** | keep polling the same id; never resubmit under a new key |
| `NEEDS_REVIEW` | automatic settlement stopped | stop; inspect `details.reason`; then `refresh=true` |

Terminal: `SUCCESS`, `REFUNDED`, `FAILED`. `UNCERTAIN` usually resolves on its own as the API
reconciles with the provider.

A withdrawal is never `PENDING_DEPOSIT`: while its bridge leg waits on 1Click it reads `PROCESSING`.
A provider `success` with settlement still pending also reads `PROCESSING`, up to the settlement
deadline; past it the operation reads `UNCERTAIN` with `failure_code: settlement_overdue`. Keep
polling the same `correlation_id`. For a deposit, `details.deposit_seen_at` is when funds were first
seen; the settlement deadline runs from there, not from when the address was issued.

A native NEAR transfer or call reads `PROCESSING` until two RPC operators agree NEAR finalized the
transaction and it is exactly the request, whatever `details.provider_status` says. Then `SUCCESS`; a
finalized failure is `FAILED` (`native_transaction_failed`); a transaction that is not the request is
`NEEDS_REVIEW` (`native_transaction_mismatch`); one NEAR still does not know 30 minutes after dispatch
is `NEEDS_REVIEW` (`native_transaction_unconfirmed`).

## Polling

```text
GET /v1/status?correlation_id=<id>&wait_ms=30000
```

- Long-poll: returns early when the status becomes terminal, needs a signature, or needs review;
  otherwise after `wait_ms` (max 30000). Reading also advances the operation (the API checks
  chain/provider on read), so polling is how slow operations make progress — don't stop early.
- Max 10 concurrent long-polls per API key; further reads answer immediately (no error), so
  don't build a tight loop on top of them. Add a small sleep when a read returns instantly with
  a non-terminal status.
- Typical loop: repeat while status ∈ {`PROCESSING`, `QUEUED`} up to your deadline (a few
  minutes for swaps/transfers, longer for cross-chain withdrawals and deposits). For
  `PENDING_APPROVAL` / `PENDING_DEPOSIT` hand control back to the user and poll on a slower
  schedule (or when they return).
- `GET /v1/agents/{id}/history` lists everything for an activity feed (newest first). Deposit
  addresses nothing was paid into (open, expired, closed by deletion) are left out unless
  `include_unpaid=true`; read one by its `op_…` id with `GET /v1/status`.
- `details.request` on an execution's status and history echoes the request as admitted, in the
  request body's field names (`origin_asset`, `destination_asset`, `asset`, `amount`,
  `min_amount_out`, `chain`, `recipient`, `memo`, `confidential`, each only when it applies), so a
  client can show a transfer's recipient or what is reserved per asset. Executions admitted before
  it was recorded have none.
- A single long poll does not promise settlement; `SUCCESS` means reconciled evidence, not just
  a broadcast hash.

```python
def settle(api, cid, deadline_s=300):
    end = time.monotonic() + deadline_s
    while True:
        s = api.get_status(cid, wait_ms=30_000)
        if s["status"] not in ("PROCESSING", "QUEUED") or time.monotonic() > end:
            return s
```

## Idempotency keys

- Required on every non-dry execution and deposit; optional on `generate-intent`.
- 8–128 characters from `[A-Za-z0-9._:-]`. A UUIDv4 or `"<action>:<your_request_id>"` works.
  Don't build keys from unbounded user data (length) or from values that repeat across distinct
  requests (e.g. `transfer-${agentId}-${recipient}` would block the second payment to the same
  person).
- **Create → persist with the body → send.** If the process dies after sending, the stored key
  lets you find or safely repeat the request.
- Same key + same body → the original operation (no double spend), even across restarts.
- Same key + different body → `409 idempotency_conflict`.
- After a definitive refusal or `FAILED`, a retry is a new request → new key.
- SDK/assets return the key they used (`result.idempotencyKey`, `AgentApiRequestError.idempotencyKey`)
  so you can retry with it after a transport error.

## UNCERTAIN

The API lost the provider's answer around dispatch, so it cannot prove whether the money moved.

1. Keep polling the same `correlation_id`. The API reconciles via the provider's records.
2. Never resubmit under a new idempotency key — that can pay twice.
3. If it stays `UNCERTAIN`, look at the operation:
   - has `provider_request_id` or `dispatch_committed_at` → **the provider got it**. Only
     polling resolves it. Never recover.
   - neither → **the provider never got it**. `POST …/recover` may dispatch an execution, once. A
     native transfer or call and a deletion have no resume: they wait for the sweep, which fails them
     with `dispatch_never_committed` after an hour.

### `POST /v1/agents/{agent_id}/recover`

```http
POST /v1/agents/{agent_id}/recover
X-API-Key: <the key that admitted the operation>
X-Grant-Token: <the token of the grant that admitted it>
Idempotency-Key: <the ORIGINAL key>
{ "correlation_id": "op_…", "request": { "type": "withdraw", …the original body, byte-identical after canonicalization… } }
```

Authorization runs again in full: if the grant was revoked, the account frozen or the policy
tightened since, recovery is refused. `operation_not_recoverable` means it was not eligible (not
UNCERTAIN, already reached the provider, cross-chain deposit, or another API key);
`operation_recovery_unavailable` means a recovery is already running. This is why you persist
the original body and key.

## NEEDS_REVIEW

Automatic settlement stopped because something did not add up (e.g. evidence conflicts).

1. Stop polling and stop retrying.
2. Read `details.reason`; cross-check `GET …/provider/requests`, `…/provider/audit`,
   balances.
3. After resolving, `GET /v1/status?correlation_id=…&refresh=true` to re-observe.
4. Surface it to an operator; don't tell the user the payment failed or succeeded.

## Operation proofs

`GET /v1/agents/{agent_id}/operations/{correlation_id}/proof` (an `op_…` id) returns every
retained audit event of one execution, each `PROVEN` with a `c2sp.org/tlog-proof@v1` against the
notary-signed checkpoint of the API's transparency log, `PENDING` until the next checkpoint covers
it, or `UNLOGGED`. Each event's `evidence` is the result it recorded, with any settlement
transaction hashes, committed by the opening's `evidenceHash`. Use it to audit an execution offline without trusting the API; it does not
change the execution's status. Check it with the SDK's `verifyOperationProof(proof, origin)`, pinning
the origin yourself (`<API host>/log`, `api.agentsonintents.com/log` for the hosted API); see
[typescript.md](typescript.md#operation-proofs) and the examples' `11:verify-operation`.

## Revocation and freezing vs in-flight work

`dispatch_committed_at` is the point of no return. Before it, revoking the grant, revoking the
API key or freezing refuses the operation. After it, the operation completes; `grant_revoke`
details and API-key revocation list those committed ids.
