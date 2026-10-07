# NEAR Intents Agent API — Agent Skills

Skills that teach coding agents (Claude Code, Claude.ai, Codex, Cursor, …) how to build on the
[NEAR Intents Agent API](https://api.agentsonintents.com/llms.txt): give every AI agent its own
custody account on NEAR Intents, let the end user sign the rules once with their own wallet, and
let the agent swap, transfer, withdraw and receive funds inside those rules — from TypeScript,
Python, Rust, Go or any language that speaks HTTP.

## What the API does, in one minute

```
 your backend (partner)            end user (owner)                  AI agent
 ─────────────────────            ────────────────                  ────────
 naa_ API key                      wallet: NEAR / EVM / passkey      ngt_ grant token (held by your backend)
 prepares owner actions   ──────►  signs: create account, policy,
 stores grant tokens               grants, freeze, approve, delete
 calls executions         ◄─────────────────────────────────────────  swap / transfer / withdraw / deposit
                                                                       … only inside the owner-signed policy
```

- **Policy = what** may happen: actions, assets, per-asset limits, allowed destinations, USD
  budget, timelock, weekly schedule, owner approval, freeze.
- **Grant = who** may act: one revocable token per session, assistant or bot.
- Your backend never holds the owner's key; the browser never sees your API key; the model never
  sees either.

## Start in five minutes

1. **Get a key.** Create a partner API key in the [partner dashboard](https://partners.near-intents.org/) (**API keys**). The
   `naa_…` token is shown once; store it as `AGENT_API_KEY` in your backend's secrets.
2. **Check it.**
   ```bash
   AGENT_API_KEY=naa_... ./near-intents-agent-api/scripts/check_api.sh
   ```
3. **Install the skill** (below) and ask your coding agent, e.g.
   *"Add NEAR Intents agent wallets to our FastAPI backend: onboarding with the user's NEAR
   wallet, one grant per chat session, and a swap tool for the assistant."*

The hosted API (`https://api.agentsonintents.com`) is mainnet with real funds. Build with small
amounts and dedicated owner accounts.

## Install

### Claude Code (plugin marketplace)

```text
/plugin marketplace add NEAR-Intents-Agent-API/skills
/plugin install near-intents-agent-api@near-intents-agent-api
```

### Claude Code (copy)

```bash
git clone https://github.com/NEAR-Intents-Agent-API/skills
cp -R skills/near-intents-agent-api ~/.claude/skills/            # all projects
# or: cp -R skills/near-intents-agent-api .claude/skills/         # this project only
```

### Claude.ai / Claude Desktop

Zip the `near-intents-agent-api` folder and upload it under **Settings → Capabilities → Skills**.

### Other agents (Codex, Cursor, Copilot, Gemini CLI, …)

Copy `near-intents-agent-api/` into your repository (e.g. `docs/skills/`) and point your agent
instructions file (`AGENTS.md`, `.cursor/rules`, …) at it:

```md
When working with the NEAR Intents Agent API, read docs/skills/near-intents-agent-api/SKILL.md
first and follow its rules and references.
```

## What's inside

```
near-intents-agent-api/
├── SKILL.md                     rules, lifecycle, language paths, where to look
├── references/
│   ├── getting-started.md       dashboard → key → first agent → grant → deposit → swap
│   ├── concepts.md              owner / agent account / partner, policy vs grant, enforcement
│   ├── http-api.md              every endpoint, header, shape, pagination, curl
│   ├── owner-intents.md         generate → sign → submit → status, BFF split, recipes
│   ├── signing.md               nep413 / nep366 / eip712 / webauthn byte formats, per language
│   ├── policy.md                every policy field, ready-made policies, cooldowns
│   ├── grants.md                tokens, commitments, storage, revocation, patterns
│   ├── executions.md            swap, withdraw, transfer, shield, deposit, quotes, chains
│   ├── status-and-recovery.md   statuses, polling, idempotency, UNCERTAIN, /recover
│   ├── errors.md                every error code and what to do
│   ├── architecture.md          data model, jobs, custody models, AI tools, security checklist
│   ├── examples.md              map of the runnable TypeScript examples
│   ├── typescript.md            @near-intents-agent-api/sdk
│   ├── python.md                Python client + signers
│   ├── rust.md                  Rust client + signers
│   └── other-languages.md       Go client, OpenAPI codegen, Java/Kotlin/C# notes
├── assets/
│   ├── python/near_intents_agent_api.py   copy-in client + NEAR/EVM signers
│   ├── python/test_vectors.py             signer self-test
│   ├── rust/                              async client + signers (feature `evm`) + tests
│   └── test-vectors.json                  ground-truth signatures from near-api-js / viem
└── scripts/check_api.sh                   read-only smoke test for a key
```

The demo app built on the same API runs at <https://demo.agentsonintents.com>.

## Related repositories

- TypeScript SDK: [`@near-intents-agent-api/sdk`](https://www.npmjs.com/package/@near-intents-agent-api/sdk)
  (its default `baseUrl` is `https://api.agentsonintents.com`)
- Runnable examples: <https://github.com/NEAR-Intents-Agent-API/examples>
- Live contract: `https://api.agentsonintents.com/openapi.json`, guide: `/llms.txt`

## Verifying the bundled code

```bash
cd near-intents-agent-api/assets
pip install httpx cryptography eth-account && python python/test_vectors.py
cd rust && cargo test --features evm
```

Both reproduce, byte for byte, signatures produced by near-api-js and viem. If you port the
signers to another language, make it pass the same `test-vectors.json`.
