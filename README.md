<div align="center">

# NEAR Intents Agent API: Agent Skills

**Teach your coding agent to build on the NEAR Intents Agent API, in any language.**

[Overview](#overview) · [Quick start](#quick-start) · [Install](#install) · [What's inside](#whats-inside) · [Verify the bundled code](#verify-the-bundled-code)

</div>

## Overview

A skill for coding agents (Claude Code, Claude.ai, Codex, Cursor, Copilot, Gemini CLI, …) that
knows the [NEAR Intents Agent API](https://api.agentsonintents.com/llms.txt): give every AI agent
its own agent account on NEAR Intents, let the end user sign the rules once with their own
wallet, and let the agent swap, transfer, withdraw and receive funds inside those rules. Works for
TypeScript, Python, Rust, Go or anything that speaks HTTP.

```
 your backend (partner)            end user (owner)                  AI agent
 ─────────────────────            ────────────────                  ────────
 naa_ API key                      wallet: NEAR / EVM / passkey      ngt_ grant token (held by your backend)
 prepares owner actions   ──────►  signs: create account, policy,
 stores grant tokens               grants, freeze, approve, delete
 calls executions         ◄─────────────────────────────────────────  swap / transfer / withdraw / deposit
                                                                       … only inside the owner-signed policy
```

- **Policy = what** may happen: actions, assets, per-asset limits, destinations, USD budget,
  timelock, weekly schedule, owner approval, freeze.
- **Grant = who** may act: one revocable token per session, assistant or bot.
- Your backend never holds the owner's key, the browser never sees your API key, and the model
  never sees either.

## Quick start

1. **Get a key.** Create a partner API key in the [partner dashboard](https://partners.near-intents.org/) (**API keys**).
   The `naa_…` token is shown once; store it as `NEAR_INTENTS_AGENT_API_KEY` in your backend's secrets.
2. **Check it.**
   ```sh
   NEAR_INTENTS_AGENT_API_KEY=naa_... ./near-intents-agent-api/scripts/check_api.sh
   ```
3. **Install the skill** (below) and ask your coding agent, for example:
   *"Add NEAR Intents agent wallets to our FastAPI backend: onboarding with the user's NEAR
   wallet, one grant per chat session, and a swap tool for the assistant."*

> [!WARNING]
> The hosted API (`https://api.agentsonintents.com`) is mainnet with real funds. Build with small
> amounts and dedicated owner accounts.

## Install

<details open>
<summary><b>Claude Code</b> (plugin marketplace)</summary>

```text
/plugin marketplace add NEAR-Intents-Agent-API/skills
/plugin install near-intents-agent-api@near-intents-agent-api
```

Or copy the skill:

```sh
git clone https://github.com/NEAR-Intents-Agent-API/skills
cp -R skills/near-intents-agent-api ~/.claude/skills/            # all projects
# or: cp -R skills/near-intents-agent-api .claude/skills/         # this project only
```

</details>

<details>
<summary><b>Claude.ai / Claude Desktop</b></summary>

Zip the `near-intents-agent-api` folder and upload it under **Settings → Capabilities → Skills**.

</details>

<details>
<summary><b>Other agents</b> (Codex, Cursor, Copilot, Gemini CLI, …)</summary>

Copy `near-intents-agent-api/` into your repository (for example `docs/skills/`) and point your
agent instructions file (`AGENTS.md`, `.cursor/rules`, …) at it:

```md
When working with the NEAR Intents Agent API, read docs/skills/near-intents-agent-api/SKILL.md
first and follow its rules and references.
```

</details>

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
│   ├── architecture.md          data model, jobs, key ownership models, AI tools, security checklist
│   ├── examples.md              map of the runnable TypeScript examples
│   ├── typescript.md            @near-intents-agent-api/sdk
│   ├── python.md                Python client + signers
│   ├── rust.md                  Rust client + signers
│   └── other-languages.md       Go client, OpenAPI codegen, Java/Kotlin/C# notes
├── assets/
│   ├── python/                  copy-in client + NEAR/EVM signers, signer self-test
│   ├── rust/                    async client + signers (feature `evm`) + tests
│   └── test-vectors.json        ground-truth signatures from near-api-js / viem
└── scripts/check_api.sh         read-only smoke test for a key
```

## Verify the bundled code

The Python and Rust signers reproduce, byte for byte, signatures produced by near-api-js and viem:

```sh
cd near-intents-agent-api/assets
pip install httpx cryptography eth-account && python python/test_vectors.py
cd rust && cargo test --features evm
```

> [!TIP]
> Porting the signers to another language? Make it pass the same `test-vectors.json`.

## Related

- [`@near-intents-agent-api/sdk`](https://www.npmjs.com/package/@near-intents-agent-api/sdk): TypeScript client ([source](https://github.com/NEAR-Intents-Agent-API/sdk-typescript))
- [examples](https://github.com/NEAR-Intents-Agent-API/examples): runnable TypeScript scripts
- [api](https://github.com/NEAR-Intents-Agent-API/api): the service; contract at `https://api.agentsonintents.com/openapi.json`, guide at `/llms.txt`
- [demo](https://demo.agentsonintents.com): an app built on the same API
