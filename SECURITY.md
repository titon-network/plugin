# Security policy

`titon-network/plugin` is a Claude Code plugin marketplace — pure markdown skill files + JSON manifests, no executable code, no key custody, no network calls.

The "security surface" is essentially: **does a skill instruct an AI agent to do something dangerous?** Examples of in-scope findings:

- A skill that tells the agent to expose mnemonics, leak `.env` contents, or send funds without explicit user approval.
- A skill with code examples that contain known-vulnerable patterns (e.g., a deploy snippet that hardcodes a mnemonic in a commit).
- A skill that misrepresents an on-chain operation's blast radius (e.g., describing `SetForgeton` as reversible when it's one-shot).

## Reporting

- Email **`security@titon.network`** with `[plugin]` in the subject.
- Or use GitHub's [private vulnerability reporting](https://github.com/titon-network/plugin/security/advisories/new).

Triage is best-effort — most reasonable concerns can also be filed as a public issue or PR since there's no exploitable runtime here.

## Out of scope

- Issues with the underlying contract behaviour the skills describe — report against the relevant protocol repo ([forgeton](https://github.com/titon-network/forgeton/security/advisories/new), [kronos](https://github.com/titon-network/kronos/security/advisories/new), [fortuna](https://github.com/titon-network/fortuna/security/advisories/new)).
- Claude Code itself — report to Anthropic via [bugcrowd](https://bugcrowd.com/anthropic).
