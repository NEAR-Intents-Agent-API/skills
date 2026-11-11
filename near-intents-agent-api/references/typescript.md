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

## Grants

```ts
const { token, commitment } = createGrantCredential();   // store token encrypted first
const g = await api.generateIntent({
  type: "grant_issue", agent_id, label: "assistant-session-42",
  credential: commitment,
  expires_at: new Date(Date.now() + 7 * 86_400_000).toISOString(),
});
// owner signs, submit, wait → status.details.grant.grant_id

const agent = api.forGrant(token);                         // one client per grant
```

## Executions

```ts
const quote = await agent.swap(agentId, {
  origin_asset: "nep141:wrap.near", destination_asset: usdc, amount: "1000000000000000000000000", dry: true,
});                                                          // QuoteResponse, no key: { dry, type, quote, spend, fee, warnings }
// Never refused by the account's rules: quote.warnings lists each one the live swap
// would fail under its code (spend_budget_exceeded, policy_schedule_denied + available_at,
// insufficient_balance + available_amount, …). Check it before the real swap.
// An unlisted asset or an invalid recipient throws unsupported_token or invalid_address (400), before any provider call.
// A token withdrawal to a NEAR account with no storage on that token is invalid_address (409), a preview too.
// A frozen, archived or deleted account gets no quote: it throws wallet_frozen, agent_archived or agent_deleted.
// withdraw(…, { dry: true }) is the same shape but needs the grant and a key; a preview that reaches the
// provider counts toward the wallet's 1,000 custody operations per rolling 24 h (swap quotes do not).

const idempotencyKey = createIdempotencyKey();               // persist with the body first
const op = await agent.swap(agentId, { origin_asset: "nep141:wrap.near", destination_asset: usdc, amount }, { idempotencyKey });
// op: StatusResponseOf<"swap"> & { idempotencyKey }

const deposit = await api.deposit(agentId, { origin_asset: usdcOnBase }, { idempotencyKey: createIdempotencyKey() });
// deposit.details: deposit_address, memo?, min_amount, expires_at, refund_to, deposit_seen_at? (SDK 0.7.3)
```

Also: `withdraw`, `transfer`, `shield`, `recover(agentId, { correlation_id, request }, { idempotencyKey })`.
`unshield` is deprecated: the API refuses every request with `public_balance_exit_only`.

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
`getOperationProof(id, cid)`, `getReceipt(id, cid)`, `listReceipts(id, { type, limit, cursor })`.

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
`nativeEvmSignIntent` / `nativeSolanaSignIntent(agentId, { payload, one_click_quotes? })` → `IntentSignature`
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

## Receipts

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
  // confidential omitted = a confidential payment (the default); `confidential: false` accepts one
  // paid from the legacy public balance
});
```

`verifyReceipt(receipt, { origin, trusted, readTile })` checks any receipt type and returns its typed
`terms`. `ReceiptNotCoveredError` and `LogUnavailableError` are retryable; `NoteError` and
`ProofError` mean reject. A withdrawal receipt proves the debit, not what arrived. See
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
    accountId: signer.account_id, publicKey: signer.public_key, finalityQuery: { finality: "final" } });
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
