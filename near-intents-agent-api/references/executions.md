# Executions: moving money

All are `POST /v1/agents/{agent_id}/<action>` and return `202` with a `StatusResponse`
(`correlation_id: "op_…"`), except dry quotes (`200`).

| Action | Headers | Body | Policy needs |
|---|---|---|---|
| `swap` | key, grant, idem | `origin_asset`, `destination_asset`, `amount`, `min_amount_out?`, `confidential?`, `dry?` | `swap` (+`confidential`) |
| `withdraw` | key, grant, idem | `asset`, `amount`, `chain`, `recipient`, `memo?`, `confidential?`, `async?`, `dry?` | `withdraw`, destination allowed |
| `transfer` | key, grant, idem | `asset`, `amount`, `recipient`, `confidential?` | `transfer`, destination allowed |
| `shield` / `unshield` | key, grant, idem | `asset`, `amount` | `confidential: true` |
| `deposit` | key, idem | `origin_asset`, `destination_asset?`, `amount?`, `confidential?` | nothing (always allowed) |
| `recover` | key, grant, original idem | `correlation_id`, `request: {type, …original body}` | re-authorized in full |

`amount` is atomic (integer string) and must be greater than zero for every action that moves funds
(`validation_failed` otherwise). Asset ids come from `GET /v1/tokens`. A recipient that is not a valid
address for its chain is refused with `400 invalid_address` before the policy is read.

## Quote first

```json
POST …/swap      { "origin_asset": "nep141:wrap.near", "destination_asset": "<usdc>", "amount": "1000000000000000000000000", "dry": true }
POST …/withdraw  { "asset": "<usdc>", "amount": "5000000", "chain": "base", "recipient": "0xabc…", "dry": true }
→ 200 { "dry": true, "type": "swap" | "withdraw", "quote": { …1Click quote, snake_cased… },
        "spend": { "amount_usd": string | null, "budget": { "daily": {…}, "weekly": {…}, "monthly": {…} } },
        "fee": { "amount_usd": string, "percent": string } | null,
        "warnings": [ { "code": "high_fee" | "asset_delivered_as_is"
                              | "policy_action_denied" | "policy_destination_denied" | "policy_asset_denied"
                              | "policy_schedule_denied" | "spend_price_unavailable" | "spend_budget_exceeded"
                              | "policy_limit_exceeded" | "policy_action_limit_exceeded" | "policy_usage_unavailable"
                              | "unsupported_token" | "invalid_address" | "insufficient_balance",
                        "detail": string, "available_at"?: string, "available_amount"?: string } ],
        "owner_approval": boolean }   // true: the live request waits in PENDING_APPROVAL for the owner
```

Dry runs charge no budget, reserve none and move nothing. A swap quote needs no grant and no
idempotency key. A withdrawal preview needs the grant and an `Idempotency-Key`: the same key replays
an identical preview for 60 s, an expired one needs a new key, and each agent gets 20 previews per
rolling 24 h. Every preview that reaches the provider uses one, failed ones and ones carrying a rule
warning included (`withdraw_preview_limit_reached`, 429 with `meta.available_at` and `Retry-After` for
when the oldest attempt leaves the window); a refusal the provider states before it quotes
(e.g. `route_unavailable`) gives the attempt back, an outage does not. The `quote`
object belongs to 1Click (amount out, fees, time estimate); treat its fields as informational
and do not hard-code its shape. Use it to show the user expected output and to choose
`min_amount_out` (minimum output).

**Dry runs report the rules and checks, they are not refused by them.** A quote or preview on a usable
account always goes to 1Click and returns `200`. It also checks what the live request would face: the
policy's action switch and destination rule, `assets`, the schedule at the time the request would run
(after `timelock_ms`), the USD budget, per-asset limits and `max_actions_per_hour`, then the request's
own checks. Each one it would fail is a `warnings` entry whose `code` is the code the live request
would get (`policy_action_denied`, `policy_destination_denied`, `policy_asset_denied`,
`policy_schedule_denied`, `spend_price_unavailable`, `spend_budget_exceeded`, `policy_limit_exceeded`,
`policy_action_limit_exceeded`, `policy_usage_unavailable`, `unsupported_token`, `invalid_address`,
`insufficient_balance`). `policy_schedule_denied` also carries `available_at`, when the schedule next
allows it; `insufficient_balance` carries `available_amount`, what the balance can still spend (atomic
units). Only the live request is refused with these codes.

An archived, deleted or frozen account gets no quote: an account that can do nothing has no use for
one. A swap quote and a withdrawal preview are both refused with 409 `agent_archived`,
`agent_deleted` or `wallet_frozen`, the code the live request gets. The refusal reaches no provider
and uses no preview attempt.

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
  resets_at }` as `usage.budget` on `GET …/policy`, before this request.
- `fee`: `{ amount_usd, percent }` or `null`. A swap compares the input's value with the quoted
  output's value at current prices. A withdrawal is valued only on chain `near`, from the
  provider's `estimated_fee` when it is zero or its `fee_token` is the withdrawn asset. A
  withdrawal to any other chain is always `null`: its `quote.amount_out` is in the destination
  chain's asset, which the preview does not name, so compare `amount` with `amount_out` yourself.
- `warnings`: `high_fee` when the route keeps more than 5%, judged only where `fee` is valued
  (never for a bridged withdrawal); `asset_delivered_as_is` when a
  non-USDC asset is withdrawn to chain `hypercore`: it lands in HyperCore spot as that asset, not as
  USDC perps margin. Swap to USDC first to fund perps. Any rule or check code (above) means the live
  request would be refused with it. Show `detail` to the user.

## Execute

```http
POST /v1/agents/{agent_id}/swap
X-API-Key: naa_…
X-Grant-Token: ngt_…
Idempotency-Key: swap:user123:2026-10-07T10:00:00Z:7f3c
Content-Type: application/json

{ "origin_asset": "nep141:wrap.near", "destination_asset": "<usdc>", "amount": "1000000000000000000000000", "min_amount_out": "2900000" }
```

Then poll `GET /v1/status?correlation_id=op_…&wait_ms=30000` (see
[status-and-recovery.md](status-and-recovery.md)). Typical paths:

- no timelock/approval: `PROCESSING` → `SUCCESS` (`details.amount_out`, `intent_hash`, `tx_hash`).
  `amount_out` is the delivered amount and is present only at `SUCCESS` (or `REFUNDED` after a
  partial fill); before then read `details.expected_amount_out`, and never treat it as final
- timelock: `QUEUED` (`details.execute_after`) → `PROCESSING` → …
- approval: `PENDING_APPROVAL` (`details.approval_id`) → owner votes → `PROCESSING` → …
- route refunded: `REFUNDED` (`details.refund_tx_hash`); never filled: `FAILED` with `failure_code`

The execution call holds the HTTP request open up to ~2 minutes while the provider answers; give
the client a ~150 s timeout. `withdraw` with `async: true` returns as soon as it is accepted.

## Swap

- Inside the account; funds never leave NEAR Intents. Needs no destination rule.
- `origin_asset` and `destination_asset` must differ (`validation_failed`).
- `min_amount_out`: atomic minimum output of the destination asset, sent to the provider. The
  quote does not apply it; a swap the provider reports below it never reads `SUCCESS` but
  `NEEDS_REVIEW` with `amount_out_below_minimum` (`details.requested_min_amount_out`).
- `amount_too_low`: below the route minimum (confidential swaps have their own minimum).
  `meta.min_amount`, when the provider states it, is that minimum in atomic units of the asset
  sent: quote at least that. A refusal recorded on an operation keeps it as `details.min_amount`,
  so a replay under the same key or `GET /v1/status` states it too.
- `route_unavailable`: no route or liquidity for this pair or amount; nothing was submitted.
  Repeating the request won't help: try a different amount or pair under a new key.
- `provider_unavailable` (the provider or quote service did not answer): read `GET /v1/network`
  (`one_click`). With `meta.retryable`, repeat the same request with the same key after
  `Retry-After`; without it the refusal is recorded, so use a new key once `one_click` reports ok.

## Withdraw

- Leaves NEAR Intents to `recipient` on `chain`. Public and confidential withdrawals reach every
  `blockchain` value `GET /v1/tokens` lists (`near`, `eth`, `sol`, `btc`, `tron`, …).
- `memo` for chains/exchanges that need one (the policy's `only` rule matches memo exactly).
- NEAR (`nep141:wrap.near`, or `near`) withdrawn to chain `near` always arrives as **native NEAR**,
  from the public or the confidential balance, as from a NEAR wallet or 1Click: no `wrap.near`
  registration needed, and what exchanges and invoices expect. A confidential one is paid out by
  1Click, which unwraps it. There is no option to withdraw wNEAR to chain `near`.
- `unsupported_chain` means a chain the catalog doesn't list; `route_unavailable` means no
  liquidity for that asset now; `invalid_address` when the address is not valid for the chain or the provider rejects it
  (a malformed one is refused before the policy is read).
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
- `confidential: true` spends from and to confidential balances. Shield first if the agent's
  confidential balance is short.
- **near.com addresses.** near.com shows its users' Intents account as `nearcom:<account>`
  (e.g. `nearcom:0x20b6…2367`). The API takes only the bare account: strip `nearcom:` and
  lowercase the rest. near.com users see their confidential balance, so send them a
  `confidential: true` transfer by default. A public transfer still works when the user asks for
  one, but it lands in the recipient's legacy public balance, which near.com does not show by
  default: tell the user before sending. A `nearcom:` address is not on any chain: never
  `withdraw` to it.

## Shield / unshield

Move `amount` of `asset` between the public and confidential balance of the same account.
Read both with `GET …/balances?source=public|confidential`.
Each move counts toward the asset's per-asset limits and `max_actions_per_hour` like any other
action (`policy_limit_exceeded` / `policy_action_limit_exceeded`), even though nothing leaves the account.

## Deposit (fund the account)

```http
POST /v1/agents/{agent_id}/deposit
X-API-Key: naa_…
Idempotency-Key: dep:user123:<uuid>
{ "origin_asset": "<asset_id on the paying chain>", "amount": "25000000" }
```

- `origin_asset` is required: the token and chain the payer sends from.
- `amount` is optional. Without it, any payment ≥ `details.min_amount` before
  `details.expires_at` is credited; with it, the quote is exact.
- `destination_asset` optional: swap to another asset on arrival. `confidential: true` credits the
  confidential balance.
- There is **no `refund_to` request field**: refunds always go back to the agent account;
  `details.refund_to` reports where.
- Response `PENDING_DEPOSIT` with `details.deposit_address` (send funds **only** there),
  `details.memo` (include it if present), `details.min_amount`, `details.expires_at`.
- Then poll the `op_…` id: `PENDING_DEPOSIT` → `PROCESSING` → `SUCCESS`. Only deposits report
  `PENDING_DEPOSIT`.
- Without `amount` an address stays open 3 days; with it, 24 h. Each lookup issues a new address.
- An address still unpaid one day after `details.expires_at` ends `FAILED` with `deposit_expired`.
  A deposit with a known deposit tx or amount never expires; it stays open until 1Click settles or
  refunds it. A late payment is refunded to `details.refund_to`, so `deposit_expired` loses nothing.
  A quote 1Click refuses in words the API does not map is `provider_refused` (400, `detail` quotes
  the reason); no address exists, so change the request and use a new key.
- Quota: 100 deposit addresses per agent per rolling 24 h → `deposit_quota_exceeded` (429).
- A deposit that has no `details.deposit_address` ten minutes after it was created never gets one and
  ends `FAILED` with `deposit_address_missing`; nothing can have been paid into it.
- Native NEAR-side funding: `GET …/addresses/near` returns the account; send NEP-141 tokens to it.

Never show the `correlation_id` as a payment address. Use a fresh idempotency key per deposit
request you want (reusing the key returns the same address).

## Recover (rare)

Only for `UNCERTAIN` operations that provably never reached the provider (no
`provider_request_id`, no `dispatch_committed_at`). Same API key, same grant token, same
`Idempotency-Key`, and `request` = the original body plus `type`. See
[status-and-recovery.md](status-and-recovery.md).

## Pre-flight checklist for an agent tool

1. `GET …/policy` in force? action allowed? destination listed? budget `remaining_usd` enough?
2. Balance enough? Read `available` / `available_raw` on `GET …/balances` (balance less queued or
   timelocked work not yet dispatched); don't subtract queued work yourself.
3. Dry quote (swap/withdraw) and show it, with `fee` and `warnings`. A rule or check code in
   `warnings` means the live call would be refused with it: stop or adjust before sending. A refused
   quote (`wallet_frozen`, `agent_archived`, `agent_deleted`) means the account can do nothing.
4. Create + persist the idempotency key and body, then send.
5. Poll to a stable status, report `SUCCESS`/`REFUNDED`/`FAILED` with hashes; report an amount
   received (`amount_out`) only from `SUCCESS`; for `UNCERTAIN` say "outcome pending" and keep
   observing.
