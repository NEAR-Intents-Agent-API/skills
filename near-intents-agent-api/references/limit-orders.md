# Limit orders

A limit order is a resting order on NEAR Intents: the agent names a price, the order is funded in
full when placed, and it sits on the book until it fills, is cancelled or reaches its `deadline`.
Use a swap when the agent wants to trade now at the current quote; use a limit order when it wants a
specific price and can wait. Fills and refunds return to the balance that funded the order.

These are the Agent API's orders on the agent's own balance. They are not 1Click's separate
"limit orders" product for confidential 1Click swaps. They run on the agent's confidential balance.

## Endpoints

| Call | Auth | Notes |
|---|---|---|
| `POST /v1/agents/{agent_id}/limit-orders` | key, grant, idem | live: `202` `StatusResponse` (`type: "limit_order"`); `dry: true`: key only, `200` quote |
| `GET /v1/agents/{agent_id}/limit-orders` | key | `?open=true&limit=<1-100>&offset=<n>` → `{ data: [...] }`, newest first. Offset-paged, no `next_cursor`; a page shorter than `limit` is the last. Does not refresh order status |
| `GET /v1/agents/{agent_id}/limit-orders/{order_id}` | key | the order live from 1Click |
| `POST /v1/agents/{agent_id}/limit-orders/{order_id}/cancel` | key, grant | no `Idempotency-Key`; unfilled part refunded |
| `POST /v1/agents/{agent_id}/limit-orders/cancel-all` | key, grant | no key; at most 50 orders per call → `{ known, cancelled, failed, remaining, complete }` |

SDK 0.8.1: `placeLimitOrder`, `listLimitOrders`, `getLimitOrder`, `cancelLimitOrder`,
`cancelAllLimitOrders` ([typescript.md](typescript.md#limit-orders-sdk-081)). Python asset:
`quote_limit_order`, `place_limit_order`, `list_limit_orders`, `get_limit_order`, `cancel_limit_order`,
`cancel_all_limit_orders`. Rust asset: the same names, with `execute(id, "limit-orders", …)` placing.

## Request

```json
POST /v1/agents/{agent_id}/limit-orders
{ "base_asset": "nep141:wrap.near", "quote_asset": "<usdt asset_id>", "side": "sell",
  "quantity": "1000000000000000000000000", "price": "5.2",
  "deadline": "2026-11-12T10:00:00Z", "dry": true }
```

- `base_asset` / `quote_asset`: `asset_id` values from `GET /v1/tokens`, different from each other
  (`unsupported_token` / `validation_failed` otherwise).
- `side` and `quantity` (atomic units of `base_asset`): a **sell** spends `quantity` of the base for
  the quote; a **buy** spends `ceil(quantity × price)` of the quote and receives `quantity` of the
  base at the full price. The spent amount is `quantity × price × 10^quote_decimals / 10^base_decimals`,
  rounded up: 1 NEAR (`1e24`) at `price: "5.2"` against a 6-decimal quote asset is `5200000`.
- `price`: quote units per one **whole** base unit, a plain positive decimal string (no exponent,
  sign or leading zeros, at most 40 characters). Never a float.
- `deadline`: optional RFC 3339, in the future and at most 7 days ahead; omitted means 7 days.
  A later `deadline` is `validation_failed` at `/deadline`.
- The order is funded from the confidential balance and pays back there (the default; never a
  fallback to the public balance). `confidential: false` is refused with `public_balance_exit_only`
  (400, not retryable, live and `dry`); a resting public order placed earlier still fills, reads and
  cancels. The policy needs `confidential: true` as well as `limit_order`.
- `note`: optional, 1 to 500 characters on one line, the agent's own words for why; owner-only, see
  [executions.md](executions.md#reference-note-and-receipts).

## The agent's procedure

1. **Check the rules.** `GET …/policy` lists `limit_order` in `actions` and has `confidential: true`; note `limit_order.max_deviation_bps` (default 100) and the budget left.
2. **Dry first.** `dry: true` needs only the API key (no grant, no key) and makes no provider call.
   It answers `200` with `spend` (USD value and budget left), `quote` (`spend_asset`, `spend_amount`,
   `receive_asset`, `min_amount_out`: the least a full fill delivers before 1Click's fee) and
   `warnings`: each rule the live order would fail, under its own code (`policy_action_denied`,
   `policy_asset_denied`, `policy_schedule_denied`, `spend_budget_exceeded`, `insufficient_balance`,
   `amount_too_low`, `policy_price_deviation_exceeded`, `market_price_unavailable`, …). A warning
   means the live order will be refused with that code: fix the price or amount first. `owner_approval:
   true` means the live order will wait in `PENDING_APPROVAL`.
3. **Place it.** Persist a new `Idempotency-Key` with the body, then send with the grant. Same-key
   retries are safe; `UNCERTAIN` is polled, never resubmitted, and `recover` takes `type: "limit_order"`
   ([status-and-recovery.md](status-and-recovery.md)).
4. **`SUCCESS` means resting, not filled.** The operation ends once the order is funded and on the
   book; `details.order_id` names it. Never tell the user the trade happened. Its receipt proves the
   order was funded and placed, not filled.
5. **Follow it.** `GET …/limit-orders/{order_id}` is live: `fill_status` (`awaiting_deposit`, `open`,
   `partially_filled`, `filled`, `canceled`, `expired`, …), `partial_fills`, `payouts`,
   `is_payout_status_final` (true once nothing more can be paid out or refunded) and `app_fees`.
   The list endpoint does not refresh status; use it to find ids (`open=true` keeps resting orders),
   then read each one. Poll slowly (minutes), not in a tight loop.
6. **Cancel when it should stop.** `cancel` returns the order's fields (all optional, `{}` when the
   provider sent none): read the order afterwards for its state. For several, repeat `cancel-all`
   until `complete` is true:

   ```ts
   let round = await agent.cancelAllLimitOrders(agentId);
   while (!round.complete) round = await agent.cancelAllLimitOrders(agentId);
   ```

## Rules that apply

- **Same rules as a swap on the asset it spends**: `assets`, per-asset `limits`, `max_actions_per_hour`
  (each placement counts, cancels do not), the shared USD budget, `timelock_ms`, `schedule` and
  `owner_approval`. `destinations` does not apply. The budget and limits are charged when the order is
  placed; a cancel does **not** give the budget charge back.
- **Price bound.** The market price is the base asset's USD price over the quote asset's. A sell may be
  priced at most `limit_order.max_deviation_bps` below it, a buy at most that far above (0 to 300,
  default 100); a better price is never limited. Over the bound: `policy_price_deviation_exceeded`
  (403, `/price`). No fresh price for either asset: `market_price_unavailable` (503,
  `/base_asset` or `/quote_asset`).
- **When the price is judged.** At placement, and again when a timelock or an owner approval
  releases the order. If the market price is briefly unavailable when a timelocked order is released,
  the API retries it later instead of failing it. A request held for approval whose approval
  deadline passes or whose grant, policy or owner lapses settles `FAILED` (`expired` /
  `authorization_stale`, `details.never_executed`, charge released).
- **Minimum 0.1 USD.** An order worth less is `amount_too_low` (400, `/quantity`); for a sell,
  `meta.min_amount` is the least `quantity` that passes. The check is skipped when the spent asset
  has no fresh USD price; the provider then applies its own minimum (`amount_too_low`, 409).
- **Fee.** 1Click charges its own app fee on every order (20 bps today, in `app_fees`). It is not an
  API or OutLayer fee.
- **Custody quota.** Placements and cancels count toward the wallet's 1,000 custody operations per
  rolling 24 h (`provider_limit_reached`); the dry preview and reads do not.

## When the owner pulls the plug

- **Freeze, or a policy that drops `limit_order`**: the API cancels the agent's resting orders (it
  repeats the batch cancel until none remain, best effort with a retry sweep). Do not place more; read
  `GET …/limit-orders?open=true` to see what is left and wait for `is_payout_status_final`.
- **Cancelling is never refused by the policy**: it works while frozen and after `limit_order` is
  dropped, with no timelock, schedule or approval. It needs a **live grant** (revoking the placing
  grant does not cancel its orders, and a revoked or expired token cannot cancel: `agent_grant_required`).
  An archived or deleted agent is refused.
- **Delete guard.** `agent_delete` is refused with `limit_orders_open` (409) while any order is not
  payout-final, checked before `deletion_assets_remaining`; `preview.deletion.open_limit_orders` counts
  them. Cancel (or freeze), let the refunds land, move the funds out, then delete.

## Errors and what to do

| Code | Layer | Do |
|---|---|---|
| `policy_action_denied` | policy | `limit_order` is not in `actions`, or the stored policy has `confidential` off: ask the owner |
| `public_balance_exit_only` | request | `confidential: false` was sent: never retry it; omit `confidential` |
| `policy_price_deviation_exceeded` | policy | re-price nearer the market (the detail names it), or the owner raises `limit_order.max_deviation_bps` |
| `market_price_unavailable` | policy | an asset has no fresh price; no `retryable` flag, nothing recorded: try again later under a new key |
| `amount_too_low` | request | below 0.1 USD (400) or the provider minimum (409): raise `quantity` (`meta.min_amount` for a sell) |
| `insufficient_balance` | balance | fund the spent asset, or send at most `meta.available_amount` |
| `spend_budget_exceeded`, `policy_limit_exceeded`, `policy_action_limit_exceeded`, `policy_schedule_denied` | budget / policy | wait for the window (`resets_at`, `available_at`) or ask the owner |
| `unsupported_token`, `validation_failed` | request | fix the asset id, price format, `deadline` or same-asset pair |
| `limit_order_not_found` | request | not this agent's order id: list the orders |
| `agent_grant_required` | grant | a cancel or placement needs a live grant |
| `wallet_busy`, `provider_rate_limited`, `provider_limit_reached`, `provider_unavailable` | provider | see [errors.md](errors.md): `meta.retryable` → same request, same key; for a cancel just call again |
| `limit_orders_open` | api | `agent_delete` only: cancel the orders and let them settle |

Full catalogue and retry rules: [errors.md](errors.md).
