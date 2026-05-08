# titon-network/plugin

Claude Code plugin marketplace for the Titon network. **One install gives any AI agent a fluent vocabulary for shipping a Titon-integrated dapp** — from "I need verifiable randomness" to deployed contract in one focused conversation. Kronos automation and Fortuna threshold-BLS VRF today; more dev-facing protocols as they ship.

> **Live on TON mainnet.** Both protocols TSA-audited (zero findings) and live on testnet + mainnet. Skill examples default to testnet for safe iteration; flip the SDK constant import (`KRONOS_TESTNET → KRONOS_MAINNET`, `FORTUNA_TESTNET → FORTUNA_MAINNET`) and the same skill recipes drive production. Pinned SDKs: `@titon-network/kronos-sdk@0.8.4` + `@titon-network/fortuna-sdk@0.4.0` (npm) / `titon-network-kronos-sdk==0.8.4` + `titon-network-fortuna-sdk==0.4.0` (PyPI).

Built for **dapp authors**. Infrastructure-side surfaces (forgeton pool admin, atlas DKG bootstrap, automaton operator ops, multi-op Fortuna share-exchange) are titon-internal and live in their own SDK skill bundles for the audiences that need them.

## Install

```
/plugin marketplace add titon-network/plugin
/plugin install titon
```

That's it. Restart Claude Code to load the skills.

## What you get

After `/plugin install titon`, **22 skills** become available — every Kronos and Fortuna recipe ships in two flavours, TypeScript and Python, kept in lockstep. Claude **auto-loads** the right one when your conversation matches (e.g. "I want to schedule a periodic on-chain call from a TS Blueprint project" → `kronos-register-job`; "…from a Python service" → `kronos-register-job-python`). You can also invoke explicitly with `/titon:<skill-name>`.

The TS skills are backed by `@titon-network/{kronos,fortuna}-sdk` on npm; the Python skills by `titon-network-{kronos,fortuna}-sdk` on PyPI. Same on-chain protocols, same TSA-audited bytecode — pick the language that matches your stack.

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

- **MCP auto-wiring.** [`@titon-network/mcp@0.6.0`](https://www.npmjs.com/package/@titon-network/mcp) is published and the team-hosted instance at `mcp.titon.network` is being brought online — once live, Claude can *call* Kronos / Fortuna tools (not just generate code that calls them) by pointing your client config at `https://mcp.titon.network`. Next plugin version will register this in your MCP config automatically.
- **More dev-facing protocols.** Future Titon services that dapp authors consume directly will join the plugin. Operator/admin/infra skills stay in their respective SDK repos.
- **Subagents.** Per-protocol architect subagents (`kronos-architect`, `fortuna-architect`) holding the full mental model in context for deep design conversations.

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
