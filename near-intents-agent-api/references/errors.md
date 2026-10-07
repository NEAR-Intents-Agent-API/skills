# Errors

Every error is a JSON:API document:

```json
{
  "errors": [{
    "status": "409",
    "code": "policy_revision_conflict",
    "title": "Policy revision conflict",
    "detail": "The policy changed since you read it. Read GET /agents/{agent_id}/policy and generate again with its revision.",
    "source": { "pointer": "/expected_revision" },
    "meta": { "retryable": false, "available_at": "2026-10-07T10:10:00Z" }
  }],
  "meta": { "request_id": "…" }
}
```

- Branch on `errors[0].code`. `title`/`detail` are for humans and may change.
- Validation failures (`400 validation_failed`) return **one entry per field** with
  `source.pointer`; show all of them.
- `meta.retryable: true` → the same request (same idempotency key and body) can succeed later.
- `meta.available_at` → when a cooldown/quota/throttle lifts. 429/503 also send `Retry-After`.
- Unknown code → fall back to the HTTP status class. Always log `meta.request_id`.

## Decision table

| Code | HTTP | What to do |
|---|---|---|
| **Auth & request** | | |
| `invalid_api_key` | 401 | wrong or revoked key, stray whitespace, or a base URL from a different deployment; fix config, don't retry |
| `validation_failed` | 400 | fix the fields in `source.pointer` |
| `invalid_json`, `invalid_cursor` | 400 | fix the request |
| `idempotency_key_required` | 400 | add `Idempotency-Key` |
| `idempotency_conflict` | 409 | same key used with a different body; resend original body or use a new key for a genuinely new request |
| `rate_limited` | 429 | wait `Retry-After`, retry the same request |
| `database_busy` | 503 | retry with the **same** idempotency key; check status first |
| **Refusals (layer named; only that layer can lift it)** | | |
| `agent_grant_required` | 403 | missing/revoked/expired/foreign grant → issue a grant or use the right token |
| `policy_action_denied` | 403 | action (or confidential form) not in policy → owner `policy_update` |
| `policy_destination_denied` | 403 | address not allowed → owner adds it to `destinations` |
| `policy_schedule_denied` | 403 | outside the owner's `schedule` when it would run (after the timelock) → submit again at `meta.available_at` with a **new** key, or owner changes `schedule` |
| `spend_budget_exceeded` | 403 | shared USD budget used up → wait for window (`usage.budget.*.resets_at`) or owner raises cap; retry with a **new** key |
| `policy_denied` | 409 | provider-side policy refused (asset, per-asset limit, hourly rate, …) → owner `policy_update` |
| `wallet_frozen` | 409 | owner froze the account → only `agent_unfreeze` resumes |
| `policy_not_ready` | 409 | latest policy not yet in force → wait for the policy intent to settle |
| `signing_policy_denied` | 403 | recipient not in the policy's `sign.recipients` |
| `signing_recipient_forbidden` | 403 | `intents.near`/`intents.far` are never signing recipients |
| **Execution / routing** | | |
| `insufficient_balance` | 409 | fund the account or lower amount |
| `amount_too_low` | 400 | below route minimum → quote a larger amount |
| `route_unavailable` | 400 | no route/liquidity now; nothing submitted → other pair or later, new key |
| `quote_unavailable` | 503 | upstream quote timed out; nothing submitted → retry shortly, new key; check `/v1/network` |
| `unsupported_chain`, `unsupported_token`, `invalid_address` | 400/409 | change chain/asset/address |
| `wallet_busy` | 409 | another money op on this wallet is running → retry with a new key after it finishes |
| `spend_price_unavailable` | 503 | no fresh USD price for budget accounting → retry later, new key |
| `authorization_stale` | 409 | grant/key revoked or policy/owner changed between admission and dispatch → re-evaluate |
| `deposit_quota_exceeded` | 429 | 500 deposit addresses / 24 h → wait `available_at` |
| **Provider (custody/1Click) state** | | |
| `provider_unavailable` | 503 | provider down/breaker open → wait; read `/v1/network` |
| `provider_rate_limited` | 429 | wait `Retry-After` |
| `provider_refused` | 409 | provider refused for its own reason → inspect `detail`, don't blind-retry |
| `provider_response_invalid`, `provider_request_mismatch` | 502 | treat outcome as unknown → poll status of the same id |
| **Owner intents** | | |
| `intent_expired` | 409 | generate a new intent |
| `intent_payload_mismatch`, `intent_standard_mismatch`, `intent_type_mismatch` | 409 | submit exactly what was generated, with the right standard/type |
| `signature_invalid`, `signer_mismatch`, `owner_proof_invalid` | 400/401/409 | wrong key, altered bytes, or wrong encoding (see [signing.md](signing.md)) |
| `owner_key_not_full_access` | 403 | NEAR owner must sign with a FullAccess key |
| `owner_nonce_invalid` | 409 | intent used/expired → regenerate |
| `intent_generation_recovery_required` | 409 | retry `generate-intent` with the same idempotency key |
| `policy_revision_conflict` | 409 | re-read policy, regenerate with new `expected_revision`, re-sign |
| `policy_reconciliation_required` | 409 | another policy intent still open → wait |
| `policy_change_throttled`, `agent_unfreeze_throttled` | 429 | cooldown → wait `available_at` |
| `wallet_already_frozen`, `wallet_already_unfrozen` | 409 | no-op; state already as requested |
| `owner_type_unsupported`, `owner_approval_unsupported`, `approval_vote_owner_unsupported` | 400/409 | feature needs another owner type (approval = NEAR only) |
| `agent_creation_quota_exceeded` | 429 | 100 agents/24 h → wait or request quota |
| `agent_not_bound` | 409 | onboarding not finished → wait for `agent_create` SUCCESS |
| `agent_archived`, `agent_deleted`, `agent_deletion_pending` | 409 | lifecycle state forbids it; restore or stop |
| `sponsor_busy`, `sponsor_balance_insufficient` | 503 | retry the same submit later |
| `transaction_unconfirmed` | 503 | broadcast but unconfirmed → poll status; do not resubmit |
| `execution_not_cancellable` | 409 | op already left `QUEUED` |
| `approval_closed`, `approval_already_voted`, `approval_not_allowed` | 409 | read the approval's current state |
| **Grants** | | |
| `grant_limit_reached` | 409 | 50 live grants → revoke unused |
| `grant_credential_reused` | 409 | new token per grant |
| `grant_expiry_invalid` | 409 | `expires_at` must be future and ≤365 days |
| **Recovery** | | |
| `operation_not_recoverable` | 409 | not UNCERTAIN, already dispatched, non-recoverable kind, or other API key → keep polling |
| `operation_recovery_unavailable` | 409 | another recovery running → poll |
| **Not found** | 404 | `agent_not_found`, `grant_not_found`, `intent_not_found`, `operation_not_found`, `status_not_found`, `approval_not_found` |
| **Disabled** | | |
| `transparency_log_disabled` | 501 | this deployment keeps no transparency log; operation proofs are unavailable |

## Retry rules

1. **Client timeout or network error on a write**: you don't know if it landed. Retry with the
   **same** `Idempotency-Key` and identical body (safe), or read status if you got a
   `correlation_id`. Never invent a new key for the same intent to "unstick" it.
2. **Refusal codes (403, most 409)**: the request is wrong for the current state. Retrying the
   same thing won't help; change state (policy, grant, balance) or the request.
3. **"Nothing was submitted" codes** (`route_unavailable`, `quote_unavailable`,
   `spend_budget_exceeded`, `policy_schedule_denied`, `spend_price_unavailable`, `wallet_busy`): the API rolled back; a
   later attempt is a new request → **new** idempotency key.
4. **Throttles** (429): wait until `available_at` / `Retry-After`.
5. **Status-level outcomes** (`FAILED`, `REFUNDED`, `UNCERTAIN`, `NEEDS_REVIEW`) are not HTTP
   errors; see [status-and-recovery.md](status-and-recovery.md).

## Wrapping errors in your code

TypeScript SDK: `AgentApiError { status, code, title, detail, retryable, availableAt, requestId,
errors, idempotencyKey }`; transport failures throw `AgentApiRequestError { idempotencyKey }`.
The Python/Rust assets mirror this (`AgentApiError.code`, `.available_at`, …; Rust
`Error::Api { … }` with `error.code()`).

For AI agents, convert errors into short tool results that name the layer and the fix, e.g.
`"refused by account policy: withdraw is not enabled. Ask the owner to allow withdrawals."` —
never echo API keys, grant tokens, or full request headers into model context.
