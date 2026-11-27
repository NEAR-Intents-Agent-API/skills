# Owner intents

Every change the owner authorizes uses one loop:

```
backend: POST /v1/generate-intent {type, …}      → { correlation_id, intent, preview, expires_at, signer }
browser: show preview → wallet signs intent.payload (method from intent.standard)
backend: POST /v1/submit-intent {type, correlation_id, signed_data: {...intent, <wallet output>}}
backend: GET /v1/status?correlation_id=…&wait_ms=30000   until SUCCESS / FAILED
```

The API builds the exact bytes, so the partner never assembles anything security-relevant;
`preview.summary` is one human sentence describing what is signed.

## Intent types

| type | Extra fields (besides `agent_id`) | Notes |
|---|---|---|
| `agent_create` | `name`, `external_user_id?`, `owner`, `policy`, `grant?` (`label`, `credential`, `expires_at`; no `agent_id`) | reserves a `PENDING` agent and returns its `agent_id` immediately. With `grant`, the one signature also authorizes the first grant: see [below](#agent_create-with-a-first-grant) |
| `policy_update` | `policy` (complete), `expected_revision` (current applied revision) | 10-min cooldown between ordinary changes |
| `agent_freeze` | — | emergency stop; immediate; never throttled; the API then cancels the agent's open limit orders |
| `agent_unfreeze` | — | own cooldown (`agent_unfreeze_throttled`) |
| `agent_archive` / `agent_restore` | — | hides/unhides from default lists; nothing changes on chain. `agent_archive` needs a frozen account, and `agent_restore` needs an unfrozen one; otherwise `pause_before_archive` / `unpause_before_restore` |
| `agent_delete` | — | `preview.deletion` lists balances and `assets_lost`; withdraw first, there is no sweep. `open_limit_orders` counts resting limit orders; while any is not payout-final it is refused with `limit_orders_open` (checked before `deletion_assets_remaining`): cancel them, or freeze, which cancels them, and let them settle. It does not read the custody account's native NEAR (that goes to the sponsor, `beneficiary`) and does not inventory native accounts (native NEAR, EVM, Solana): tell the owner to move native funds first. Any public or confidential balance refuses it (`deletion_assets_remaining`), whatever the preview showed when the owner signed. The API reads the balances again right before it commits: funds that arrived since fail the deletion with `deletion_assets_remaining`, an unreadable balance with `provider_unavailable`. Both are recorded `failed`, nothing is retired and the agent stays archived and restorable. A signature from a deletion that was later restored is refused with `owner_command_stale`: sign again. Open deposit addresses never block it: deletion closes them (`agent_deleted`), and funds sent later are lost |
| `grant_issue` | `label`, `credential` (sha256 hex of token, or an `oidc` / `public_key` object), `expires_at` (≤365 days) | see [grants.md](grants.md) |
| `grant_revoke` | `grant_id` | committed operations still finish. Your API key can revoke a grant with no signature: [grants.md](grants.md#revoke) |
| `execution_cancel` | `correlation_id` of a `QUEUED` op | only while the timelock holds it |
| `payout_authorize` | `key: "evm" \| "solana"`, `payout: { origin_asset, amount_in, destination_asset, min_amount_out, recipient, recipient_type }`, `valid_until` | one 1Click deposit the key may pay through `sign-intent` outside `sign_intent.receivers` / `withdrawals`: [sign-intent.md](sign-intent.md#paying-one-deposit-the-owner-signed). `amount_in` is a ceiling, `min_amount_out` a floor, `valid_until` 5 minutes to 24 hours ahead; `preview.payout` shows the terms with the chain the API read |
| `approval_vote` | `approval_id`, `verdict: "approve" \| "reject"` | any owner; EVM/passkey votes go through their `0s` wallet |

## Generate response

```ts
{
  correlation_id: "intent_…",
  type, agent_id,
  status: "PENDING_SIGNATURE",
  expires_at: string,                         // submit before this (minutes, not hours)
  signer: OwnerWallet,                        // the wallet that must sign
  intent: { standard: "nep413" | "nep366" | "eip712" | "webauthn", payload },
  preview: {
    summary: string,
    revision?, previous_revision?, policy_hash?, policy?,   // policy changes
    deletion?: { near_account_id, beneficiary, public[], confidential[], open_limit_orders, assets_lost, retirement },
    approval?: { approval_id, request_type, request_hash, verdict },
    grant?: { grant_id, owner_message },        // agent_create with `grant`: the grant the signature also authorizes
    payout?: { origin_asset, amount_in, destination_asset, min_amount_out, recipient, recipient_type, key, chain, valid_until },   // payout_authorize
    wallet_request?: object,                    // passkey, wallet-request steps: what the WebAuthn challenge hashes
    offchain_message?: { chain_id, signer_id, timestamp, payload, path? }   // passkey, consent steps: what the challenge hashes
  }
}
```

**Passkey owners sign a hash.** Every `webauthn` intent carries exactly one preimage in its preview
(NEAR and EVM intents carry none): `wallet_request` (NEP-616) for steps the wallet contract runs,
`offchain_message` (NEP-641) for consent steps: `grant_issue`, `grant_revoke`, a local
`policy_update`, `execution_cancel`, `agent_archive` / `agent_restore`, `approval_vote` and
`payout_authorize`. On your backend, before the owner signs, recompute the challenge from it and compare
it to `intent.payload.challenge`, then check the content against what you asked for
([signing.md](signing.md#webauthn--passkey-owners); TypeScript: `verifyPasskeyChallenge(generated)`).
`offchain_message.payload` is the same text a NEAR owner signs as the `nep413` `message`.

The SDK and the reference clients also return the idempotency key they used. If
`generate-intent` is interrupted (timeout, crash), retry with the **same** `Idempotency-Key`;
`intent_generation_recovery_required` means keep that key and retry it rather than starting over.

## `agent_create` with a first grant

`grant` takes the terms of `grant_issue` ([grants.md](grants.md#the-first-grant-with-the-agent)).
The grant is hashed into the policy commitment the owner signs: the revision 1 `agent_policy_hash`
is `sha256(canonical JSON of { domain: "near-intents-agent-api.agent-create.v1", policy_commitment,
grant })`, and the plain `policy_commitment` when there is no grant. NEAR and EVM wallets show only
that hash, and a passkey signs a WebAuthn challenge over it, which `preview.wallet_request` lets
you recompute. Check, on your backend before the wallet signs, that the hash commits to exactly the
previewed policy and the grant you asked for (TypeScript: `verifyAgentCreateIntent`).

On `SUCCESS`, `details.grant_install` is `"installed"`, `"refused:grant_credential_reused"` or
`"refused:grant_expired"`. The agent exists in all three; hand the token to the agent only on
`"installed"`. A token already bound is refused by `generate-intent` itself, and `expires_at` must
outlast the 10-minute signing window.

## Submit

```json
{
  "type": "policy_update",
  "correlation_id": "intent_…",
  "signed_data": { "standard": "nep413", "payload": { "…": "exactly as generated" },
                   "public_key": "ed25519:…", "signature": "<base64>" }
}
```

| standard | `signed_data` = intent + |
|---|---|
| `nep413` | `public_key: "ed25519:…"`, `signature` (base64, base58, `ed25519:<b58>` or hex) |
| `nep366` | `signed_delegate` (base64 borsh `SignedDelegate`) |
| `eip712` | `signature: "0x…"` (65 bytes, 130 hex) |
| `webauthn` | `credential: AuthenticationResponseJSON` (from `@simplewebauthn/browser`) |

Submitting the same signature twice is safe and returns the same operation. The response is
usually `PROCESSING`; on-chain intents (`nep366`, sponsored EVM/passkey calls) settle in seconds
to a minute; off-chain ones settle almost immediately.

## Which standard you get

| Owner | Change enforced by provider (on chain) | Change enforced by API (off chain) |
|---|---|---|
| NEAR | `nep366` delegate action (sponsor pays gas): `agent_create`, provider policy edits, freeze/unfreeze | `nep413`: destinations/budget/timelock/schedule/`limit_order.max_deviation_bps`-only edits, grants, cancel, archive, votes, payout authorizations, … |
| EVM | `eip712` | `eip712` |
| Passkey | `webauthn` | `webauthn` |

Always switch on `intent.standard`; never assume from the type.

## Backend-for-frontend (BFF) split

The browser never sees the API key or any grant token. A typical pair of routes:

```text
POST /app/agents/:id/intents        (your auth: user owns agent :id)
  → api.generate_intent({...})       → returns { correlation_id, intent, preview, expires_at } to the browser

POST /app/intents/:cid/submit        (your auth: user owns the intent you stored for :cid)
  body: { signed_data }               ← wallet output from the browser
  → api.submit_intent(type, cid, signed_data)
  → api.get_status(cid, wait_ms=30000) → return status
```

Rules:
- Persist `{correlation_id, type, agent_id, user_id, created_at}` when you generate, and on
  submit check that the user owns that correlation id. Never let a client pick `type` or
  `correlation_id` you did not issue to them.
- Validate the user owns `agent_id` (your `external_user_id` mapping) before generating.
- Let the browser only *forward* the wallet output; don't let it supply a different `payload`
  (the API refuses mismatches with `intent_payload_mismatch` anyway).
- Show `preview.summary`, the full `preview.policy`, and for deletes `preview.deletion.assets_lost`.
- For a `webauthn` intent, verify the challenge against its preimage before the browser sees it, and
  refuse an intent that carries none.

Browser signing:

```ts
switch (intent.standard) {
  case "eip712":
    return { ...intent, signature: await walletClient.signTypedData({ account, ...intent.payload }) };
  case "nep366": {
    const r = await wallet.signDelegateActions({ delegateActions: [intent.payload] });
    return { ...intent, signed_delegate: r.signedDelegateActions[0] };
  }
  case "nep413": {
    const nonce = Uint8Array.from(atob(intent.payload.nonce), (c) => c.charCodeAt(0));
    const r = await wallet.signMessage({ ...intent.payload, nonce });
    return { ...intent, public_key: r.publicKey, signature: r.signature };
  }
  case "webauthn":
    return { ...intent, credential: await startAuthentication({ optionsJSON: intent.payload }) };
}
```

Adapt to your wallet library's result shape (near-connect / wallet-selector return `publicKey` and
`signature`; the server normalizes base58/base64/hex encodings).

## Errors you should handle here

| Code | Meaning / action |
|---|---|
| `intent_expired` (409) | signing window closed → generate again |
| `intent_payload_mismatch`, `intent_standard_mismatch` | the payload or standard was altered → sign the original |
| `signature_invalid` | malformed wallet output → submit exactly what the wallet returned |
| `signer_mismatch` | wrong key/account signed → have the `signer` wallet sign |
| `owner_key_not_full_access` (403) | NEAR key is a function-call key → use a FullAccess key |
| `owner_nonce_invalid` | intent already used/expired → generate again |
| `owner_command_stale` (409) | the intent was signed for an earlier owner state, or is the signature of a deletion that was later restored → generate again and sign again |
| `limit_orders_open` (409) | `agent_delete`: a limit order is not payout-final yet → cancel or let it settle, then delete again |
| `deletion_assets_remaining` (409) | `agent_delete`: the wallet still holds a public or confidential balance → withdraw it (a public one with `confidential: false`), then delete again |
| `grant_credential_reused` (409), `grant_expiry_invalid` (400 here, 409 for `grant_issue`) | `generate-intent` for `agent_create` with `grant` (or `grant_issue`): the token is already bound to a grant, or `expires_at` is past, inside the signing window or beyond 365 days → a new token / a later expiry; no agent was created |
| `policy_revision_conflict` (409) | re-read `GET …/policy`, regenerate with its `revision`, re-sign |
| `policy_too_large` (400) | the policy is too big for the custody contract to log in one write (encrypted form over 15,360 bytes); refused at generate-intent, before anyone signs, and also by submit-intent → allow fewer assets, per-asset limits or contracts, generate again |
| `provider_unavailable` / `provider_policy_unavailable` (503) | an on-chain change reads the wallet's freeze state from the chain: unreachable chain is `provider_unavailable` (retryable, same request after `Retry-After`), a missing policy record `provider_policy_unavailable`; nothing sent |
| `policy_reconciliation_required` | another policy intent is still open → wait for it to settle/expire |
| `policy_change_throttled` / `agent_unfreeze_throttled` (429) | wait until `meta.available_at` (also on `GET /v1/agents/{id}` cooldowns) |
| `wallet_already_frozen` / `wallet_already_unfrozen` | already in that state |
| `agent_creation_quota_exceeded` (429) | 100 agents/24h → wait for `available_at` or ask for a higher quota |
| `sponsor_busy` (503) | `generate-intent` or `submit-intent`: the sponsor's keys are busy, or an earlier write of the same owner key is still in flight; nothing was sent → with `meta.retryable`, send the same request (same key) after `Retry-After` |
| `sponsor_balance_insufficient` (503) | retry the same submit later |
| `owner_verification_unavailable` (502) | the RPC operators did not answer or did not agree on the NEAR owner's key; not an invalid signature → send the same request (same key) shortly |
| `transaction_unconfirmed` (503) | broadcast but not confirmed → poll status, or submit the same signed intent again (never sends twice); never generate a new intent for it |
| `agent_not_bound` | the agent is still onboarding → wait for `agent_create` SUCCESS |

## Recipes

**Edit policy safely**

```text
view = GET /v1/agents/{id}/policy
assert view.status == "APPLIED"
new  = modify(copy(view.policy))                  # complete policy, never a diff
gen  = generate_intent({type:"policy_update", agent_id, policy:new, expected_revision:view.revision})
… owner signs, submit, wait SUCCESS …
poll GET …/policy until status APPLIED && provider_policy_synced
```

**Emergency stop**: `agent_freeze` (no cooldown, takes effect at once). To stop one actor only,
`grant_revoke`, or revoke it with your API key alone ([grants.md](grants.md#revoke)), which needs no
signature and so also works when the owner is not around. To see what is still live: `GET …/containment`. A freeze refuses new requests but
cannot recall a signature already returned: a `sign-intent` payload signed earlier stays
publishable until its deadline (up to 96 h for a 1Click deposit payload). Tell the owner so.

**Enable `sign_intent` / `sign_transfer` / `sign_in`** (a `policy_update` that adds them to
`native`): show the owner what each lets a grant do ([sign-intent.md](sign-intent.md#what-to-tell-the-owner)):
`sign_intent` pays listed `receivers` (and 1Click deposits, with `one_click_deposits`/`withdrawals`)
from the EVM or Solana key's own Intents account in any amount, outside `destinations`, limits,
budget, timelock and approval; `sign_transfer` signs one Solana transfer to an allowed destination
without an amount limit; `sign_in` signs a login for a listed domain only.

**Cancel a timelocked payment**: list `GET …/executions/scheduled`, then `execution_cancel`
with that `correlation_id` before `execute_after`.

**Approve a payment** (policy `owner_approval: true`, any owner type): list `GET …/approvals`, show
`request_data`, then `approval_vote` with `verdict`. Each approval's `votes` lists the owner votes
submitted through this API (`{ voter, verdict, voted_at }`, oldest first, `voter` = the owner's principal id,
`[]` when none); a vote cast directly at the provider is not listed. At most 100 requests per agent wait at once:
the next is refused with `approval_backlog_full` (409, nothing sent) until the owner votes on one. The execution moves out of
`PENDING_APPROVAL`; keep polling its `op_…` id. `reject` ends it and releases its budget
charge (`FAILED`, `details.never_executed`). A `reject` cast within a block or two of a policy
change can be stored by the provider without cancelling the request. The API resends the owner's
signed reject for a few seconds. If the vote's status then has `details.repeat_required: true`, the
request is still pending: generate the `approval_vote` intent again and have the owner sign another
`reject` for the same approval. `repeat_required` is `null` while the vote is not applied yet and
`false` when nothing more is needed. `details.confirmation: "cancellation_observed"` means a fresh
reject lost its reply and the API observed the cancellation from the earlier receipt, so the reject
took effect. Approves are unaffected. `approve` re-runs the admission checks first: a
revoked or expired grant, a frozen or changed policy, or a destination no longer allowed refuses
the vote (`authorization_stale`, `wallet_frozen`, `policy_destination_denied`), nothing is
sent, and the request stays held for a `reject` while the refusal can still clear (a frozen
account). A held limit order's price is judged against the market again when the vote releases it. A refusal that can never clear ends it: once the policy, owner or lifecycle moved on or
the grant was revoked or expired, it fails `authorization_stale`; once the approval's
`expires_at` passes, it fails `expired`. Either way it settles `FAILED` with
`details.never_executed`, the charge is released, it leaves `GET …/approvals`, and a later
`approve` is refused (`authorization_stale` or `approval_closed`): don't retry the vote, tell the
user and start a new request if they still want it. `approval_not_recorded` means the API has not
yet recorded the request under its approval (its dispatch reply was lost): send the same vote
again a few minutes later.

**Delete**: withdraw balances first, `agent_delete`, show `preview.deletion`; refuse to proceed
in your UI when `assets_lost` is true unless the owner explicitly accepts.
