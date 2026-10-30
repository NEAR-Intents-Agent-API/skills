# Errors

Every error is a JSON:API document:

```json
{
  "errors": [{
    "status": "409",
    "code": "policy_revision_conflict",
    "layer": "policy",
    "title": "Policy revision conflict",
    "detail": "The policy changed since you read it. Read GET /agents/{agent_id}/policy and generate again with its revision.",
    "source": { "pointer": "/expected_revision" },
    "meta": { "retryable": false, "available_at": "2026-10-07T10:10:00Z" }
  }],
  "meta": { "request_id": "…" }
}
```

- Branch on `errors[0].code`. `title`/`detail` are for humans and may change.
- `errors[0].layer` (SDK 0.7: `error.layer`) says whose rule refused: see [Layers](#layers) for what to do in each.
- Validation failures (`400 validation_failed`) return **one entry per field** with
  `source.pointer`; show all of them.
- `meta.retryable: true` → the same request (same idempotency key and body) can succeed after
  `Retry-After`. Server failures carry it too: they may have taken effect, so only the same key is
  safe. Absent → repeating will not help now. It says **nothing** about whether an earlier
  attempt took effect; a new key is allowed only after a proven refusal (retry rule 4). Never
  decide from `detail`.
- `meta.available_at` → when a cooldown/quota/throttle lifts. 429/503 also send `Retry-After`.
- `meta.min_amount` → on `amount_too_low` only, when the provider states the route minimum as a
  whole number: that minimum in atomic units of the asset sent. Quote at least that. Absent → the
  provider gave no number; quote a larger amount. A refusal recorded on an operation keeps it as
  `details.min_amount`.
- `meta.available_amount` (SDK 0.7: `error.availableAmount`) → on `insufficient_balance` only, when known: what the balance can still
  spend, in atomic units (the balances route's `available_raw`). Send at most that, or fund.
- Unknown code → fall back to its `layer` and the HTTP status class. Always log `meta.request_id`.

## Layers

| `layer` | Whose rule | Your move |
|---|---|---|
| `request` | the request: shape, ids, assets, addresses, amount minimums, idempotency, resource state | fix the request; repeating it unchanged won't help |
| `auth` | API key, owner signature, nonce, intent | fix the credential, or regenerate and re-sign the intent |
| `grant` | the grant that admits the agent's calls | use a live grant's token, or issue one |
| `policy` | the account policy and its state (actions, assets, destinations, schedule, limits, freeze, archive) | ask the owner (`policy_update`, `agent_unfreeze`), or wait for the schedule/window |
| `budget` | the shared USD budget and its price | ask the owner to raise a cap, or wait until `resets_at`; for `spend_price_unavailable` wait for a price |
| `balance` | the agent's balance | fund the agent, or send a smaller amount (`meta.available_amount`) |
| `provider` | custody provider, 1Click, routing | `meta.retryable` → same request, same key, after `Retry-After`; else change the pair/amount, or wait; nothing was sent |
| `api` | this service: internal, database, sponsor, RPC, quotas, rate limits | `meta.retryable` → same request, same key, after `Retry-After`; else try later and give support the `request_id` |

## Decision table

| Code | Layer | HTTP | What to do |
|---|---|---|---|
| **Auth & request** | | | |
| `invalid_api_key` | `auth` | 401 | wrong or revoked key, stray whitespace, or a base URL from a different deployment; fix config, don't retry |
| `validation_failed` | `request` | 400 | fix the fields in `source.pointer` (also: an amount that moves funds is `0`, a swap's two assets are the same, a native `sign-transaction` body is not one transaction) |
| `invalid_json`, `invalid_cursor` | `request` | 400 | fix the request |
| `idempotency_key_required` | `request` | 400 | add `Idempotency-Key` |
| `idempotency_conflict` | `request` | 409 | same key used with a different body; resend original body or use a new key for a genuinely new request |
| `rate_limited` | `api` | 429 | wait `Retry-After`, retry the same request. Every attempt counts against its key, refused ones too, so a retry loop keeps its own key limited; only attempts within the key's allowance count against the tenant |
| `database_busy` | `api` | 503 | retryable; may have taken effect → retry with the **same** idempotency key, never a new one |
| **Refusals (layer named; only that layer can lift it)** | | | |
| `agent_grant_required` | `grant` | 403 | missing/revoked/expired/foreign grant → issue a grant or use the right token |
| `policy_action_denied` | `policy` | 403 | action (or confidential form) not in policy → owner `policy_update` |
| `policy_transaction_limit_exceeded` | `policy` | 403 | amount above `limits.per_transaction` for the asset it spends (native NEAR and wNEAR share one cap). Plain refusal, nothing recorded → send less, or owner raises it |
| `policy_destination_denied` | `policy` | 403 | address not allowed → owner adds it to `destinations`. Native NEAR a `native/near/transfer` sends, or a `native/near/call` attaches as deposit, is judged as a `withdraw` on chain `near` |
| `policy_schedule_denied` | `policy` | 403 | outside the owner's `schedule` when it would run (after the timelock); nothing recorded → submit the same request at `meta.available_at`, or owner changes `schedule` |
| `spend_budget_exceeded` | `budget` | 403 | shared USD budget used up. A plain refusal for a request that runs now: no operation, no `FAILED` status, no charge (a timelocked one released over budget ends `FAILED`). Wait for the period to end (`usage.budget.*.resets_at`, or `spend.budget` on a dry run: the whole cap is back then) or owner raises cap, then send the request again |
| `policy_limit_exceeded` | `policy` | 403 | the asset's hourly/daily/monthly limit would be passed (shield/unshield count too). Plain refusal, nothing recorded → wait for the window or owner raises it, then send again |
| `policy_action_limit_exceeded` | `policy` | 403 | `max_actions_per_hour` used up. Plain refusal → wait for the hour or owner raises it, then send again |
| `policy_usage_unavailable` | `policy` | 409 | earlier actions have no per-asset count and a limit applies; nothing recorded or sent → clears when they leave the window, or the operator reconciles them |
| `policy_denied` | `policy` | 409 | provider-side policy refused (asset, per-asset limit, hourly rate, …) → owner `policy_update` |
| `wallet_frozen` | `policy` | 409 | owner froze the account → only `agent_unfreeze` resumes |
| `policy_not_ready` | `policy` | 409 | latest policy not yet in force → wait for the policy intent to settle |
| `native_method_denied` | `policy` | 403 | the policy's `native` switch for this route is off, or the contract/domain is not listed, or the domain entry's `primary_types` omits the request's `primaryType` → owner `policy_update` |
| `native_contract_forbidden` | `policy` | 403 | native transfer to or call of `intents.near`/`intents.far`; never allowed |
| `signing_policy_denied` | `policy` | 403 | `near/sign-message` recipient not in `native.near.sign_message` |
| `signing_recipient_forbidden` | `policy` | 403 | `intents.near`/`intents.far` are never signing recipients |
| **Execution / routing** | | | |
| `insufficient_balance` | `balance` | 400/409 | the balance the request spends from (public or confidential) holds less than it spends; nothing submitted or charged. `meta.available_amount` is what it can still spend → fund the account or send at most that. The API's own check is 400 with no history row; a provider refusal is 409 |
| `amount_too_low` | `request` | 400 | below route minimum → quote at least `meta.min_amount` (atomic units of the asset sent) when present, else a larger amount |
| `route_unavailable` | `provider` | 409 (400 on deposit) | no route or liquidity for this pair or amount; nothing submitted. Repeating the request won't help → try a different amount or pair under a new key |
| `unsupported_chain`, `unsupported_token`, `invalid_address` | `request` | 400/409 | change chain/asset/address. `invalid_address`: the recipient is not a valid address for the chain (refused before the policy is read), the provider refused it, or a `transfer` recipient is a named NEAR account that does not exist. Nothing was sent |
| `near_rpc_unavailable` | `api` | 503 | `transfer` only: NEAR RPC did not answer, so a named recipient could not be checked; retryable, nothing sent → retry the same request (same key) shortly |
| `wallet_busy` | `provider` | 409 | another money op on this wallet is running; nothing sent, refusal recorded (not retryable) → after it finishes, a **new** key |
| `spend_price_unavailable` | `budget` | 503 | no fresh USD price for budget accounting. Plain refusal, nothing recorded → send again once `GET /v1/tokens` shows a price |
| `authorization_stale` | `auth` | 409 | grant/key revoked or policy/owner changed between admission and dispatch → re-evaluate |
| `deposit_quota_exceeded` | `api` | 429 | 100 deposit addresses / 24 h for this agent → wait `available_at` |
| **Provider (wallet service/1Click) state** | | | |
| `provider_unavailable` | `provider` | 503 | the provider or the quote service behind it did not answer → read `/v1/network` (`one_click`). With `meta.retryable`: same request, same key, after `Retry-After`. Without it the refusal is recorded (nothing submitted): new key once the provider is ok |
| `provider_limit_reached` | `provider` | 429 | custody provider's daily quota for this wallet: 1,000 operations / rolling 24 h, `dry` previews included, quotes not; nothing sent → new work at `available_at` (don't burn the quota on repeated previews) |
| `provider_rate_limited` | `provider` | 429 | a provider's rate limiter refused before running; nothing sent; always `Retry-After: 10`. With `meta.retryable` (reads, and calls that took a key): same request, same key, after it. Without it the refusal is recorded → new key after the wait |
| `provider_refused` | `provider` | 409 (400 on a deposit quote) | the provider refused these terms; on a quote `detail` quotes 1Click's reason. Nothing sent or charged, repeating won't help → change the request, new key |
| `provider_response_invalid`, `provider_request_mismatch` | `provider` | 502 | treat outcome as unknown → poll status of the same id |
| **Owner intents** | | | |
| `intent_expired` | `auth` | 409 | generate a new intent |
| `intent_payload_mismatch`, `intent_standard_mismatch`, `intent_type_mismatch` | `auth`/`request` | 409 | submit exactly what was generated, with the right standard/type |
| `signature_invalid` | `auth` | 400 | malformed wallet output → submit exactly what the wallet returned |
| `signer_mismatch`, `owner_proof_invalid`, `policy_delegate_signature_invalid` | `auth` | 401/409 | wrong key or altered bytes (see [signing.md](signing.md)) |
| `owner_key_not_full_access` | `auth` | 403 | NEAR owner must sign with a FullAccess key |
| `owner_nonce_invalid` | `auth` | 409 | intent used/expired → regenerate |
| `owner_command_stale` | `auth` | 409 | signed for an earlier owner state, or the signature of a deletion that was later restored → generate a new intent and sign again |
| `deletion_assets_remaining` | `balance` | 409 | `agent_delete`: the wallet holds a public or confidential balance, including funds that arrived since the owner signed (the API reads the balances again right before committing; an unreadable balance there is `provider_unavailable`). Nothing was retired; a failure at that final read is recorded `failed` and the agent stays archived and restorable → withdraw the funds, then delete again |
| `intent_generation_recovery_required` | `api` | 409 | retry `generate-intent` with the same idempotency key |
| `policy_revision_conflict` | `policy` | 409 | re-read policy, regenerate with new `expected_revision`, re-sign |
| `policy_reconciliation_required` | `policy` | 409 | another policy intent still open → wait |
| `policy_change_throttled`, `agent_unfreeze_throttled` | `policy` | 429 | cooldown → wait `available_at` |
| `wallet_already_frozen`, `wallet_already_unfrozen` | `request` | 409 | no-op; state already as requested |
| `owner_type_unsupported` | `auth` | 400 | this deployment does not accept that owner type for new agents (check `supported_owner_types` on `GET /v1/network`; EVM is off by default until the final EVM wallet build) |
| `agent_creation_quota_exceeded` | `api` | 429 | 100 agents/24 h → wait or request quota |
| `agent_not_bound` | `request` | 409 | onboarding not finished → wait for `agent_create` SUCCESS |
| `agent_archived`, `agent_deleted`, `agent_deletion_pending` | `policy` | 409 | lifecycle state forbids it; restore or stop |
| `pause_before_archive`, `unpause_before_restore` | `request` | 409 | `agent_freeze` before `agent_archive`; `agent_unfreeze` before `agent_restore` |
| `sponsor_busy`, `sponsor_balance_insufficient` | `api` | 503 | retry the same submit later |
| `transaction_unconfirmed` | `api` | 503 | retryable; broadcast but unconfirmed → poll status, or repeat with the **same** key (never sends twice); never as new work |
| `execution_not_cancellable` | `request` | 409 | op already left `QUEUED` |
| `approval_closed`, `approval_already_voted`, `approval_not_allowed` | `request`/`auth` | 409 | read the approval's current state |
| `approval_signature_invalid` | `auth` | 401 | the vote was not signed by the owner's key or wallet → regenerate and sign again |
| `approval_vote_unsupported` | `provider` | 409 | the provider lists no such owner wallet build on this network → ask the operator; nothing was stored |
| **Grants** | | | |
| `grant_limit_reached` | `grant` | 409 | 50 live grants → revoke unused |
| `grant_credential_reused` | `grant` | 409 | new token per grant |
| `grant_expiry_invalid` | `grant` | 409 | `expires_at` must be future and ≤365 days |
| **Recovery** | | | |
| `operation_not_recoverable` | `api` | 409 | not UNCERTAIN, already dispatched, non-recoverable kind, or other API key → keep polling |
| `operation_recovery_unavailable` | `api` | 409 | another recovery running → poll |
| `agent_not_found`, `grant_not_found`, `intent_not_found`, `operation_not_found`, `approval_not_found` | `request` | 404 | wrong id or another account's; pass `correlation_id` exactly as returned |
| **Disabled** | | | |
| `transparency_log_disabled` | `api` | 501 | this deployment keeps no transparency log; operation proofs are unavailable |

**Native NEAR outcomes are statuses, not HTTP errors.** A `native/near/transfer` or `native/near/call` reads
`SUCCESS` only once two RPC operators agree NEAR finalized a transaction that is exactly the request;
until then `PROCESSING`. A finalized failure is `FAILED` with `native_transaction_failed` and the chain's reason (never resent). A transaction that is not the request is `NEEDS_REVIEW` with
`native_transaction_mismatch`; one NEAR still does not know 30 minutes after dispatch is `NEEDS_REVIEW` with
`native_transaction_unconfirmed`. Stop and review those; see [status-and-recovery.md](status-and-recovery.md).

**Dry runs warn with the live codes.** The account's rules do not refuse a swap quote or withdrawal
preview (`dry: true`): it returns `200`. Each one the live request would fail is a `warnings[]` entry
whose `code` is the live code: `policy_action_denied`, `policy_destination_denied`,
`policy_asset_denied`, `policy_transaction_limit_exceeded`, `policy_schedule_denied` (with `available_at`), `spend_price_unavailable`,
`spend_budget_exceeded`, `policy_limit_exceeded`, `policy_action_limit_exceeded`,
`policy_usage_unavailable` or `insufficient_balance` (with `available_amount`), with the same meaning
and the same layer to change. Such a code in `warnings` means the live request will be refused with
it: tell the user, reduce the amount, wait, or ask the owner to change the policy. An asset
`GET /v1/tokens` does not list (`unsupported_token`) or an invalid recipient (`invalid_address`)
refuses the quote itself with 400, before any provider call. A
withdrawal preview that reaches the provider counts toward the wallet's 1,000 custody
operations per rolling 24 h, rule warning or not. An archived, deleted or frozen
account gets no quote: a swap quote and a withdrawal preview are both refused with 409
`agent_archived`, `agent_deleted` or `wallet_frozen`, reach no provider.

## Retry rules

1. **Client timeout or network error on a write**: you don't know if it landed. Retry with the
   **same** `Idempotency-Key` and identical body (safe), or read status if you got a
   `correlation_id`. Never invent a new key for the same intent to "unstick" it.
2. **Refusal codes (403, most 409)**: the request is wrong for the current state. Retrying the
   same thing won't help; change state (policy, grant, balance) or the request. Let `layer` pick
   the move: `policy`/`budget` → ask the owner; `balance` → fund or send less; `request` → fix it;
   `provider` + `meta.retryable` → same key, later.
3. **Server failures** (`internal_error` 500, `database_busy` 503, `transaction_unconfirmed` 503,
   all with `meta.retryable`): the request may have taken effect. Repeat it with the **same** key,
   which replays or reconciles it, or read status. Never a new key: that can spend twice.
4. **Proven refusals** (`route_unavailable`, `wallet_busy`, a recorded `provider_unavailable`, and
   the policy/grant/validation refusals in the decision table): nothing was sent. A refusal
   recorded as an operation's outcome replays under its key, so a later attempt is a new request →
   **new** idempotency key. Budget and limit refusals of a new execution
   (`spend_budget_exceeded`, `spend_price_unavailable`, `policy_limit_exceeded`,
   `policy_action_limit_exceeded`, `policy_usage_unavailable`) and the API's own `insufficient_balance` (400) are
   plain refusals: no operation, no `FAILED` status, no charge; the same key is judged again and refuses
   again while the cause lasts. `FAILED` is recorded only for a request that loses a concurrent race, or a timelocked one released over budget or a limit (its key then replays that outcome). Only proven
   refusals permit a new key. A `FAILED` or `REFUNDED`
   status is **not** such proof: a partial failure is `FAILED`. Read its `details` and the
   balances, and send new work only as an explicit decision, never as an automatic retry.
5. **Earlier work under the key** (`intent_generation_recovery_required`,
   `operation_not_recoverable`, `operation_not_pending`, any error from `/recover`, and any error
   after a server failure or lost response): the original outcome may still be unknown. Keep the original key and
   `correlation_id`, read status, and never start the same work under a new key.
6. **Throttles** (429): wait until `available_at` / `Retry-After`.
7. **Status-level outcomes** (`FAILED`, `REFUNDED`, `UNCERTAIN`, `NEEDS_REVIEW`) are not HTTP
   errors; see [status-and-recovery.md](status-and-recovery.md).

## Wrapping errors in your code

TypeScript SDK: `AgentApiError { status, code, title, detail, retryable, availableAt, retryAfterSeconds,
minAmount, requestId, errors, idempotencyKey, layer, availableAmount }` (raw HTTP: `errors[0].layer`, `errors[0].meta.available_amount`,
the `Retry-After` header; `retryAfterSeconds` from SDK 0.7.2 is that header in whole seconds, set when the response sent one; `minAmount` from SDK 0.7; earlier versions read
`errors[0].meta.min_amount`); transport failures throw `AgentApiRequestError { idempotencyKey }`.
The Python/Rust assets mirror this (`AgentApiError.code`, `.available_at`, …; Rust
`Error::Api { … }` with `error.code()`).

For AI agents, convert errors into short tool results that name the layer and the fix, e.g.
`"refused by account policy: withdraw is not enabled. Ask the owner to allow withdrawals."` —
never echo API keys, grant tokens, or full request headers into model context.
