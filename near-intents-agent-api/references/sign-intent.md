# Signing NEAR Intents payloads and logins with the agent's own keys

The agent's EVM and Solana keys each own a NEAR Intents account of their own (`signer_id`: the
lowercase `0x` EVM address, or the lowercase hex of the Solana public key). These routes let a
grant holder sign for it. They need a grant, are off until the owner's `native` policy switches
them on ([policy.md](policy.md#native-accounts-native)), are refused while the account is frozen,
and show up in history as type `native`. Advanced: for ordinary swaps, transfers and withdrawals
use the money routes.

| Route (`POST /v1/agents/{agent_id}/native/…`) | Switch | SDK | Body |
|---|---|---|---|
| `evm/sign-intent` | `evm.sign_intent` | `nativeEvmSignIntent` | `{ payload }` |
| `solana/sign-intent` | `solana.sign_intent` | `nativeSolanaSignIntent` | `{ payload }` |
| `solana/sign-in` | `solana.sign_in` lists the domain | `nativeSolanaSignIn` | `{ domain, nonce, issued_at, expiration_time, statement?, uri?, version?, chain_id? }` |
| `solana/sign-transfer` | `solana.sign_transfer` | `nativeSolanaSignTransfer` | `{ unsigned_tx, recipient }` |

Each switch is separate. `sign_message` never opens `sign_intent`; `sign_transaction` is not
needed for `sign_transfer`.

## sign-intent: build the payload

`payload` is a **string**: the exact JSON text of one NEAR Intents payload. The API signs the bytes
it was given and refuses text that is not the canonical form of what it parses to, so build the
object, `JSON.stringify` it once, and send that string unchanged (no pretty printing, no repeated
keys, no second serialization).

```ts
const payload = JSON.stringify({
  signer_id: wallet.evm_account_id.toLowerCase(),     // Solana: lowercase hex of the 32-byte public key
  verifying_contract: "intents.near",                 // or "intents.far" (confidential)
  deadline: new Date(Date.now() + 120_000).toISOString(),   // RFC 3339 UTC ending in Z
  nonce: Buffer.from(crypto.getRandomValues(new Uint8Array(32))).toString("base64"),
  intents: [{ intent: "transfer", receiver_id: "treasury.near", tokens: { [assetId]: "1000000" } }],
});
const { signed_data, signature, account_id } = await agent.nativeEvmSignIntent(agentId, { payload });
```

- `intents` holds only `transfer` intents (`receiver_id`, `tokens` as `{ token_id: atomic amount }`,
  optional `memo`, `msg`, `min_gas` ≤ u64), or none. A transfer to the signer itself, or to
  `intents.near`/`intents.far`, is refused.
- `signer_id` must be that key's own Intents account, lowercase (a checksummed EVM spelling is
  refused). Read the address from `GET /v1/agents/{id}/wallet` (`evm_account_id`, `solana_account_id`).
- `deadline`: at most **300 seconds** ahead and still in the future. Whoever holds a signed payload
  can publish it until then, so sign one that is due soon. The one exception is a payload that pays
  only 1Click deposits (below): up to **96 hours**, because 1Click writes its own quote deadline.
- For a 1Click deposit, take the payload from 1Click's `generate-intent` for the quote rather than
  hand-building it, then pass its text through untouched.

The answer is the usual `NativeSignature` plus `signed_data`, ready to publish **as is** to
`intents.near` (`execute_intents`), to 1Click, or to a solver relay: `{ standard: "erc191", payload,
signature: "secp256k1:…" }` for the EVM key, `{ standard: "raw_ed25519", payload, public_key,
signature }` for Solana. The API verifies the provider's signature against the agent's own key
before answering; a signature that fails is discarded (`502 provider_signature_invalid`, no value
returned, safe to ask again). The signature is returned only to this call.

## Who may be paid

These rules are the key's own. `destinations`, `actions`, `assets`, `limits`, `budget`,
`timelock_ms`, `schedule` and `owner_approval` **do not apply** to sign-intent: no amount rule
exists, and a listed receiver is paid whatever the payload says, from that key's Intents balance.

1. **`receivers`**: every `receiver_id` must be in `sign_intent.receivers`. Otherwise
   `403 policy_destination_denied` before the provider is called.
2. **1Click deposits** (`one_click_deposits: true`, or a non-empty `withdrawals`): a receiver that is
   not listed may be a 1Click deposit address, at most 2 per payload. The API reads 1Click's status
   for that address (any partner's quote) and signs only when all of this holds:
   - the quote is `PENDING_DEPOSIT`;
   - it pays out to the payload's `signer_id` or a listed receiver (inside Intents, on the contract's
     account type: `INTENTS`, `CONFIDENTIAL_INTENTS` for `intents.far`), or, with `withdrawals`, to a
     listed `{ chain, address }` on another chain. The chain is the one 1Click's token listing gives
     the quote's destination asset, never the quote request's; the address matches in canonical
     spelling; a listing that cannot be read is `503 provider_unavailable`;
   - it refunds to the `signer_id`, deposits into the contract's account type, and has no rebates,
     custom recipient message or virtual-chain recipients;
   - its app fees are only 1Click's own fee (to 1Click's fee account), and slippage tolerance plus
     that fee is at most `max_slippage_bps` (0 to 300, default 100); any other fee is refused;
   - the payload pays it with one plain transfer (no `msg`, no `min_gas`) of **exactly** the quote's
     `amountIn` of its origin asset (on `intents.far`, the token `imt:51e8f94d77b5e90dc9852ca6113771e11e8382ce69473a75e313e41665de7cbe:<originAsset>`, as 1Click's `generate-intent` writes it),
     due no later than the quote's deadline.

   `withdrawals` works without `one_click_deposits` and is how an agent withdraws or funds a venue
   (Hyperliquid, Polygon, Solana, Base, …) from this key. The loss bound is the quote's slippage and
   1Click's fee, not an amount rule.

## Refusals

| Code | Status | Meaning and move |
|---|---|---|
| `native_method_denied` | 403 | `sign_intent` is off for this key → owner `policy_update` |
| `policy_destination_denied` | 403 | a receiver is not allowed, or the quote check failed. **Read `detail`**: an unlisted receiver or withdrawal address → owner adds it; a quote that is not `PENDING_DEPOSIT`, a wrong amount, a refund elsewhere, an extra fee or too much slippage → get a fresh quote and rebuild, nothing for the owner to change (unless the slippage cap is too low for the route) |
| `intents_type_forbidden` | 403 | an intent other than `transfer` (`source.pointer`, e.g. `/payload/intents/0/intent`, names it) → remove it |
| `intents_contract_forbidden` | 403 | `verifying_contract` is not `intents.near`/`intents.far`, or a receiver is one of them |
| `intents_signer_mismatch` | 403 | `signer_id` is not this key's Intents account → use the lowercase address from the wallet |
| `intents_deadline_invalid` | 400 | not RFC 3339 UTC with `Z`, already past, or beyond 300 s (96 h for deposit-only payloads) → new deadline |
| `intents_payload_invalid` | 400 | not canonical JSON, unknown or repeated field, bad nonce or amount (`source.pointer`) → rebuild and stringify once |
| `provider_unavailable`, `provider_rate_limited` | 503, 429 | 1Click could not be read for a deposit address → retry later, same request ([errors.md](errors.md)) |
| `provider_signature_invalid` | 502 | the returned signature did not verify and was discarded → try again; if it repeats, report the `request_id` |
| `wallet_frozen` | 409 | account frozen → owner `agent_unfreeze` |
| `intents_payload_use_sign_intent` | 403 | you sent an Intents payload to `evm/sign-message` or `solana/sign-message` → use `sign-intent` |

## What to tell the owner

- Enabling `sign_intent` lets a grant holder move that key's Intents balance to the listed
  `receivers`, in any amount, with no limit, budget, timelock, schedule or approval. List only
  accounts the owner controls or trusts fully.
- `one_click_deposits` and `withdrawals` let it start swaps and withdrawals through 1Click that pay
  the key, a listed receiver or a listed address. In `withdrawals`, list only the owner's own
  addresses. Slippage plus 1Click's fee is the only bound.
- **Freeze caveat, say it plainly:** a freeze refuses new payloads but cannot recall one that was
  already signed. A payload signed before the freeze stays publishable by anyone who holds it until
  its own deadline: up to 5 minutes, or up to 96 hours for a deposit payload. Where you describe
  freeze as an emergency stop, say it does not cover signatures already returned.
- `solana.sign_message` signs any bytes (32-byte digests too), which NEAR Intents accepts under
  other signature standards, so it amounts to full control of that key's Intents account. Offer
  `sign_in` for logins, `sign_transfer` for payments and `sign_intent` for Intents instead.

## solana/sign-in

Sign-In-With-Solana with the agent's Solana key, so a dApp can log in as `wallet.solana_account_id`.
The API writes the message itself, so the signature can only be that login:

- `domain` must be listed in `native.solana.sign_in` (lowercase host, with its port if any).
- `nonce` 8 to 64 letters and digits; `issued_at` within 300 s of the API's clock; `expiration_time`
  still ahead and at most 10 minutes after `issued_at`; `statement` one line of printable ASCII;
  `uri` an `https` URI on `domain`; `version` `1`; `chain_id` `mainnet` or `solana:mainnet`.
  `not_before`, `request_id` and `resources` are refused (`validation_failed`, `source.pointer`).
- The answer adds `signed_message`, the exact text signed. Give the relying party its UTF-8 bytes
  with `signature` and `account_id`. Request fresh timestamps per attempt: a login that expires while
  being signed is discarded.

## solana/sign-transfer

One SOL or SPL transfer without the full-control `sign_transaction`. Build the message yourself
(`versionedTx.message.serialize()`, base64) and pass the `recipient` wallet:

- legacy or v0 message with **no address lookup tables**; the agent is the only signer and fee payer;
  no account appears twice;
- exactly one System Program `Transfer` to `recipient`, or one SPL Token or Token-2022
  `TransferChecked` into `recipient`'s associated token account; optionally one `CreateIdempotent`
  of that account paid by the agent and a compute unit limit and price with a priority fee ≤
  1,000,000 lamports. A memo or a durable-nonce `AdvanceNonceAccount` is refused;
- `recipient` must be allowed by `destinations` as a `withdraw` on chain `sol` with no memo
  (`policy_destination_denied`); anything else in the message is
  `403 solana_transaction_forbidden` with the reason in `detail`, and nothing is signed.

This stops payments to arbitrary recipients; it does **not** limit the amount (`assets`, `limits`
and `budget` are not applied yet). The caller broadcasts the signed transaction while its blockhash
is recent. The history record keeps `details.target` (recipient), `details.amount` and the transfer
as read in `details.message_json`.
