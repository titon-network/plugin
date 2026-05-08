# titon plugin

Skills for **dapp authors building on the Titon network**. Today: Kronos automation (recurring on-chain jobs) and Fortuna threshold-BLS VRF (verifiable randomness). Tomorrow: more dev-facing protocols as they ship.

Installed via Claude Code's plugin marketplace — see the [marketplace README](../README.md).

## Scope: dev-facing services only

This plugin only ships skills for **services dapp devs consume**. Infrastructure layers (forgeton-sdk shared-security pool, atlas-sdk threshold-BLS substrate) are titon-internal — relevant to the Titon team and to operators running infrastructure, not to dapp authors. Their skills live in their own SDK repos under `sdk/skills/` for the audiences that need them.

## Layout

```
titon/
├── .claude-plugin/
│   └── plugin.json                    # `name: "titon"` becomes the skill namespace
├── skills/                            # one directory per skill (SKILL.md inside)
│   ├── kronos-register-job/           # build + submit a recurring job
│   ├── kronos-job-lifecycle/          # update / pause / withdraw / cancel
│   ├── kronos-target-receiver/        # write the Tolk side Kronos calls
│   ├── kronos-gas-sizing/             # pick the right gasLimit
│   ├── kronos-integration-test/       # sandbox-test the trigger end-to-end
│   ├── kronos-scaffold/               # one-shot "bolt Kronos onto my contract"
│   ├── kronos-monitor-events/         # indexer / dashboard / alerting
│   ├── kronos-debug-exit-code/        # diagnose a failed tx by exit code
│   ├── fortuna-integrate-consumer/    # add VRF to a product (the main one)
│   ├── fortuna-monitor-events/        # indexer for VRF lifecycle
│   └── fortuna-debug-exit-code/       # diagnose a failed VRF tx
└── README.md  ← you are here
```

Adding a new dev-facing protocol = a new set of `<protocol>-*` skills. The plugin grows; users get them on next `/plugin update titon`.

## Why skills (not commands)

Per the [Claude Code skills docs](https://docs.claude.com/en/docs/claude-code/skills): both `commands/` and `skills/` expose `/<name>` invocation. Skills add:

1. **Auto-invocation.** Claude reads each skill's `description:` and loads the matching one when conversation context fits — the user doesn't have to remember the slash command. Saying "I need verifiable randomness in my contract" auto-loads `fortuna-integrate-consumer`.
2. **Bundled resources.** Each skill is a directory; future versions can ship templates, example bodies, sample tx traces alongside the recipe.
3. **Frontmatter control.** Optional fields like `disable-model-invocation: true` for skills with side effects (none of ours qualify — every Titon skill is documentation, the user signs all txs).

## Source of truth

Each `SKILL.md` is the production version of the corresponding skill in the protocol's own SDK skill bundle:

- Kronos skills mirror [`titon-network/kronos`](https://github.com/titon-network/kronos) under `sdks/typescript/skills/` (TS) and `sdks/python/skills/` (Python).
- Fortuna skills mirror [`titon-network/fortuna`](https://github.com/titon-network/fortuna) under `sdks/typescript/skills/` (TS) and `sdks/python/skills/` (Python).

Updates flow from there. The plugin tracks the published SDK majors (currently `@titon-network/kronos-sdk@0.8.4` + `@titon-network/fortuna-sdk@0.4.0` on npm; `titon-network-kronos-sdk==0.8.4` + `titon-network-fortuna-sdk==0.4.0` on PyPI); pin a plugin version if you need stability across upgrades.

## Conventions

- **One skill per protocol concern.** Don't fold five concerns into one; Claude picks the right skill more reliably when each is narrowly scoped and its `description` clearly enumerates trigger phrases.
- **Skills cross-reference dev-facing only.** Use `/titon:<skill-name>` in "Related skills" footers — the actual slash-command form a user would type. Don't cross-reference operator/admin skills (they're not in this plugin).
- **Code snippets reference the right SDK as the dependency.** `@titon-network/kronos-sdk` for registry ops, `@titon-network/fortuna-sdk` for VRF ops. Forgeton/atlas surfaces don't appear in dapp-author snippets.
- **Live-deployment addresses come from SDK constants.** `KRONOS_TESTNET` / `KRONOS_MAINNET` / `FORTUNA_TESTNET` / `FORTUNA_MAINNET` — never hardcoded in snippets. Both networks are live; skill snippets default to testnet for safe iteration but the same recipe drives mainnet by swapping the import. Imports from `@titon-network/<protocol>-sdk` resolve to the latest published version.
