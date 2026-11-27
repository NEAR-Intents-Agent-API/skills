# Executions: moving money

All are `POST /v1/agents/{agent_id}/<action>` and return `202` with a `StatusResponse`
(`correlation_id: "op_…"`), except dry quotes (`200`). Limit orders live at
`POST …/limit-orders`; see [limit-orders.md](limit-orders.md).

| Action | Headers | Body | Policy needs |
|---|---|---|---|
| `swap` | key, grant, idem | `origin_asset`, `destination_asset`, then `amount`, `min_amount_out?` (exact input) or `amount_out`, `max_amount_in` (exact output), `confidential?`, `note?`, `dry?` | `swap`, `confidential: true` |
| `limit-orders` | key, grant, idem | `base_asset`, `quote_asset`, `side`, `quantity`, `price`, `deadline?`, `confidential?`, `note?`, `dry?` | `limit_order`, `confidential: true` |
| `withdraw` | key, grant, idem | `asset`, `chain`, `recipient`, then `amount`, `min_amount_out?` (exact input) or `amount_out`, `max_amount_in` (exact output), `reference?`, `confidential?`, `note?`, `async?`, `dry?` | `withdraw`, destination allowed |
| `transfer` | key, grant, idem | `asset`, `amount`, `recipient`, `reference?`, `confidential?`, `note?` | `transfer`, destination allowed, `confidential: true` |
| `shield` | key, grant, idem | `asset`, `amount` | `confidential: true` |
| `unshield` | key | `asset`, `amount` | refused: every request is `public_balance_exit_only` |
| `deposit` | key, idem | `origin_asset`, `destination_asset?`, `amount?`, `confidential?`, `note?` | `confidential: true` (no grant) |
| `recover` | key, grant, original idem | `correlation_id`, `request: {type, …original body}` | re-authorized in full |

**Confidential by default.** The agent's account is its confidential balance: omit `confidential`
(or send `true`) and every action above uses it. `confidential: false` is refused with
`public_balance_exit_only` (400, layer `request`, not retryable; live and `dry`, before any provider
call) on `swap`, `limit-orders`, `transfer` and `deposit`. `withdraw` accepts it and pays from the
legacy public balance: with `shield` the only way to empty it. `unshield` is refused for every
request, so never offer it. Work already recorded on the public balance (held, in flight, resting
limit orders) still settles, and a replay of its key returns its record.

Amounts are atomic integer strings: digits only, no sign and no leading zeros, and greater than zero for
every action that moves funds (`validation_failed` otherwise). Asset ids come from `GET /v1/tokens`. A recipient that is not a valid
address for its chain is refused with `400 invalid_address` before the policy is read.

## Exact input or exact output

`swap` and `withdraw` take one of two shapes. Never mix them (`validation_failed`): exactly one of
`amount` and `amount_out` is required, `max_amount_in` goes only with `amount_out`, and
`min_amount_out` only with `amount`.

- **Exact input**: `amount` is what the account spends; `min_amount_out` (optional) is the least the
  recipient must get.
- **Exact output**: `amount_out` is what the recipient gets, exactly; `max_amount_in` is the most the
  account may spend to get it. The unused input returns to the balance it was spent from.

```json
POST …/swap      { "origin_asset": "nep141:wrap.near", "destination_asset": "<usdc>", "amount": "1000000000000000000000000", "min_amount_out": "2900000" }
POST …/swap      { "origin_asset": "nep141:wrap.near", "destination_asset": "<usdc>", "amount_out": "3000000", "max_amount_in": "1100000000000000000000000" }
POST …/withdraw  { "asset": "<usdc>", "chain": "base", "recipient": "0xabc…", "amount": "5000000", "min_amount_out": "4900000" }
POST …/withdraw  { "asset": "<usdc>", "chain": "base", "recipient": "0xabc…", "amount_out": "5000000", "max_amount_in": "5200000" }
```

- The balance must hold all of `max_amount_in`, and per-asset limits and the USD budget charge all
  of it, even the part returned. Size the ceiling to what the account may risk.
- A withdrawal from the public balance (`confidential: false`) to chain `near` takes `amount` alone and
  delivers it exactly: `amount_out` or `min_amount_out` there is `validation_failed`, live and `dry`.
- A withdrawal takes **no `memo`** (`validation_failed`): no route carries one to the recipient, so do
  not offer to send one, and do not withdraw to an address that needs one (an exchange deposit tag).
- The provider takes a fresh quote for each live request and checks your figure against it before
  anything is signed or sent. A `min_amount_out` above the quote's guaranteed output
  (`quote.min_amount_out`, not the estimate `quote.amount_out`) is refused `below_min_amount_out`; a
  `max_amount_in` below the quote's `amount_in` is refused `above_max_amount_in`. Both are 409, send
  nothing and are not retryable: change the figure and use a new key. The refusal states the fresh
  quote's figure, `meta.quoted_min_amount_out` or `meta.quoted_amount_in`, when the provider gives
  one: a gap of a few units is drift, a large one a real price move. A dry run's quote is an
  earlier one, so leave 0.5 to 1% room between it and the floor or ceiling you send, and more for
  a request a timelock or owner approval holds: it is quoted when it releases and fails for good
  on a miss. Without `min_amount_out` the floor is the fresh quote's own `min_amount_out`, set by
  the provider's slippage tolerance.
- At settlement an exact-output request completes only on a reported delivery of at least
  `amount_out`; less stops as `NEEDS_REVIEW` with `amount_out_below_minimum`.

## Quote first

```json
POST …/swap      { "origin_asset": "nep141:wrap.near", "destination_asset": "<usdc>", "amount": "1000000000000000000000000", "dry": true }
POST …/withdraw  { "asset": "<usdc>", "amount": "5000000", "chain": "base", "recipient": "0xabc…", "dry": true }
→ 200 { "dry": true, "type": "swap" | "withdraw" | "limit_order", "quote": { amount_in?, min_amount_in?, amount_out?, min_amount_out?,
                                                                    withdraw_fee?, refund_fee?, amount_in_usd?, amount_out_usd? },
        "spend": { "amount_usd": string | null, "budget": { "daily": {…}, "weekly": {…}, "monthly": {…} } },
        "fee": { "amount_usd": string, "percent": string } | null,
        "warnings": [ { "code": "high_fee" | "asset_delivered_as_is" | "provider_would_refuse"
                              | "policy_action_denied" | "policy_destination_denied" | "policy_asset_denied"
                              | "policy_transaction_limit_exceeded" | "policy_schedule_denied" | "spend_price_unavailable" | "spend_budget_exceeded"
                              | "policy_limit_exceeded" | "policy_action_limit_exceeded" | "policy_usage_unavailable"
                              | "approval_backlog_full"
                              | "insufficient_balance"
                              | "below_min_amount_out" | "above_max_amount_in"
                              | "amount_too_low" | "policy_price_deviation_exceeded" | "market_price_unavailable",   // the last three: limit orders
                        "detail": string, "available_at"?: string, "available_amount"?: string } ],
        "owner_approval": boolean }   // true: the live request waits in PENDING_APPROVAL for the owner
```

Dry runs charge no budget, reserve none and move nothing. A swap quote needs no grant and no
idempotency key; nor does a limit-order preview, which makes no provider call (its `quote` is the
API's own: what the order spends and the least a full fill delivers). A withdrawal preview needs the grant and an `Idempotency-Key`: the same key replays
an identical preview for 60 s, an expired one needs a new key. The API sets no preview cap: every preview that
reaches the provider counts toward the wallet's 1,000 custody operations per rolling 24 h
(`provider_limit_reached` when spent; swap quotes do not count). The `quote`
object carries 1Click's figures, each only when it sent one: `amount_in`, `min_amount_in`,
`amount_out` (the estimate), `min_amount_out` (the guaranteed output), `withdraw_fee`, `refund_fee`,
`amount_in_usd` and `amount_out_usd`. Use it to show the user the expected output and to choose
`min_amount_out` (exact input) or `max_amount_in` (exact output, with room above `quote.amount_in`).

**Dry runs report the rules, they are not refused by them.** A quote or preview on a usable
account goes to 1Click and returns `200`. It also checks what the live request would face: the
policy's action switch and destination rule, `assets`, `limits.per_transaction`, the schedule at the time the request would run
(after `timelock_ms`), the USD budget, per-asset limits and `max_actions_per_hour`, then the balance.
Each one it would fail is a `warnings` entry whose `code` is the code the live request
would get (`policy_action_denied`, `policy_destination_denied`, `policy_asset_denied`,
`policy_transaction_limit_exceeded`, `policy_schedule_denied`, `spend_price_unavailable`, `spend_budget_exceeded`, `policy_limit_exceeded`,
`policy_action_limit_exceeded`, `policy_usage_unavailable`, `approval_backlog_full`, `insufficient_balance`; for a swap or
withdrawal whose fresh quote already misses the figure sent also `below_min_amount_out` and `above_max_amount_in`; for a limit order
also `amount_too_low`, `policy_price_deviation_exceeded`, `market_price_unavailable`).
`policy_schedule_denied` also carries `available_at`, when the schedule next
allows it; `insufficient_balance` carries `available_amount`, what the balance can still spend (atomic
units). Only the live request is refused with these codes. A `below_min_amount_out` or
`above_max_amount_in` warning means the quote already misses your floor or ceiling: raise the
ceiling, lower the floor, or expect the live request to be refused.

The request's own checks refuse the quote itself. An asset `GET /v1/tokens` does not list
(`unsupported_token`) or a recipient that is not a valid address (`invalid_address`) is a 400, as it
is for the live request: no quote can answer it. It is refused before any provider call.

An archived, deleted or frozen account gets no quote: an account that can do nothing has no use for
one. A swap quote and a withdrawal preview are both refused with 409 `agent_archived`,
`agent_deleted` or `wallet_frozen`, the code the live request gets. The refusal reaches no provider.

Before executing, read `warnings`. A rule or check code there means the live request will be refused
with that code: tell the user, reduce the amount, wait (until the exceeded cap's
`spend.budget.*.resets_at`; `available_at` for a schedule warning),
or ask the owner to change the policy (see [errors.md](errors.md)). A quote without such a warning
is not a guarantee: it is a snapshot, other work can use the budget first, and a timelocked request
is charged at dispatch.

What the rest of the answer holds:

- `spend.amount_usd`: what the budget would count for this request; `null` when it is uncounted or
  unpriced.
- `spend.budget`: `{ daily, weekly, monthly }`, each `{ limit_usd, spent_usd, remaining_usd,
  resets_at }` as `usage.budget` on `GET …/policy`, before this request; every field is `null` for
  an uncapped window.
- `fee`: `{ amount_usd, percent }` or `null`. A swap compares the input's value with the quoted
  output's value at current prices. A withdrawal on chain `near` is valued from the provider's
  `estimated_fee` when it is zero or its `fee_token` is the withdrawn asset, otherwise from
  `amount` minus `quote.amount_out` (on NEAR the route delivers the withdrawn asset itself). A
  withdrawal to another chain delivers the asset of the same symbol 1Click lists there and is
  valued from `amount` minus `quote.amount_out` in that asset; `null` when the quote states no
  `amount_out` (a public withdrawal's preview) or the chain lists no single asset of that symbol.
- `warnings`: `high_fee` when the route keeps more than 5%, judged only where `fee` is valued;
  `asset_delivered_as_is` when a
  non-USDC asset is withdrawn to chain `hypercore`: it lands in HyperCore spot as that asset, not as
  USDC perps margin. Swap to USDC first to fund perps. `provider_would_refuse` (withdrawal previews):
  the wallet provider's own dry run of the withdrawal says it would refuse it; its reason is in
  `detail`. Any rule or check code (above) means the live request would be refused with it. Show
  `detail` to the user.

## Execute

```http
POST /v1/agents/{agent_id}/swap
X-API-Key: <partner API key>
X-Grant-Token: ngt_…
Idempotency-Key: swap:user123:2026-10-07T10:00:00Z:7f3c
Content-Type: application/json

{ "origin_asset": "nep141:wrap.near", "destination_asset": "<usdc>", "amount": "1000000000000000000000000", "min_amount_out": "2900000" }
```

(Exact output sends `amount_out` and `max_amount_in` in place of `amount` and `min_amount_out`.)
Then poll `GET /v1/status?correlation_id=op_…&wait_ms=30000` (see
[status-and-recovery.md](status-and-recovery.md)). Typical paths:

- no timelock/approval: `PROCESSING` → `SUCCESS` (`details.amount_out`, `intent_hash`, `tx_hash`).
  `amount_out` is the delivered amount and is present only at `SUCCESS` (or `REFUNDED` after a
  partial fill); before then read `details.expected_amount_out`, and never treat it as final. A
  settled exact-output request also carries `details.amount_in` (the input actually spent; the rest of
  `max_amount_in` is back in the balance) and `details.quote` (the quote it ran with, the same
  figures as a dry run's `quote`)
- timelock: `QUEUED` (`details.execute_after`) → `PROCESSING` → …
- approval: `PENDING_APPROVAL` (`details.approval_id`) → owner votes → `PROCESSING` → …
- route refunded: `REFUNDED` (`details.refund_tx_hash`); never filled: `FAILED` with `failure_code`

The execution call holds the HTTP request open up to ~2 minutes while the provider answers; give
the client a ~150 s timeout (the SDK gives a live `swap`, `placeLimitOrder`, `withdraw`, `transfer` and `shield`
that budget). `withdraw` with `async: true` returns as soon as it is accepted.

## Swap

- Inside the account; funds never leave NEAR Intents. Needs no destination rule.
- `origin_asset` and `destination_asset` must differ (`validation_failed`).
- `min_amount_out`: atomic minimum output of the destination asset, sent to the provider and checked
  against a fresh quote's guaranteed output (`below_min_amount_out`, 409, before anything is signed).
  A swap the provider reports below it never reads `SUCCESS` but `NEEDS_REVIEW` with
  `amount_out_below_minimum` (`details.requested_min_amount_out`). `amount_out` with `max_amount_in`
  buys an exact amount instead (see [Exact input or exact output](#exact-input-or-exact-output)).
- `amount_too_low`: below the route minimum (confidential swaps have their own minimum).
  `meta.min_amount`, when the provider states it, is that minimum in atomic units of the asset
  sent: quote at least that. A refusal recorded on an operation keeps it as `details.min_amount`,
  so a replay under the same key or `GET /v1/status` states it too.
- `route_unavailable`: no route or liquidity for this pair or amount; nothing was submitted.
  Repeating the request won't help: try a different amount or pair under a new key.
- `provider_unavailable` (the provider or quote service did not answer): read `GET /v1/network`
  (`one_click`). With `meta.retryable`, repeat the same request with the same key after
  `Retry-After`; without it the refusal is recorded, so use a new key once `one_click` reports ok.

## Limit order

A resting order on the agent's confidential balance (`confidential: false` is refused). `SUCCESS`
means funded and resting, not filled: the order id is `details.order_id`. Placing, following and cancelling
one is covered in [limit-orders.md](limit-orders.md).

## Withdraw

- Leaves NEAR Intents to `recipient` on `chain`. Public and confidential withdrawals reach every
  `blockchain` value `GET /v1/tokens` lists (`near`, `eth`, `sol`, `btc`, `tron`, …).
- Pays from the confidential balance. `confidential: false` pays from the legacy public balance,
  the one place that flag is still accepted: use it to empty leftover public funds (or `shield`
  them). The policy's `withdraw` action and destination rule apply either way.
- No `memo`: a withdrawal request never carries one, and the policy's `only` rule matches a
  `withdraw` destination signed with `memo: null`.
- Exact input (`amount`, optional `min_amount_out`) or exact output (`amount_out` with
  `max_amount_in`); see [Exact input or exact output](#exact-input-or-exact-output). Exact input
  debits `amount`, and the recipient gets it less the network or bridge fee in the destination chain's
  units, so a payee owed an exact figure should be paid exact output.
- NEAR is `nep141:wrap.near` (`near` and `native` are refused with `unsupported_token`). Withdrawn
  to chain `near` it always arrives as **native NEAR**, from the public or the confidential
  balance, as from a NEAR wallet or 1Click: no `wrap.near` registration needed, and what exchanges
  and invoices expect. A confidential one is paid out by
  1Click, which unwraps it. There is no option to withdraw wNEAR to chain `near`.
- `unsupported_chain` means a chain the catalog doesn't list; `route_unavailable` means no
  liquidity for that asset now; `invalid_address` when the address is not valid for the chain or the account does not
  exist (a malformed one is refused before the policy is read). A token withdrawal to a NEAR account with no
  storage on that token's contract is `recipient_storage_unregistered` (409, not retryable), a preview too:
  register the receiver's storage on that token contract first (NEP-145 `storage_deposit`) and send again
  under a new key, or move the funds inside NEAR Intents.
- Moving funds to your own account on another chain or venue (your Hyperliquid or Polymarket
  address, say) counts against the USD budget like any withdrawal, and a deposit back does not
  credit it.
- A non-USDC asset withdrawn to chain `hypercore` lands in HyperCore spot as that asset, not as USDC
  perps margin (the preview warns `asset_delivered_as_is`). Swap to USDC first to fund perps.
- `details.destination_tx_hash` appears once the destination chain confirms.

## Transfer

- To another NEAR Intents account (`recipient` = NEAR account id), no bridge, near-instant.
  A recipient is a named account (`alice.near`), a 64-hex implicit account or a lowercase `0x`
  EVM address.
- A named recipient must exist on NEAR, else `400 invalid_address`: NEAR Intents credits any id, and
  funds sent to an unregistered name cannot be recovered. Implicit ids (64 hex, or `0x` + 40 hex)
  need no account. If NEAR RPC cannot answer, the transfer is refused with retryable
  `503 near_rpc_unavailable`; nothing is sent.
- Spends from the confidential balance to the recipient's confidential balance (the default).
  `confidential: false` is refused with `public_balance_exit_only`. Shield first if the agent's
  confidential balance is short.
- **near.com addresses.** near.com shows its users' Intents account as `nearcom:<account>`
  (e.g. `nearcom:0x20b6…2367`). The API takes only the bare account: strip `nearcom:` and
  lowercase the rest. near.com users see their confidential balance, which is where a transfer
  lands. A `nearcom:` address is not on any chain: never `withdraw` to it.

## Shield

Moves `amount` of `asset` from the public (legacy) balance into the confidential balance of the same
account. Read them with `GET …/balances` (confidential) and `GET …/balances?source=public`.
A shield counts toward the asset's per-asset limits and `max_actions_per_hour` like any other
action (`policy_limit_exceeded` / `policy_action_limit_exceeded`), even though nothing leaves the account.

`unshield` (confidential back to public) is still an endpoint but refuses every request with
`public_balance_exit_only`. Do not offer it; to get funds out of the account, `withdraw`.

## Deposit (fund the account)

```http
POST /v1/agents/{agent_id}/deposit
X-API-Key: <partner API key>
Idempotency-Key: dep:user123:<uuid>
{ "origin_asset": "<asset_id on the paying chain>", "amount": "25000000" }
```

- `origin_asset` is required: the token and chain the payer sends from.
- `amount` is optional. Without it, any payment ≥ `details.min_amount` before
  `details.expires_at` is credited; with it, the quote is exact.
- `destination_asset` optional: swap to another asset on arrival. The address credits the
  confidential balance; `confidential: false` is refused with `public_balance_exit_only`. The policy
  needs `confidential: true` (a stored policy without it gets `policy_action_denied`).
- There is **no `refund_to` request field**: refunds always go back to the agent account;
  `details.refund_to` reports where.
- Response `PENDING_DEPOSIT` with `details.deposit_address` (send funds **only** there),
  `details.memo` (include it if present), `details.min_amount`, `details.expires_at`. Once funds are
  seen, `details.deposit_seen_at` is set.
- Then poll the `op_…` id: `PENDING_DEPOSIT` → `PROCESSING` → `SUCCESS`. Only deposits report
  `PENDING_DEPOSIT`.
- Without `amount` an address stays open 3 days; with it, 24 h. Each lookup issues a new address.
- An address still unpaid one day after `details.expires_at` ends `FAILED` with `deposit_expired`.
  A deposit with a known deposit tx or amount never expires; it stays open until 1Click settles or
  refunds it. A late payment is refunded to `details.refund_to`, so `deposit_expired` loses nothing.
  A payment seen at an already expired address reopens the deposit (it drops `deposit_expired`)
  until 1Click settles or refunds it; the settlement deadline runs from `details.deposit_seen_at`,
  not from when the address was issued.
  A quote 1Click refuses in words the API does not map is `provider_refused` (400, `detail` quotes
  the reason); no address exists, so change the request and use a new key.
- Quota: 100 deposit addresses per agent per rolling 24 h → `deposit_quota_exceeded` (429).
- A deposit that has no `details.deposit_address` ten minutes after it was created never gets one and
  ends `FAILED` with `deposit_address_missing`; nothing can have been paid into it.
- Native NEAR-side funding: `GET …/addresses/near` returns the account; send NEP-141 tokens to it.

Never show the `correlation_id` as a payment address. Use a fresh idempotency key per deposit
request you want (reusing the key returns the same address).

## Reference, note and receipts

- **`reference`** (transfer, withdraw): your label for the payment, such as an order id; 1 to 128
  printable ASCII characters, no spaces. Recorded with the request and proved by the receipt; never
  sent to the provider or on chain, and not checked for uniqueness. Part of the request, so the same
  `Idempotency-Key` with another `reference` is `idempotency_conflict`.
- **`note`** (swap, limit order, withdraw, transfer, deposit): the agent's own words for why it runs
  this, 1 to 500 characters on one line. For the owner only: it shows in `details.request.note`, the
  receipts list and the audit evidence, and never reaches the provider, the chain or a receipt.
  Nothing checks it is true; part of the request like `reference`. Dry runs accept and ignore both.
- **`usd_value`**: a spending execution's status details carry the API's USD valuation of the amount
  at dispatch (absent when no price was available). Not a quote or a settlement price.
- **Receipts.** A completed transfer, withdrawal, swap, limit order or deposit has one:
  `GET /v1/agents/{agent_id}/operations/{correlation_id}/receipt`, and
  `GET /v1/agents/{agent_id}/receipts` (newest first; `type` = `transfer`, `withdraw`, `swap`,
  `limit_order` or `deposit`; page with `cursor`, a page can be short while `next_cursor` is set).
  Each item is `{ correlation_id, type, status, details, receipt }`: `status` is `PROVEN`,
  `PENDING` (the next checkpoint will cover it) or `UNLOGGED` (before the log); `receipt` is null
  until `PROVEN`. `details` is the account's own view (adds `usd_value`, `note`). The `receipt`
  itself holds only the settled terms (recipient, asset, `reference`, chain, settlement hashes, and
  the amounts: `amount` with `min_amount_out?` and a reported `amount_out?` for exact input, or
  `amount_out` and `max_amount_in` for exact output; `memo` only on a withdrawal receipt issued
  before withdrawals dropped it), no agent, account or `note`, so it is safe to hand to a payee. It does not
  prove who paid; a limit order's proves it was funded and placed, not filled. Anything else
  (unfinished, shield, unshield, native signing) answers 409 `receipt_not_available`. Receipts last
  as long as the audit log (90 days). Verify one with the SDK:
  [status-and-recovery.md](status-and-recovery.md#receipts).

## Recover (rare)

Only for `UNCERTAIN` operations that provably never reached the provider (no
`provider_request_id`, no `dispatch_committed_at`). Same API key, same grant token, same
`Idempotency-Key`, and `request` = the original body plus `type`. See
[status-and-recovery.md](status-and-recovery.md).

## Pre-flight checklist for an agent tool

1. `GET …/policy` in force? action allowed? destination listed? budget `remaining_usd` enough?
2. Balance enough? Read `available` / `available_raw` on `GET …/balances` (confidential, the account
   the request spends from; `?source=public` only for a legacy balance to shield or withdraw). That is
   the balance less queued or timelocked work not yet dispatched; don't subtract queued work yourself.
3. Dry quote (swap/limit order/withdraw) and show it, with `fee` and `warnings`. For exact output,
   set `max_amount_in` from `quote.amount_in` plus 0.5 to 1% room. A rule or check code in
   `warnings` means the live call would be refused with it: stop or adjust before sending. A refused
   quote (`wallet_frozen`, `agent_archived`, `agent_deleted`) means the account can do nothing.
4. Create + persist the idempotency key and body, then send, with a `reference` for a payment tied
   to an order and a `note` saying why.
5. Poll to a stable status, report `SUCCESS`/`REFUNDED`/`FAILED` with hashes; report an amount
   received (`amount_out`) only from `SUCCESS` (an exact-output request: and the input spent, `amount_in`); for `UNCERTAIN` say "outcome pending" and keep
   observing. After `SUCCESS` a payee can be given the receipt.
