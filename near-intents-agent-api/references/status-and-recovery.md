# Status, idempotency and recovery

## Statuses

| Status | Meaning | Your move |
|---|---|---|
| `PENDING_SIGNATURE` | owner intent generated, not yet signed | get the signature or let it expire |
| `QUEUED` | held by the policy timelock until `details.execute_after` | wait; owner may `execution_cancel` |
| `PENDING_APPROVAL` | waiting for the owner's `approval_vote` (`details.approval_id`). If its deadline passes or its authority lapses first it settles `FAILED` (`expired` / `authorization_stale`, `details.never_executed`, charge released) | ask the owner; poll slowly |
| `PENDING_DEPOSIT` | deposit address issued, waiting for the payer's funds. **Only deposits report it** | show `deposit_address`; poll until `expires_at` (a payment after it reopens the deposit) |
| `PROCESSING` | submitted, settlement not yet observed. Includes a withdrawal's or swap's bridge leg waiting on 1Click (`details.provider_status` keeps the provider's own state) and a provider `success` whose settlement it still reports pending | keep polling |
| `SUCCESS` | settled with evidence | done; `details.amount_out` (delivered) is present only now, or at `REFUNDED` after a partial fill. Before that read `expected_amount_out` and treat it as an estimate. An exact-output swap or withdrawal also has `details.amount_in`, the input it spent (the rest of `max_amount_in` is back in the balance), and `details.quote`. A **limit order** is different: `SUCCESS` means funded and resting, not filled (`details.order_id`; see [limit-orders.md](limit-orders.md)) |
| `REFUNDED` | route refunded the input (`refund_tx_hash`) | done; tell the user |
| `FAILED` | never executed, or failed with `failure_code` (an unpaid deposit: `deposit_expired`, a day after `expires_at`; a native NEAR transaction NEAR finalized as failed: `native_transaction_failed`; a timelocked request queued in a shape the API no longer accepts, such as a withdrawal with a `memo`: `request_no_longer_accepted`, never sent to the provider, so send it again in the current shape under a new key) | done; new request needs a new key |
| `UNCERTAIN` | outcome not yet proven, **may have executed** | keep polling the same id; never resubmit under a new key |
| `NEEDS_REVIEW` | automatic settlement stopped | stop; inspect `details.reason`; then `refresh=true` |

Terminal: `SUCCESS`, `REFUNDED`, `FAILED`. `UNCERTAIN` usually resolves on its own as the API
reconciles with the provider.

A withdrawal is never `PENDING_DEPOSIT`: while its bridge leg waits on 1Click it reads `PROCESSING`.
A provider `success` with settlement still pending also reads `PROCESSING`, up to the settlement
deadline; past it the operation reads `UNCERTAIN` with `failure_code: settlement_overdue`. Keep
polling the same `correlation_id`. For a deposit, `details.deposit_seen_at` is when funds were first
seen; the settlement deadline runs from there, not from when the address was issued.

A limit order's operation ends at `SUCCESS` once the order is funded and resting; its fills, payouts and
cancellation are not that operation's status. Read the order itself with
`GET /v1/agents/{agent_id}/limit-orders/{order_id}` (`fill_status`, `partial_fills`, `payouts`,
`is_payout_status_final`); a later cancel or fill never changes the placement's `SUCCESS`.

A native NEAR transfer or call reads `PROCESSING` until two RPC operators agree NEAR finalized the
transaction and it is exactly the request, whatever `details.provider_status` says. Then `SUCCESS`; a
finalized failure is `FAILED` (`native_transaction_failed`); a transaction that is not the request is
`NEEDS_REVIEW` (`native_transaction_mismatch`); one NEAR still does not know 30 minutes after dispatch
is `NEEDS_REVIEW` (`native_transaction_unconfirmed`).

An `approval_vote` that reads `SUCCESS` means the vote was recorded. Its `details` are
`{ approval_id, verdict, repeat_required, confirmation? }`. A `reject` cast within a block or two of a
policy change can be stored by the provider while the request stays pending. The API resends the
owner's signed reject for a few seconds. Then read `details.repeat_required`:

- `true`: the request is still pending. Generate the `approval_vote` intent again and have the owner
  sign another `reject` for the same approval.
- `false`: nothing more to do.
- `null`: not known yet, the vote has not been applied.

`details.confirmation: "cancellation_observed"` means a fresh reject lost its reply and the API
observed the cancellation from the earlier receipt: the reject took effect. Approves are
unaffected. SDK 0.9.4 types both fields.

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
  `min_amount_out`, `amount_out`, `max_amount_in`, `chain`, `recipient`, `reference`, `note`, `confidential`, each only when it applies; a limit
  order adds `side`, `price`, `quantity`, `deadline`, with `origin_asset`/`amount` what it spends), so a
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

- Required on every non-dry execution (including `placeLimitOrder`) and deposit; optional on `generate-intent`.
  Cancelling a limit order takes none: it needs the grant only.
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
- Lost the response and never stored the `correlation_id`? Read it back with the key
  ([below](#lost-the-response-look-the-key-up)) before you send anything again.

## Lost the response: look the key up

`GET /v1/agents/{agent_id}/operations?idempotency_key=<key>` (API key only, no grant) recovers the
`correlation_id` of a request whose response never reached you, **without replaying it**. It is a read:
nothing is dispatched, no provider call is made, nothing is spent.

```http
GET /v1/agents/{agent_id}/operations?idempotency_key=withdraw-9f2c1a
X-API-Key: <partner API key>
→ 200 { "data": [ StatusResponse, … ], "retired": [ "op_…" ] }
```

- `idempotency_key` is required, 8–128 chars `[A-Za-z0-9._:-]` (anything else is a 400).
- `data`: the retained operations the key admitted, **newest first**, shaped like `GET …/history`
  entries and not reconciled with the provider (to settle one, `GET /v1/status` with its id).
  `details.request` carries the admitted terms (asset, amount, recipient) when they were recorded;
  older operations lack them.
- `retired`: ids whose detail was removed after retention (90 days after the last update by default).
  Only the permanent admission record remains: the key was admitted and **stays spent**, so it never
  runs again, but the API can no longer tell you the outcome.
- Keys are scoped to the grant: the same key under two grants names two operations, and both come
  back. Revoked and expired grants are searched too, and so is the grant-free scope (deposits).
- It covers requests that consumed a key at admission (executions, deposit-address allocations,
  native NEAR transfers and calls). A dry preview creates no operation, so there is nothing to find.
  An owner intent is read by its own `intent_…` id. Native signing routes take no key: find them in history.
- **Empty lists prove nothing.** The request may still be in admission, may have been refused before
  anything was recorded, or its original grant may no longer be known. Only a replay settles it: the
  **same key and the same body, under the original grant, while that grant can still authorize the
  request**. A replacement grant is another scope and can admit a second operation, and if the
  original grant can no longer authorize it the replay may leave the outcome unresolved.
- A key you did not keep cannot be recovered (the API does not store keys): persist it before sending.

```ts
const { data, retired } = await api.lookupOperations(agentId, idempotencyKey);   // SDK 0.9.3+
const found = data[0]?.correlation_id;       // then getStatus(found, { waitMs: 30_000 })
```

## UNCERTAIN

The API lost the provider's answer around dispatch, so it cannot prove whether the money moved.

1. Keep polling the same `correlation_id`. The API reconciles via the provider's records. If you
   never stored it, [look the key up](#lost-the-response-look-the-key-up).
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

A limit order is recovered the same way, with `"type": "limit_order"` and its original body. A swap or withdrawal body is exact input or exact output, as when it was sent.
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
transaction hashes, committed by the opening's `evidenceHash`. Use it to audit an execution without trusting the API; it does not
change the execution's status. Verification needs a trusted checkpoint: with SDK 0.8.0 or later, fetch a
fresh head using `fetchWitnessCheckpoint(apiUrl, origin, witnessPublicKey)`, request the proof
with `?from_size=` set to that head's size, and call `verifyOperationProof(proof, origin, head)`.
Pin the origin (`<API host>/log`, `api.agentsonintents.com/log` for the hosted API) and the
witness key yourself, and never take either from an API response. A server behind the head
answers `409 transparency_checkpoint_behind`: keep your trusted checkpoint, never reset it, and retry once the server has recovered. See
[typescript.md](typescript.md#operation-proofs) and the examples' `11:verify-operation`.

## Receipts

A completed transfer, withdrawal, swap, limit order or deposit has a receipt: a leaf of the same log,
with no agent, account, `usd_value` or `note` in it, so it can be handed to a payee. Read it with
`GET /v1/agents/{agent_id}/operations/{correlation_id}/receipt` (`receipt` is null while `PENDING`,
and for `UNLOGGED`) or list them with `GET …/receipts?type=` ([executions.md](executions.md#reference-note-and-receipts)).
With the SDK, `api.getReceipt` / `api.listReceipts` read them. Anyone verifies `item.receipt` with
`verifyReceipt(receipt, { origin, trusted, readTile })`; a payee owed a transfer or withdrawal uses
`verifyPaymentReceipt`, with `expect` holding the exact `recipient`, `asset`, `amount` and the
`reference` it issued (plus `chain` for a withdrawal; `memo` only for a receipt issued before withdrawals dropped it). Give `expect` exactly one of `amount` and `amount_out`: `amount` matches an exact-input payment, `amount_out` an exact-output withdrawal, which paid the recipient exactly that (the result's `amountOut`, with `maxAmountIn` the most it could spend, and no `amount`; `VerifiedPayment.amount` is optional, SDK 0.9.6). A payee owed an exact figure should ask for an exact-output withdrawal. `expect.confidential` omitted
accepts a **confidential** payment only, the account's default; pass `confidential: false` to accept
one paid from the legacy public balance (a withdrawal sent with `confidential: false`). A receipt does
not prove who paid, and an exact-input withdrawal's `amount` is the debit, not what arrived: issue one `reference`
per order and accept each `correlation_id` once. `ReceiptNotCoveredError` or `LogUnavailableError`:
retry with a newer head or later; `NoteError` or `ProofError`: reject. See
[typescript.md](typescript.md#receipts). Receipts last as long as the audit log (90 days).

## Revocation and freezing vs in-flight work

`dispatch_committed_at` is the point of no return. Before it, revoking the grant, revoking the
API key or freezing refuses the operation. After it, the operation completes; `grant_revoke`
details and API-key revocation list those committed ids.

The same holds for signatures. A native signing route returns its signature to the caller, so
revoking, freezing or deleting afterwards cannot recall it: a `sign-intent` payload signed before
the freeze stays publishable by anyone who holds it until its own `deadline` (up to 300 s, or 96 h
for a payload that pays only 1Click deposits), see [sign-intent.md](sign-intent.md).
