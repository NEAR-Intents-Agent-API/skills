# TypeScript / JavaScript

```bash
npm i @near-intents-agent-api/sdk          # or pnpm add / yarn add / bun add
```

Node 24+ (or any runtime with `fetch` and Web Crypto). The SDK is a thin, typed client generated
from the server's endpoint registry: one method per endpoint, JSON:API errors, no retries, no
wallet signers, **server-side only** (it needs the API key).

## Client

```ts
import { AgentApi, AgentApiError, AgentApiRequestError, createGrantCredential, createIdempotencyKey } from "@near-intents-agent-api/sdk";

const api = new AgentApi({
  apiKey: process.env.NEAR_INTENTS_AGENT_API_KEY!,           // from the partner portal (opaque string)
  baseUrl: process.env.NEAR_INTENTS_AGENT_API_URL,           // optional; defaults to https://api.agentsonintents.com
});

await api.getNetwork();
await api.whoami();
await api.getPartnerQuota();
const tokens = await api.getTokens();          // TokenView[]
```

Options: `baseUrl`, `timeoutMs` (default 150 s for settlement writes: live swap, limit order, withdraw, transfer, shield; 65 s otherwise),
`maxResponseBytes`, `fetch`, `grantToken`.

## Owner intent loop

```ts
const generated = await api.generateIntent({
  type: "agent_create",
  name: "Trading assistant",
  external_user_id: user.id,
  owner: { type: "near", account_id: "alice.near", public_key: "ed25519:…" },
  policy,
});
// generated: { correlation_id, agent_id, intent: { standard, payload }, preview, expires_at, signer, idempotencyKey }

// → send { correlation_id, intent, preview } to the browser; it returns signed_data
const signed_data = await browserSigns(generated.intent);

await api.submitIntent({ type: generated.type, correlation_id: generated.correlation_id, signed_data });
let status = await api.getStatus(generated.correlation_id, { waitMs: 30_000 });
```

`generateIntent` returns `idempotencyKey`; to retry an interrupted generation pass the same one:
`api.generateIntent(req, { idempotencyKey })`.

Typed narrowing: `GenerateIntentResponseOf<"policy_update">`, `StatusResponseOf<"swap">`, `StatusResponseOf<"limit_order">`.

SDK 0.9.7. A `webauthn` intent signs only a hash, so its preview carries what the hash covers:
`preview.wallet_request` (NEP-616 wallet-request steps) or `preview.offchain_message` (NEP-641 consent
steps: `grant_issue`, `grant_revoke`, a local `policy_update`, `execution_cancel`, `agent_archive`,
`agent_restore`, `approval_vote`, `payout_authorize`); NEAR and EVM intents carry neither. Check it on
your backend before the owner signs:

```ts
import { verifyPasskeyChallenge, offchainMessageChallenge, walletRequestChallenge } from "@near-intents-agent-api/sdk";

if (generated.intent.standard === "webauthn" && !verifyPasskeyChallenge(generated))
  throw new Error("the passkey challenge does not hash the previewed message");   // false also when no preimage came
// by hand: offchainMessageChallenge(preview.offchain_message) or walletRequestChallenge(preview.wallet_request)
// must equal generated.intent.payload.challenge; then compare the preimage's content (its `payload`) with your request
```

`GenerateIntentResponse` is a union by preview: narrow before reading a preimage,
`"wallet_request" in generated.preview` or `"offchain_message" in generated.preview`.

## The first grant, in the same signature as the agent

SDK 0.9.5. `agent_create` takes an optional `grant` with the terms of `grant_issue` (`label`,
`credential`, `expires_at`). The owner's one signature then creates the agent and authorizes that
grant, because the grant is hashed into the policy commitment the wallet signs. A NEAR or EVM wallet
shows the owner only that hash, and a passkey signs a challenge over it (`preview.wallet_request`),
so check the generated intent on your backend before any wallet sees it:

```ts
import { createGrantCredential, verifyAgentCreateIntent } from "@near-intents-agent-api/sdk";

const { token, commitment } = createGrantCredential();       // store token encrypted first
const grant = {
  label: "Trading assistant",
  credential: commitment,
  expires_at: new Date(Date.now() + 86_400_000).toISOString(),   // must outlast the 10-minute signing window
};
const generated = await api.generateIntent({ type: "agent_create", name, owner, policy, grant });
// Throws AgentCreateVerificationError unless the signed hash commits to exactly
// generated.preview.policy and this grant (NEAR, EVM and passkey owners).
verifyAgentCreateIntent(generated, { grant });
// The owner signs generated.intent; submit and wait as in "Owner intent loop".
const status = await api.getStatus(generated.correlation_id, { waitMs: 30_000 });
status.details.grant_install;      // "installed" | "refused:grant_credential_reused" | "refused:grant_expired" | null
const assistant = api.forGrant(token);                         // use it only when "installed"
```

- Without `grant`, `verifyAgentCreateIntent(generated, { tenantId })` still checks the policy
  commitment; the preview names no tenant then, so pass your partner tenant id.
- `preview.grant` echoes the grant (`grant_id`, `owner_message`). Passing a grant the preview does
  not carry, or the preview carrying one you did not ask for, throws as well.
- A token already bound to a grant is refused by `generateIntent` (`grant_credential_reused`, 409),
  before any agent exists; `expires_at` that is past, inside the signing window or beyond 365 days is
  `grant_expiry_invalid`.
- The agent is never refused because of its grant. If another onboarding bound the token first, or
  the grant expired before confirmation, the agent exists without it and `grant_install` says why:
  issue a grant with `grant_issue`. The token is useless until `grant_install` is `"installed"`.
- `listGrants` shows the grant with `issued_via: { type: "agent_create", transaction_hash,
  policy_revision: 1, policy_commitment }` (`{ type: "grant_issue" }` for a separately signed one).
  The transaction's `agent_policy_hash` is `sha256(canonical JSON of { domain:
  "near-intents-agent-api.agent-create.v1", policy_commitment, grant: owner_message })`; with no
  grant it is the plain `policy_commitment`. `agentCreateCommitment`, `walletRequestChallenge` and
  `serializeWalletRequestMessage` are exported for custom checks.

## Grants

```ts
const { token, commitment } = createGrantCredential();   // store token encrypted first
const g = await api.generateIntent({
  type: "grant_issue", agent_id, label: "assistant-session-42",
  credential: commitment,
  expires_at: new Date(Date.now() + 7 * 86_400_000).toISOString(),
});
// owner signs, submit, wait → status.details.grant.grant_id

const agent = api.forGrant(token);                         // one client per grant; takes ngt_ or ngs_
```

Revoking needs no owner signature (SDK 0.9.7): the API key alone does it, and the grant reads
`revoked_reason: "partner_revoked"` ([grants.md](grants.md#revoke)).

```ts
const { grant, committed_correlation_ids, committed_cancellations, committed_truncated } =
  await api.revokeGrant(agentId, grantId);      // the grant is returned unchanged if already revoked or expired
const { data: revoked } = await api.revokeAllGrants(agentId);   // every live grant; `data` is empty when none was live
```

Its next request fails `agent_grant_required`, and its sessions end with it. Committed work finishes:
follow `committed_correlation_ids` with `getStatus`. Issuing a grant and freezing stay owner intents.

## Workload identity

A federated grant has no token to store (which kind to pick, the owner's signature, platform recipes,
KMS signers: [workload-identity.md](workload-identity.md); this section needs SDK 0.9.2 or later). `grant_issue.credential` is a `GrantIssueCredential`: the token commitment,
`OidcGrantCredential` or `PublicKeyGrantCredential`. Grant views carry `credential_type`
(`GrantCredentialType`: `token`, `oidc`, `public_key`).

```ts
import {
  createWorkloadKey, localJwkSigner, oidcGrantAudience, DEFAULT_BASE_URL,
  type WorkloadSigner,
} from "@near-intents-agent-api/sdk";

// OIDC grant. Omit `audience` (only Microsoft Entra ID names it); the API sets it.
await api.generateIntent({
  type: "grant_issue", agent_id: agentId, label: "CI deploy (production)",
  credential: { type: "oidc", issuer: "https://token.actions.githubusercontent.com",
                claims: { repository_owner_id: "123", repository_id: "456", environment: "production" } },
  expires_at: new Date(Date.now() + 30 * 86_400_000).toISOString(),
});
// The workload requests its platform token for this audience, then:
const audience = oidcGrantAudience(DEFAULT_BASE_URL, agentId);
const bot = api.forGrantSession(agentId, {
  subjectToken: () => requestPlatformToken(audience),      // called per exchange: read it fresh
  // grantId, subjectTokenType, refreshBeforeSeconds (default 60)
});
await bot.transfer(agentId, { asset: "nep141:wrap.near", amount: "1", recipient: "bob.near" }, { idempotencyKey });

// Key-bound grant.
const { privateJwk, publicJwk } = createWorkloadKey("Ed25519");   // or "P-256"; owner signs { type: "public_key", jwk: publicJwk }
const signer: WorkloadSigner = localJwkSigner(privateJwk);        // or your KMS-backed { publicJwk, sign }
const viaSession = api.forGrantSession(agentId, { signer });      // signs the exchange assertions itself
const viaProof = api.forGrantProof({ grantId, signer });          // X-Grant-Proof per request; grantId from grant_issue / listGrants
```

- `api.exchangeGrantToken(agentId, { subject_token, subject_token_type?, grant_id? }, options?)` returns
  `GrantTokenExchangeResponse` (`access_token` `ngs_…`, `expires_in`, `expires_at`, `grant_id`, `label`).
  `forGrantSession` calls it lazily, caches the session and shares one exchange between concurrent
  calls. From `refreshBeforeSeconds` (default 60, at most half the session's life) before the end it
  renews in the background while the current token keeps serving until 5 s before the end; a failed
  exchange is not repeated for 5 s; a delegated call refused with `agent_grant_required` throws, is
  not repeated, and drops the session so the next call exchanges again; call it directly only if you manage sessions yourself, then
  `api.forGrant(access_token)`.
- `forGrantProof` signs every delegated attempt with a new proof; a retry keeps the same
  `idempotencyKey`. A client sends a token **or** a proof, never both. Keep one client per grant.
- `WorkloadSigner = { publicJwk, sign(signingInput: Uint8Array): Uint8Array | Promise<Uint8Array> }`;
  `sign` returns the raw 64-byte JWS signature (P-256: `r || s`, **not** a KMS's DER; the SDK throws
  otherwise). A signer runs once per proof or exchange: with a billed KMS prefer `forGrantSession`.
- Hand-rolled flows: `jwkThumbprintUri(publicJwk)`, `createKeyAssertion(signer, { baseUrl, agentId })`
  (single use, for `subject_token`), `createGrantProof(signer, { grantId, method, url, body?, idempotencyKey? })`
  (types `GrantProofRequest`, `GrantSessionOptions`, `SubjectTokenType`, `WorkloadKey`, `WorkloadPrivateJwk`).
- `baseUrl` must be the URL the API publishes as `servers[0]`: audiences and proofs are built from it.
- Errors: `subject_token_invalid`, `grant_proof_invalid`, `grant_ambiguous`, `grant_session_limit_reached`,
  `issuer_unavailable`, `grant_credential_invalid` ([errors.md](errors.md#workload-identity)).

## Executions

```ts
const quote = await agent.swap(agentId, {
  origin_asset: "nep141:wrap.near", destination_asset: usdc, amount: "1000000000000000000000000", dry: true,
});                                                          // QuoteResponse, no key: { dry, type, quote, spend, fee, warnings }
// Never refused by the account's rules: quote.warnings lists each one the live swap
// would fail under its code (spend_budget_exceeded, policy_schedule_denied + available_at,
// insufficient_balance + available_amount, …). Check it before the real swap.
// An unlisted asset or an invalid recipient throws unsupported_token or invalid_address (400), before any provider call.
// A token withdrawal to a NEAR account with no storage on that token throws recipient_storage_unregistered (409), a preview too.
// A frozen, archived or deleted account gets no quote: it throws wallet_frozen, agent_archived or agent_deleted.
// withdraw(…, { dry: true }) is the same shape but needs the grant and a key; a preview that reaches the
// provider counts toward the wallet's 1,000 custody operations per rolling 24 h (swap quotes do not).

const idempotencyKey = createIdempotencyKey();               // persist with the body first
const op = await agent.swap(agentId, { origin_asset: "nep141:wrap.near", destination_asset: usdc, amount }, { idempotencyKey });
// op: StatusResponseOf<"swap"> & { idempotencyKey }

// Exact output: receive amount_out exactly, spending at most max_amount_in (never with amount / min_amount_out).
const exactOut = { origin_asset: "nep141:wrap.near", destination_asset: usdc, amount_out: "3000000" };
const preview = await agent.swap(agentId, { ...exactOut, max_amount_in: mostYouWouldPay, dry: true }); // preview.quote.amount_in; preview.warnings
const ceiling = ((BigInt(preview.quote.amount_in ?? "0") * 101n) / 100n).toString();         // 1% room above the quote
await agent.swap(agentId, { ...exactOut, max_amount_in: ceiling }, { idempotencyKey: createIdempotencyKey() });
// settled: details.amount_in = the input spent; the rest of max_amount_in returns to the balance
// withdraw takes the same two shapes (amount + min_amount_out?, or amount_out + max_amount_in) and no memo.

const deposit = await api.deposit(agentId, { origin_asset: usdcOnBase }, { idempotencyKey: createIdempotencyKey() });
// deposit.details: deposit_address, memo?, min_amount, expires_at, refund_to, deposit_seen_at? (SDK 0.7.3)
```

Also: `withdraw`, `transfer`, `shield`, `recover(agentId, { correlation_id, request }, { idempotencyKey })`.
`unshield` is deprecated: the API refuses every request with `public_balance_exit_only`.

A `min_amount_out` above a fresh quote's guaranteed output throws `below_min_amount_out`, and a `max_amount_in` below its `amount_in` throws `above_max_amount_in` (both 409, nothing sent; change the figure, new key); a dry run whose quote already misses the figure warns with the same code. A withdrawal with a `memo` is `validation_failed`, and a public-balance withdrawal to chain `near` takes `amount` alone (SDK 0.9.6).

`confidential` defaults to `true`. `swap`, `placeLimitOrder`, `transfer` and `deposit` throw
`AgentApiError` with `code: "public_balance_exit_only"` (400, `layer: "request"`, not retryable) for
`confidential: false`; do not retry. Only `withdraw(…, { confidential: false })` still pays from the
legacy public balance, and `shield` moves public funds into the confidential balance. Pass
`reference` (transfer, withdraw) to tie a payment to an order and `note` (any execution) to say why;
the note is owner-only.

### Limit orders (SDK 0.8.1)

```ts
const order = { base_asset: "nep141:wrap.near", quote_asset: usdt, side: "sell" as const,
                quantity: "1000000000000000000000000", price: "5.2" };  // price: quote units per whole base
const preview = await agent.placeLimitOrder(agentId, { ...order, dry: true });   // no grant, no key, no provider call; QuoteResponse
const placed = await agent.placeLimitOrder(agentId, order, { idempotencyKey });  // SUCCESS = funded and resting, not filled
const orderId = placed.details?.order_id;

const { data } = await agent.listLimitOrders(agentId, { open: true, limit: 100, offset: 0 });  // { data }, not unwrapped; no grant
const live = await agent.getLimitOrder(agentId, orderId!);                       // fill_status, partial_fills, payouts, is_payout_status_final
await agent.cancelLimitOrder(agentId, orderId!);                                 // grant, no key
let round = await agent.cancelAllLimitOrders(agentId);                           // up to 50 a call
while (!round.complete) round = await agent.cancelAllLimitOrders(agentId);
```

Types: `LimitOrderRequest`, `LimitOrderView`, `LimitOrderPage`, `CancelLimitOrderResult`,
`CancelAllLimitOrdersResult`. Rules, polling and error handling: [limit-orders.md](limit-orders.md).

## Reads

`listAgents({ external_user_id, cursor })`, `getAgent`, `getWallet`, `getBalances(id, { source, asset })` (confidential unless `source: "public"`),
`getPolicy`, `getPolicyHistory`, `listGrants`, `listScheduledExecutions`, `getContainment`,
`listApprovals`, `getApproval`, `listLimitOrders`, `getLimitOrder`, `getAddress(id, "near")`, `listProviderRecords(id, kind)`,
`getHistory(id, { cursor, limit, include_unpaid })`, `getStatus(cid, { waitMs, refresh })`,
`lookupOperations(id, idempotencyKey, options?)` (SDK 0.9.3),
`getOperationProof(id, cid)`, `getReceipt(id, cid)`, `listReceipts(id, { type, limit, cursor })`.
`listApprovals` returns the array (at most 100) and drops the listing's `truncated` flag (SDK 0.9.1).

`lookupOperations` answers `{ data, retired }` for a request whose response was lost: `data` is the
retained operations the key admitted (newest first, as `getHistory` entries), `retired` the ids whose
detail was purged after retention (the key stays spent). Both empty is inconclusive; only a same-key,
same-body replay under the original grant settles it
([status-and-recovery.md](status-and-recovery.md#lost-the-response-look-the-key-up)). It needs
SDK 0.9.3 or later (`npm i @near-intents-agent-api/sdk@^0.9.7` installs the current one), which also types `ApprovalView.votes` and
the `recipient_storage_unregistered` code:

```ts
const { data, retired } = await api.lookupOperations(agentId, idempotencyKey);
if (data.length) await api.getStatus(data[0].correlation_id, { waitMs: 30_000 });   // newest first
```

## Native accounts

With a grant, and only for methods the policy's `native` block switches on
([policy.md](policy.md#native-accounts-native)):

```ts
// NEP-413 login for a recipient listed in native.near.sign_message; message and nonce come from the service
const proof = await agent.nativeNearSignMessage(agentId, { message, recipient: "app.example.com", nonce /* 32 bytes, base64 */ });
// { correlation_id, near_account_id, public_key, signature (base64), nonce (echo), recipient }

// EIP-712 under a domain listed in native.evm.sign_typed_data (and a primaryType the entry's
// primary_types allows, when it sets them; otherwise 403 native_method_denied)
const { signature } = await agent.nativeEvmSignTypedData(agentId, { chain: "arbitrum", typed_data });

// Native NEAR from near_account_id (native.near.transfer); idempotent like any execution
const sent = await agent.nativeNearTransfer(agentId, { to: "bob.near", amount: "1000000000000000000000000" }, { idempotencyKey });
```

Also `nativeNearCall`, `nativeEvmSignMessage`, `nativeEvmSignTransaction`,
`nativeSolanaSignMessage`, `nativeSolanaSignTransaction`. SDK 0.7.4 adds
`nativeEvmSignIntent` / `nativeSolanaSignIntent(agentId, { payload, one_click_quotes?, payout_authorization_id? })` → `IntentSignature`
(`signed_data` to publish as is), `nativeSolanaSignIn(agentId, SolanaSignInRequest)` →
`SolanaSignIn` (adds `signed_message`) and `nativeSolanaSignTransfer(agentId, { unsigned_tx, recipient })`
→ `NativeSignature`; policy types `SignIntentRules`, `EvmSignIntentRequest`,
`SolanaSignIntentRequest`, `SolanaSignInRequest`, `SolanaSignTransferRequest`; SDK 0.8.2 adds
`one_click_quotes` and the type `OneClickQuote` (1Click's quote answer, sent unchanged):

```ts
const payload = JSON.stringify({ signer_id, verifying_contract: "intents.near", deadline, nonce, intents }); // once, unchanged
const { signed_data } = await agent.nativeEvmSignIntent(agentId, { payload });   // publish signed_data as is
// a payload that pays a 1Click deposit address: also the live quote (dry: false), whole and unchanged
// await agent.nativeEvmSignIntent(agentId, { payload, one_click_quotes: [quote] });
```

SDK 0.9.7 adds `payout_authorization_id`: to pay one 1Click deposit that `receivers` and `withdrawals` do
not allow, the owner signs a `payout_authorize` intent (`PayoutAuthorizeRequest`; `preview.payout` is a
`PayoutPreview`) and the agent sends its `correlation_id`:

```ts
const authorization = await api.generateIntent({
  type: "payout_authorize", agent_id: agentId, key: "evm",
  payout: { origin_asset, amount_in /* ceiling */, destination_asset, min_amount_out /* floor */,
            recipient, recipient_type: "DESTINATION_CHAIN" },
  valid_until: new Date(Date.now() + 3_600_000).toISOString(),   // 5 minutes to 24 hours ahead
});
// the owner signs authorization.intent, you submit it; its status is PROCESSING until used
await agent.nativeEvmSignIntent(agentId, { payload, one_click_quotes: [quote],
  payout_authorization_id: authorization.correlation_id });
```

Rules, status and refusals (`payout_authorization_invalid`, `payout_authorization_expired`,
`payout_authorization_voided`, `payout_authorization_refused`): [sign-intent.md](sign-intent.md#paying-one-deposit-the-owner-signed).

Rules, 1Click deposit checks, refusals and the freeze caveat: [sign-intent.md](sign-intent.md).
Signing methods return the signature and
never broadcast, and only they return the signature: status and history show `signature: null`. `nativeNearTransfer` and `nativeNearCall` read `SUCCESS` only once NEAR finalizes the requested transaction (see [policy.md](policy.md#native-accounts-native)). For logins, the relying party verifies the NEP-413 signature over `message`,
`nonce` and `recipient`, checks the nonce is the one it issued, and that `public_key` is a
full-access key of `near_account_id`.

## Operation proofs

```ts
import {
  fetchWitnessCheckpoint, NoteError, ProofError, verifyOperationProof,
} from "@near-intents-agent-api/sdk";

// Pin the origin and the witness key yourself. Never read either from an API response.
const ORIGIN = "api.agentsonintents.com/log";
const WITNESS_KEY = "598165a18b588f504e159c0e2573d73983ae22bf15cf80d36902276d63a11101"; // production

const head = await fetchWitnessCheckpoint("https://api.agentsonintents.com", ORIGIN, WITNESS_KEY);
const proof = await api.getOperationProof(agentId, "op_…", { from_size: String(head.size) }); // executions only
const verified = verifyOperationProof(proof, ORIGIN, head);
// verified.proven: [{ index, fields, checkpoint, cosignedAt, evidence }]; verified.pending, verified.unlogged
```

Without trusting the API: `verifyOperationProof(proof, origin, trusted)` needs a trusted
checkpoint as well as the pinned origin, and refuses without one. `fetchWitnessCheckpoint` fetches
a fresh head through `/tee/witness` and checks the front-1 witness signature with the key you
pinned, even though the API carries the reply. Fetch the head right before requesting the proof
and send its size as `from_size`; the proof then carries a consistency path from that head. A
server behind it answers `409 transparency_checkpoint_behind`: keep your trusted checkpoint and retry later. A checkpoint you stored earlier
proves continuation from that file only, not that it is current. Each `PROVEN` event hashes to a leaf of a checkpoint signed by
the log key and cosigned by the notary, and names this execution. Its `evidence`, the result it
recorded with any settlement transaction hashes, must hash to the opening's `evidenceHash` and comes
back parsed (`null` when none was served). A proof that does not hold, an altered or rolled-back
checkpoint, or altered evidence throws `NoteError` or `ProofError`. `PENDING` events wait for the
next checkpoint. To tie the keys to attested code, check the notary's TDX birth quote
(`proof.notary.birth`) and compare its `report_data` with `verified.birthReportData`; the API
runs in a Google Cloud TDX confidential VM with keys from its own KMS. A deployment without a log
answers `501 transparency_log_disabled`. The two-argument `verifyOperationProof(proof, origin)` of
SDK 0.7.2 and earlier did not check for rollback; upgrade to 0.8.0.

## Receipts (SDK 0.9.0)

```ts
import { fetchWitnessCheckpoint, logTileReader, verifyPaymentReceipt } from "@near-intents-agent-api/sdk";

const paid = await agent.transfer(agentId, {
  asset: usdc, amount: "2500000", recipient: "shop.near",
  reference: "order-42",                          // shared with the payee through the receipt
  note: "Paying order 42, as the owner asked",    // owner-only; never in the receipt
}, { idempotencyKey });
// After getStatus reads SUCCESS:
const item = await api.getReceipt(agentId, paid.correlation_id);   // { status, details, receipt }; receipt is null until PROVEN
const page = await api.listReceipts(agentId, { type: "transfer", limit: 50 });  // keep paging while next_cursor is set

// The payee verifies what it was handed, against a head it fetched and pinned itself:
const payment = await verifyPaymentReceipt(item.receipt!, {
  origin: ORIGIN, trusted: head, readTile: logTileReader("https://log.agentsonintents.com/"),
  expect: { recipient: "shop.near", asset: usdc, amount: "2500000", reference: "order-42" },
  // an exact-output withdrawal: amount_out instead of amount (exactly one of the two), plus chain;
  // the result then has amountOut and maxAmountIn, and amount is absent (optional in SDK 0.9.6)
  // confidential omitted = a confidential payment (the default); `confidential: false` accepts one
  // paid from the legacy public balance
});
```

`verifyReceipt(receipt, { origin, trusted, readTile })` checks any receipt type and returns its typed
`terms`. `ReceiptNotCoveredError` and `LogUnavailableError` are retryable; `NoteError` and
`ProofError` mean reject. A withdrawal receipt proves the debit, not what arrived. An exact-output withdrawal's receipt (`terms` hold `amount_out` and `max_amount_in`) is issued only once 1Click reports delivering at least `amount_out`. See
[status-and-recovery.md](status-and-recovery.md#receipts).

## Errors

```ts
try {
  await agent.withdraw(agentId, req, { idempotencyKey });
} catch (error) {
  if (error instanceof AgentApiError) {
    switch (error.code) {
      case "policy_destination_denied": /* ask owner to allow the address; for sign-intent read error.detail first (unlisted receiver vs a missing or failed 1Click quote) */ break;
      case "spend_budget_exceeded":     /* show remaining budget (a dry quote or preview warns with this code); nothing was recorded */ break;
      case "insufficient_balance":      /* fund, or send at most error.availableAmount (atomic units) */ break;
      case "policy_price_deviation_exceeded": /* limit order: price closer to the market, or owner raises limit_order.max_deviation_bps */ break;
      case "policy_schedule_denied":    /* outside owner hours: same request at error.availableAt */ break;
      case "policy_change_throttled":   /* error.availableAt */ break;
      default:
        // error.layer: "request" | "auth" | "grant" | "policy" | "budget" | "balance" | "provider" | "api"
        // policy/budget → ask the owner; balance → fund or send less; error.retryable → same key after error.availableAt,
        // else after error.retryAfterSeconds (the Retry-After header in whole seconds, SDK 0.7.2);
        // request → fix the request
        throw error;
    }
  } else if (error instanceof AgentApiRequestError) {
    // transport failure: retry with error.idempotencyKey and the same body
  }
}
```

## Waiting helper

```ts
async function settle(api: AgentApi, cid: string, deadlineMs = 300_000) {
  const end = Date.now() + deadlineMs;
  for (;;) {
    const s = await api.getStatus(cid, { waitMs: 30_000 });
    if (!["PROCESSING", "QUEUED"].includes(s.status) || Date.now() > end) return s;
  }
}
```

## Signing on the server (scripts, tests, company-held owner keys)

NEAR (near-api-js v7) — from the examples' `support/sign-near-intent.ts`:

```ts
import { createHash } from "node:crypto";
import { actions, encodeDelegateAction, encodeSignedDelegate, JsonRpcProvider, KeyPair, Signature } from "near-api-js";

async function signNearIntent(generated, keyPair: KeyPair, rpcUrl: string) {
  const { intent, signer } = generated;
  if (intent.standard === "nep413")
    return { ...intent, public_key: signer.public_key,
             signature: await signNep413(keyPair, { ...intent.payload, nonce: Buffer.from(intent.payload.nonce, "base64") }) };
  const access = await new JsonRpcProvider({ url: rpcUrl }).viewAccessKey({
    accountId: signer.account_id, publicKey: signer.public_key, finalityQuery: { finality: "optimistic" } });
  const delegateAction = {
    senderId: signer.account_id, receiverId: intent.payload.receiverId, publicKey: keyPair.getPublicKey(),
    nonce: access.nonce + 1n, maxBlockHeight: BigInt(access.block_height) + 120n,
    actions: intent.payload.actions.map(({ params }) =>
      actions.functionCall(params.methodName, Buffer.from(JSON.stringify(params.args)), BigInt(params.gas), BigInt(params.deposit))),
  };
  const digest = createHash("sha256").update(encodeDelegateAction(delegateAction)).digest();
  const signature = new Signature({ keyType: 0, data: keyPair.sign(digest).signature });
  return { ...intent, signed_delegate: Buffer.from(encodeSignedDelegate({ delegateAction, signature })).toString("base64") };
}
```

(`encodeDelegateAction` already prepends the NEP-461 prefix. `signNep413` is in the examples'
`support/near-owner.ts`: sha256 of the NEP-413 borsh body, signed with the key pair, base64.)

EVM (viem):

```ts
import { privateKeyToAccount } from "viem/accounts";
const account = privateKeyToAccount(pk);
const owner = { type: "evm", address: account.address.toLowerCase(), chain_id: 1, public_key: `0x${account.publicKey.slice(4)}` };
const signed_data = { ...intent, signature: await account.signTypedData(intent.payload) };
```

Browser signing for all four standards: [owner-intents.md](owner-intents.md#backend-for-frontend-bff-split).

## Next.js / serverless notes

- Keep the SDK in server code only (route handlers, server actions); never import it into a
  client component with the key.
- Long-polls take up to 30 s and executions up to ~2 min: set the function timeout accordingly
  or move polling to a background job / queue.
