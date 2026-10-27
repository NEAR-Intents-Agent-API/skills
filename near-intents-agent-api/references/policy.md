# Policy

One complete, owner-signed rulebook per agent account. Every grant acts within it. Each
`agent_create` and `policy_update` carries the **whole** policy, never a diff; every field below
is required except `schedule` and `native`.

```ts
Policy = {
  frozen: boolean,
  actions: ("swap" | "transfer" | "withdraw")[],          // unique; deposits are always allowed
  confidential: boolean,                                  // shield/unshield + confidential form of each action
  owner_approval: boolean,                                // every outgoing money action waits for the owner's vote
  assets: "any" | string[],                               // asset_id values, ≤128; "*" is refused, use "any"
  limits: {                                               // per-asset caps, atomic strings (1–78 digits); keys are asset_ids, never "*"
    per_transaction?: Record<asset_id, string>,
    hourly?:          Record<asset_id, string>,
    daily?:           Record<asset_id, string>,
    monthly?:         Record<asset_id, string>,
  },
  max_actions_per_hour: number | null,                    // positive int ≤ 1_000_000, null = no cap
  destinations: { mode: "only" | "except" | "any", list: Destination[] },   // list ≤256; "any" ⇒ []
  budget: { daily_usd: string | null, weekly_usd: string | null, monthly_usd: string | null },
  timelock_ms: number,                                    // 0 … 2_592_000_000 (30 days)
  schedule?: Schedule,                                    // omit = money actions may run any time
  native?: Native                                         // omit = every native method off
}

Native = {                                                // when present, every key is required
  near:   { sign_message: string[], transfer: boolean, call: string[] },   // 0–256 entries each: NEAR accounts (sign_message also takes domains)
  evm:    { sign_typed_data: EvmTypedDataDomain[], sign_message: boolean, sign_transaction: boolean },
  solana: { sign_message: boolean, sign_transaction: boolean },
}

EvmTypedDataDomain = {                                    // ≥1 domain field, 0–64 entries unique by domain
  name?: string, version?: string, chain_id?: number | string,
  verifying_contract?: string, salt?: string,             // lowercase 0x
  primary_types?: string[],                               // 1–32 unique message types; omit = any type
}

Destination =
  | { action: "withdraw", chain: string, address: string, memo: string | null }   // external chain address
  | { action: "transfer", address: string, confidential: boolean }                // NEAR Intents account

Schedule = {
  mode: "only" | "except",                                // run only inside the windows, or pause inside them
  time_zone: string,                                      // IANA zone, e.g. "Europe/Berlin", "UTC"
  windows: {                                              // 1 … 28
    days: ("mon" | "tue" | "wed" | "thu" | "fri" | "sat" | "sun")[],   // unique, 1 … 7
    start: string,                                        // "HH:MM", 24-hour, inclusive
    end: string,                                          // "HH:MM" or "24:00", exclusive, ≠ start
  }[]
}
```

Unknown fields are rejected (strict objects). Field names are snake_case.

## Field by field

| Field | Effect | Enforced by |
|---|---|---|
| `frozen` | kill switch: every money action refused (`wallet_frozen`). Use `agent_freeze`/`agent_unfreeze` intents, not a policy edit | provider |
| `actions` | which money actions exist at all. Missing → `policy_action_denied` | provider |
| `confidential` | enables `shield`, `unshield`, and `confidential: true` on swap/withdraw/transfer | provider |
| `owner_approval` | outgoing actions go `PENDING_APPROVAL` until the owner votes; deposits unaffected. Hard limits still reject first | provider |
| `assets` | tokens the agent may spend (`"any"` or explicit ids; `"*"` is not an asset id and is refused with `validation_failed`, here and as a `limits` key). Swaps may buy unlisted tokens but not sell them | provider |
| `limits` | per-asset caps per transaction and per rolling hour/day/month. Needs named assets to be meaningful | provider; windows also **API** |
| `max_actions_per_hour` | rate cap on money actions regardless of amount | provider and **API** |
| `destinations` | where withdrawals/transfers may go | **API** |
| `budget` | total USD value across all assets per day / week / month, counted from when each cap was turned on; shared by every grant | **API** |
| `timelock_ms` | every outgoing money action is `QUEUED` this long; owner can `execution_cancel` | **API** |
| `schedule` | weekly windows on the owner's clock when money actions may run (`only`) or are paused (`except`) | **API** |

`budget` notes: charged atomically at dispatch using 1Click USD prices; an asset with no fresh
price is refused with `spend_price_unavailable` (only while a cap is on; with none on, nothing is counted and no price is read). A larger budget never
lifts a per-asset limit; both must pass. Live usage is in `GET …/policy` →
`usage.budget.{daily,weekly,monthly}.{limit_usd, spent_usd, remaining_usd, resets_at}`, all `null`
for an uncapped window, which counts nothing. Over budget,
a new execution that runs now is a plain refusal (`spend_budget_exceeded`): no operation, no `FAILED`
status, no charge. `FAILED` is recorded only for a request that loses a concurrent race, or a timelocked one released over budget. Each cap's
periods run from the moment the applied policy turned it on (daily: every 24 h from then, on at
05:00 UTC → resets at 05:00 UTC; weekly: every 7 days; monthly: the same date and time each month in
UTC, clamped to the last day of shorter months, Jan 31 → Feb 28 → Mar 31): `resets_at` is when the
current period ends and the whole cap is back. A new amount keeps the period; turning a cap off and on starts a new one. Periods are fixed, not
rolling, so up to twice a cap can move across a reset ($100 at 04:59 and $100 at 05:00 under a
$100 daily cap on from 05:00); a smaller shorter-window cap, per-asset hourly `limits` or
`timelock_ms` bound that burst. Moving funds
to your own account on another chain or venue (your Hyperliquid or Polymarket address) counts like
any withdrawal, and a deposit back does not credit it. Swap quotes and withdrawal previews
(`dry: true`) charge nothing: they report the budget as `spend.budget` and warn with
`spend_budget_exceeded` when the live request would be refused.

`limits` notes: the API also counts hourly/daily/monthly limits and `max_actions_per_hour` per
agent, shared by every grant, at dispatch. Swaps, withdrawals, transfers, **shield and unshield**
all count; `near` and `nep141:wrap.near` share one count. Pending/uncertain actions count until
settled; a completed one counts for the full window after completion; removing a limit keeps the
usage. Over → `policy_limit_exceeded` / `policy_action_limit_exceeded` (403); earlier actions
without a count → `policy_usage_unavailable` (409). Remaining per-asset allowance is not readable. These are plain refusals of an execution that runs now (no
operation, no `FAILED` status, no charge; a timelocked one is judged, and can end `FAILED`, when released). Dry-run quotes and previews are not refused for these;
they carry the same codes in `warnings`.

`destinations` notes:
- `only` + `[]` (the safe default) keeps funds inside the account: swaps only.
- `only` matches the **exact** entry, memo included. `except` blocks an account under any memo.
- EVM addresses (including `hyperevm`, `adi` and `hood`) must be lowercase hex in the entry.
- `only` and `except` compare addresses in canonical form: bech32 and bech32m (`btc`, `ltc`, Cardano
  `addr1`, Zcash `zs1`/`u1`, `aleo`) and Bitcoin Cash CashAddr are lowercase, with or without
  `bitcoincash:`; `starknet`, `sui`, `aptos` and `movement` hex is lowercase and zero-padded to
  64 digits. Base58 addresses keep their exact case.
- `transfer` entries are NEAR Intents account ids (`alice.near`, implicit hex accounts).

`schedule` notes:
- A window is `start` inclusive, `end` exclusive, in `time_zone`. `00:00`–`24:00` is a whole day.
- An `end` earlier than `start` runs past midnight and belongs to the day it starts on:
  `{"days":["fri"],"start":"22:00","end":"02:00"}` covers Friday night into Saturday morning.
- Daylight saving follows the time zone: a window means the same local hours all year.
- Judged when the action would **run**: at submit for `now + timelock_ms`, again when the timelock
  releases, and again at dispatch. A timelocked action whose release falls outside the schedule
  fails; nothing is sent.
- Refusal: `403 policy_schedule_denied` with `meta.available_at` (next allowed moment) and
  `Retry-After`. Nothing was recorded: submit the same request again at `available_at`, or the owner
  changes `schedule`.
- Exempt: deposits, `native` requests and owner approval votes.
- Not sent to the provider: a schedule-only edit is an off-chain owner signature.

## Native accounts (`native`)

`native` lets every grant use the agent's own keys outside NEAR Intents, one method at a time.
Omit it and every native method is off. A policy update replaces the whole policy, so leaving
`native` out of an update turns it off.

| Switch | Route (`POST /v1/agents/{agent_id}/native/…`) | What it allows |
|---|---|---|
| `near.sign_message: ["app.example.com"]` | `near/sign-message` | NEP-413 message signing (site logins) for a listed recipient (NEAR account or domain) |
| `near.transfer: true` | `near/transfer` (Idempotency-Key) | native NEAR from `near_account_id` to any account |
| `near.call: ["app.example.near"]` | `near/call` (Idempotency-Key) | a function call on a listed contract |
| `evm.sign_typed_data: [domain, …]` | `evm/sign-typed-data` | EIP-712 v4 under a listed domain only, and only the entry's `primary_types` if it lists them |
| `evm.sign_message: true` | `evm/sign-message` | EIP-191 message |
| `evm.sign_transaction: true` | `evm/sign-transaction` | any EVM transaction (one serialized unsigned EIP-2718 or EIP-155 transaction for `chain`; anything else is `validation_failed`) |
| `solana.sign_message` / `solana.sign_transaction` | `solana/sign-message` / `solana/sign-transaction` | Solana message / transaction (`sign-transaction` takes one legacy or v0 transaction message; anything else is `validation_failed`) |

- **The switch is the whole fence.** `destinations`, `budget`, `timelock_ms`, `schedule` and
  `owner_approval` do not apply. A native NEAR transfer can go anywhere; a typed-data signature
  (a permit, a `TransferWithAuthorization`) or a signed transaction can move everything at that
  EVM or Solana address. Warn the owner before they sign it on; prefer `sign_typed_data` with
  narrow domains over `sign_transaction`.
- **Typed-data domains** match field for field: a request's `domain` must equal one entry, a
  missing or extra field is another domain. `chainId` may be a number, decimal or hex string.
  `types.EIP712Domain` may be omitted (derived); when sent it declares exactly those fields in
  EIP-712 order with standard types. `primaryType: "EIP712Domain"` is refused. Hyperliquid's
  `Exchange` domain (trading) never signs a `HyperliquidSignTransaction` withdrawal.
- **`primary_types`** narrows the message types signed under an entry's domain: 1 to 32 unique
  strings, each 1–256 chars, case-sensitive, never `EIP712Domain`; an empty list is invalid. Omitted
  = any message type under the domain. A request matches when its domain equals the entry's domain
  field for field **and** (no `primary_types` or its `primaryType` is listed); otherwise
  `403 native_method_denied`. Entries are unique by domain: one domain listed twice with different
  types is refused, so give one entry all its types.
  ```json
  { "name": "HyperliquidSignTransaction", "version": "1", "chain_id": 42161,
    "verifying_contract": "0x0000000000000000000000000000000000000000",
    "primary_types": ["HyperliquidTransaction:Withdraw"] }
  ```
  Under that domain Hyperliquid also signs `HyperliquidTransaction:SpotSend` and
  `HyperliquidTransaction:UsdSend`, which pay any address. `primary_types` narrows the **kind** of
  message, never its fields: an allowed `HyperliquidTransaction:Withdraw` can still name any
  destination and amount.
- **NEP-413 signing:** `near/sign-message` signs `{ message, recipient, nonce }` under the NEP-413
  tag, never a transaction or an intent. `recipient` is a NEAR account or domain (printable ASCII,
  ≤ 128), matched exactly against the list, no wildcard. The caller supplies the 32-byte `nonce`
  (base64); `callbackUrl` is not supported. The signature (base64) is verified before it returns.
- `intents.near` and `intents.far` are never accepted: listing them is `validation_failed`, a
  transfer or call to them `403 native_contract_forbidden`, a signature for them
  `403 signing_recipient_forbidden`.
- The wallet provider treats native NEAR as wNEAR: a transfer or call deposit needs `assets` to
  allow `nep141:wrap.near` (or `any`) and stays within its `limits`.
- Signing routes return the signature and never broadcast. NEAR transfer and call are sent by the
  wallet provider and read like any execution (`GET /v1/status`, type `native`).
- A typed-data record (status and history, type `native`) also holds `details.domain` (the matched
  EIP-712 domain, in the policy's spelling) and `details.message_json` (the signed `message` as
  canonical JSON, keys sorted, a string); both are `null` for other native methods. A typed-data
  `message` over 65536 bytes as canonical JSON is refused with `validation_failed`.
- Freezing the account stops every native route. Changing `native` is an on-chain policy update.
- Addresses: `wallet.near_account_id`, `wallet.evm_account_id` (same on every EVM chain) and
  `wallet.solana_account_id`; the last two are null until the provider reports them.

## Ready-made policies

**Swap-only trading assistant** (funds never leave the account):

```json
{
  "frozen": false, "actions": ["swap"], "confidential": false, "owner_approval": false,
  "assets": ["nep141:wrap.near", "<usdc asset_id>"],
  "limits": { "per_transaction": { "nep141:wrap.near": "10000000000000000000000000" } },
  "max_actions_per_hour": 30,
  "destinations": { "mode": "only", "list": [] },
  "budget": { "daily_usd": "100", "weekly_usd": null, "monthly_usd": "1000" },
  "timelock_ms": 0
}
```

**Payroll / payouts to an allow-list with a review window**:

```json
{
  "frozen": false, "actions": ["transfer", "withdraw"], "confidential": false, "owner_approval": false,
  "assets": ["<usdc asset_id>"],
  "limits": { "per_transaction": { "<usdc asset_id>": "500000000" }, "daily": { "<usdc asset_id>": "2000000000" } },
  "max_actions_per_hour": 10,
  "destinations": { "mode": "only", "list": [
    { "action": "transfer", "address": "vendor.near", "confidential": false },
    { "action": "withdraw", "chain": "base", "address": "0xabc…", "memo": null }
  ] },
  "budget": { "daily_usd": "2000", "weekly_usd": "5000", "monthly_usd": null },
  "timelock_ms": 3600000
}
```

**Business hours only** (any policy plus a schedule; here Mon–Fri 09:00–17:00 Berlin time):

```json
"schedule": {
  "mode": "only", "time_zone": "Europe/Berlin",
  "windows": [{ "days": ["mon","tue","wed","thu","fri"], "start": "09:00", "end": "17:00" }]
}
```

Pause overnight instead: `{"mode":"except","time_zone":"UTC","windows":[{"days":["mon","tue","wed","thu","fri","sat","sun"],"start":"22:00","end":"07:00"}]}`.

**Human-in-the-loop** (any owner type): as above with `"owner_approval": true`.

**Broad, capped by USD**: `"actions": ["swap","transfer","withdraw"]`, `"assets": "any"`,
`"limits": {}`, `"destinations": {"mode":"any","list":[]}`, budget set. Only for owners who
explicitly accept that the agent can send anywhere.

## Changing a policy

1. `view = GET /v1/agents/{id}/policy`. Require `status == "APPLIED"`.
2. Check `GET /v1/agents/{id}` → `cooldowns.policy_change_available_at` is null or past.
3. Build the complete new policy from `view.policy` (to remove a schedule, omit `schedule`).
4. `generate-intent {type: "policy_update", agent_id, policy, expected_revision: view.revision}`.
5. Owner signs (`nep413`/`eip712` when only API-enforced fields changed, otherwise on chain).
6. Submit, wait `SUCCESS`, then poll `GET …/policy` until `APPLIED` and
   `provider_policy_synced: true`. Until then executions fail with `policy_not_ready`.

Cooldowns (per agent, default 5 minutes):
- any ordinary change (anything other than a pure freeze/unfreeze) → next ordinary change waits
  → `policy_change_throttled` with `meta.available_at`;
- `agent_freeze` is never throttled;
- an unfreeze waits 5 minutes after the previous unfreeze → `agent_unfreeze_throttled`;
- `agent_create` does not start a cooldown, so a freshly created agent can be edited at once.

Plan UX around it: batch edits into one signature, and show `available_at` instead of a
generic error.

## Validation errors

A malformed policy returns `400 validation_failed` with one error per field
(`source.pointer` like `/policy/destinations/list/0/address`). Other policy-related codes:
`policy_revision_conflict`, `policy_reconciliation_required`,
`policy_blocks_delete` (an old-shape policy must be re-signed before deletion).
