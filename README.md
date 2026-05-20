# titon-network/plugin

Claude Code plugin marketplace for the Titon network. **One install gives any AI agent a fluent vocabulary for shipping a Titon-integrated dapp** — from "I need verifiable randomness" to deployed contract in one focused conversation. Four products on equal footing: Kronos automation, Fortuna threshold-BLS VRF, Phoebe price oracle, and Themis sealed-bid threshold-decryption. More dev-facing protocols as they ship.

> **Live on TON mainnet.** All four protocols TSA-audited (zero findings) and live on testnet + mainnet. Skill examples default to testnet for safe iteration; flip the SDK constant import (`KRONOS_TESTNET → KRONOS_MAINNET`, etc.) and the same recipes drive production. Pinned SDKs: `@titon-network/{kronos-sdk@0.8.5,fortuna-sdk@0.6.0,phoebe-sdk@0.5.0,themis-sdk@0.3.0}` (npm); Kronos/Fortuna additionally on PyPI as `titon-network-{kronos,fortuna}-sdk`.

Built for **dapp authors**. Infrastructure-side surfaces (forgeton pool admin, atlas DKG bootstrap, automaton operator ops, multi-op share-exchange) are titon-internal and live in their own SDK skill bundles for the audiences that need them.

## Install

```
/plugin marketplace add titon-network/plugin
/plugin install titon
```

That's it. Restart Claude Code to load the skills.

## What you get

After `/plugin install titon`, **32 skills** become available across four protocols. Kronos and Fortuna recipes ship in two flavours (TypeScript + Python, kept in lockstep); Phoebe and Themis are TypeScript today (Python parity follow-up). Claude **auto-loads** the right one when your conversation matches (e.g. "I want to schedule a periodic on-chain call from a TS Blueprint project" → `kronos-register-job`; "read TON/USD from Phoebe" → `phoebe-consume-price`; "build a sealed-bid auction" → `themis-integrate-consumer`). You can also invoke explicitly with `/titon:<skill-name>`.

The TS skills are backed by `@titon-network/{kronos,fortuna,phoebe,themis}-sdk` on npm; the Python skills by `titon-network-{kronos,fortuna}-sdk` on PyPI. Same on-chain protocols, same TSA-audited bytecode — pick the language that matches your stack.

### Kronos — decentralized automation

Register recurring on-chain jobs; permissionless automatons execute them. **🛡️ TSA-audited — zero findings** ([report](https://github.com/titon-network/kronos/blob/main/tsa-analysis/AUDIT-REPORT.md)).

| TypeScript skill | Python skill | Auto-loads when… |
|------------------|--------------|------------------|
| `kronos-register-job` | `kronos-register-job-python` | You want to schedule a periodic on-chain call |
| `kronos-job-lifecycle` | `kronos-job-lifecycle-python` | You want to update / pause / withdraw / cancel an existing job |
| `kronos-target-receiver` | `kronos-target-receiver-python` | You're writing the Tolk target Kronos will call |
| `kronos-gas-sizing` | `kronos-gas-sizing-python` | You ask "what `gasLimit` should I use?" / your job OOGs silently |
| `kronos-integration-test` | `kronos-integration-test-python` | You're writing a sandbox / pytest test for a Kronos-triggered contract |
| `kronos-scaffold` | `kronos-scaffold-python` | You want a one-shot "bolt Kronos onto my contract" action |
| `kronos-monitor-events` | `kronos-monitor-events-python` | You're building an indexer / dashboard / alerting bot |
| `kronos-debug-exit-code` | `kronos-debug-exit-code-python` | You paste a failed transaction or ask what a code means |

### Fortuna — verifiable randomness (threshold-BLS VRF)

Add unbiasable randomness to a product contract: raffle, NFT trait roll, dice, lottery, mystery box, shuffled queue. **🛡️ TSA-audited — zero findings** ([report](https://github.com/titon-network/fortuna/blob/main/AUDIT_REPORT.md)).

| TypeScript skill | Python skill | Auto-loads when… |
|------------------|--------------|------------------|
| `fortuna-integrate-consumer` | `fortuna-integrate-consumer-python` | You want to add VRF to your product (write `VrfCallback`, send `RequestRandomness`) |
| `fortuna-monitor-events` | `fortuna-monitor-events-python` | You're building an indexer for randomness requests / fulfillments |
| `fortuna-debug-exit-code` | `fortuna-debug-exit-code-python` | You paste a failed Fortuna tx or ask what a code means |

### Phoebe — BLS-attested price oracle

Pull verified on-chain prices into your dapp: vault collateralisation, lending markups, perp / options settlement, liquidation gates. Operators heartbeat a BLS-attested merkle root every ~30s; consumers pull individual feeds with one `BLS_VERIFY` + one cell-walk. **🛡️ TSA-audited — zero findings**.

| TypeScript skill | Auto-loads when… |
|------------------|------------------|
| `phoebe-consume-price` | You're reading a price from a **TS/frontend layer** — `fetchVerifiedPrice` fetches leaves from operator HTTP, verifies the merkle root against on-chain, returns `(leaf, proof)` |
| `phoebe-integrate-consumer` | You're writing the **Tolk consumer contract** that receives `FulfillPrice` (Mode A cached + Mode B fresh-update) |
| `phoebe-deploy-and-admit` | You're standing up a Phoebe instance (testnet / mainnet) |
| `phoebe-handle-event` | You're decoding Phoebe events for an indexer / dashboard |
| `phoebe-debug-revert` | You hit a revert and want the exit code → root-cause map |

### Themis — sealed-bid threshold-decryption

Build MEV-resistant on-TON products: sealed-bid auctions, encrypted limit orders, confidential governance, batched fair-clearing DEX swaps. Users encrypt intents under Atlas's group key; operators threshold-decrypt batches off-chain after a commit window; your consumer contract receives a verified batch reveal. **🛡️ TSA-audited — zero findings**. Permissionless `DeployChamber` — every dapp deploys its own chamber via the singleton factory.

| TypeScript skill | Auto-loads when… |
|------------------|------------------|
| `themis-integrate-consumer` | You're writing the **Tolk consumer** that processes `RevealCallback` (batched sealed-bid reveal → your domain logic) |
| `themis-bidder-flow` | You're building the **bidder side** — encrypt a sealed bid under the chamber's group key and submit `SubmitCiphertext` |
| `themis-deploy-chamber` | You're spawning a **per-dapp chamber** via the singleton factory's `DeployChamber` |
| `themis-debug` | You hit a revert and want the exit code → root-cause map (BLS_VERIFY failed, operator not active, AEAD decrypt failed, etc.) |

Each skill loads a focused recipe — what to ask the user, what code to generate, what gotchas to surface — into Claude's context only when relevant. They cross-reference where flows intersect.

The Tolk receiver code is identical regardless of language; what differs across the TS/Python pair is the **off-chain tooling** (deploy, test, indexing) and the SDK API surface (camelCase TS vs snake_case Python; `@ton/sandbox` vs testnet pytest; etc.).

## Quick start a Fortuna dapp in 30 seconds

**TypeScript:**

```bash
npm install @titon-network/fortuna-sdk @ton/core @ton/blueprint
npx fortuna init                  # writes FORTUNA.md AI-context brief into cwd
npx fortuna scaffold consumer     # drops a fully-commented Tolk consumer template
npx fortuna estimate request --callback-gas 0.05   # exact request value floor
```

Or clone the worked dapp:

```bash
EX=node_modules/@titon-network/fortuna-sdk/examples/coin-flip
cp $EX/coin-flip.tolk          contracts/
cp $EX/CoinFlip.compile.ts     wrappers/
cp $EX/CoinFlip.ts             wrappers/
cp $EX/CoinFlip.spec.ts        tests/
npx blueprint test             # full request → fulfill → callback in sandbox
```

**Python:**

```bash
pip install titon-network-fortuna-sdk pytoniq
```

Then ask Claude "I want to integrate Fortuna VRF from Python" — `fortuna-integrate-consumer-python` auto-loads with the Tolk consumer skeleton + a Python trigger that calls `Fortuna.create_from_address(FORTUNA_TESTNET.fortuna).send_request_randomness(...)`. The Tolk part is identical to the TS path; only the off-chain trigger and tests differ.

## What's coming

- **Python parity for Phoebe + Themis.** Mirror of the existing Kronos / Fortuna `*-python` skill pattern, backed by `titon-network-{phoebe,themis}-sdk` on PyPI.
- **Phoebe Mode B helper.** Off-chain BLS-partial aggregation for sub-heartbeat freshness reads (Pyth-style update + read in one tx). Tolk side already supported; TS helper is on the v0.6 roadmap.
- **Themis chamber discovery.** Auto-pickup of new chambers via the factory's `ChamberDeployed` events, so operators serving multiple chambers don't need static config.
- **MCP auto-wiring.** [`@titon-network/mcp@0.6.0`](https://www.npmjs.com/package/@titon-network/mcp) is published and the team-hosted instance at `mcp.titon.network` is being brought online — once live, Claude can *call* tools (not just generate code that calls them) by pointing your client config at `https://mcp.titon.network`. Next plugin version will register this in your MCP config automatically.
- **More dev-facing protocols.** Future Titon services that dapp authors consume directly will join the plugin. Operator/admin/infra skills stay in their respective SDK repos.
- **Subagents.** Per-protocol architect subagents (`kronos-architect`, `fortuna-architect`, `phoebe-architect`, `themis-architect`) holding the full mental model in context for deep design conversations.

## Layout

```
.
├── .claude-plugin/
│   └── marketplace.json       # marketplace manifest — Claude Code reads this on `marketplace add`
└── titon/                     # the plugin
    ├── .claude-plugin/
    │   └── plugin.json        # plugin manifest — `name` field becomes the skill namespace
    ├── skills/                # one directory per skill (Claude Code's preferred format)
    │   └── <skill-name>/
    │       └── SKILL.md       # frontmatter + recipe body
    └── README.md              # plugin-level docs
```

The `name: "titon"` in `plugin.json` is what namespaces every skill — that's why explicit invocation is `/titon:<skill-name>`.

## Updating

`/plugin update titon` pulls the latest from this repo. Skill content tracks the published Kronos and Fortuna SDK majors; pin a plugin version explicitly if you want stability across upgrades.

## Other AI tools

This plugin uses Claude Code's plugin format. For Cursor / Zed / Aider / generic LLMs, the same skill content ships in the SDK packages:

- `@titon-network/fortuna-sdk` (npm) and `titon-network-fortuna-sdk` (PyPI) include `AGENTS.md`, `skills/`, `templates/`, `examples/`, `llms-full.txt` / `llms.txt`. Run `npx fortuna init` to write a `FORTUNA.md` brief into your repo root.
- `@titon-network/kronos-sdk` (npm) and `titon-network-kronos-sdk` (PyPI) include `AGENTS.md`, `skills/`, `examples/`. Reference from your assistant's `.cursorrules` / `agents.md`.

Same content, different distribution mechanism.

## License

MIT.
