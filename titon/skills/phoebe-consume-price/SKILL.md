---
name: phoebe-consume-price
description: Read a verified TON-mainnet price from Phoebe in a dapp's TypeScript layer — fetch the latest snapshot leaves from any operator's public HTTP endpoint, verify the merkle root against on-chain `phoebe.lastRoot`, return a `(leaf, proof)` pair ready to submit. Use when the user says "read a price from Phoebe", "get TON/USD from Phoebe", "use fetchVerifiedPrice", "build a Phoebe-consuming frontend", "I need a price oracle for my TON dapp", or "how does my dapp consume Phoebe without running an operator". For the Tolk consumer contract side use `phoebe-integrate-consumer`.
---

# Consume a Phoebe price from a dapp (TypeScript side)

You're helping a dapp author read a verified price from Phoebe — the
TON-native threshold-BLS price oracle. This skill is the **TypeScript /
frontend / off-chain** side. For the on-chain `RequestPrice` Tolk
consumer contract that receives the verified `(mantissa, expo)` callback,
load `phoebe-integrate-consumer` instead.

The two compose: this skill gets the dapp a `(leaf, proof)` pair; the
other skill writes the Tolk contract that submits them.

## Why this skill exists

Phoebe stores **only the merkle root on-chain** — never individual
prices. To USE a price, a consumer contract has to submit a `(leaf,
proof)` pair that hashes to the root. The leaves live in the operator
network's memory + are served over a public HTTP endpoint:

```
GET <op-url>/phoebe/v1/snapshot
→ { timestamp, rootHex, operatorAddress, leaves: [...] }
```

`fetchVerifiedPrice` (added in `@titon-network/phoebe-sdk@0.5.0`) is
the one-call API that does this safely: parallel-fetch all operators,
reconstruct the merkle root locally from the leaves, compare to
`phoebe.lastRoot` on-chain, and return only when one operator's
response verifies. Lying / stale / unreachable operators get skipped
automatically.

## The three-line read

```ts
import { TonClient } from '@ton/ton';
import { Phoebe, PHOEBE_MAINNET, assertDeployment, fetchVerifiedPrice }
    from '@titon-network/phoebe-sdk';

const client = new TonClient({ endpoint: 'https://toncenter.com/api/v2/jsonRPC' });
const dep    = assertDeployment('mainnet');                    // PHOEBE_MAINNET
const phoebe = client.open(Phoebe.createFromAddress(dep.phoebe));

const quote = await fetchVerifiedPrice(phoebe, /*feedId*/ 0, dep.operators ?? []);
// quote.mantissa, quote.expo  → price = mantissa × 10^expo
// quote.proof, quote.leaf     → pass to your consumer contract
```

`dep.operators` ships in the SDK and is kept in lockstep with the
live operator set (currently EU + US). Dapps can also supply their own
list.

## Reading the price as a number (UI)

```ts
const price = Number(quote.mantissa) * Math.pow(10, quote.expo);
console.log(`TON/USD = $${price.toFixed(4)}`);
// → "TON/USD = $1.9550"
```

`mantissa` is `bigint` to preserve precision for large markets
(BTC at $100k × 10⁸ fits without rounding). Cast to `Number` only when
you've narrowed the range.

## Submitting `(leaf, proof)` to your consumer contract

The consumer contract is your dapp's on-chain code. It sends
`RequestPrice` (0x71) to Phoebe with the leaf + proof; Phoebe verifies
the proof against `lastRoot` and calls back the consumer at
`FulfillPrice` (0x72) with the verified price.

```ts
import { sendRequestPrice } from '@titon-network/phoebe-sdk';

await sendRequestPrice(consumer, {
    feedId:    quote.feedId,
    leaf:      quote.leaf,
    proof:     quote.proof,
    // …consumer-specific fields…
});
```

For the **Tolk side** of this — writing the consumer that handles
`FulfillPrice` — load **`phoebe-integrate-consumer`**.

## Staleness handling

`fetchVerifiedPrice` returns whatever the operators have, which may
be a few seconds old. Three options:

```ts
// 1. Inspect ageSec in app code:
if (quote.ageSec > 60) throw new Error('phoebe price too stale');

// 2. Enforce client-side via maxAgeSec:
const quote = await fetchVerifiedPrice(phoebe, 0, dep.operators!, {
    maxAgeSec: 30,
});

// 3. Enforce on-chain — your consumer contract checks `leaf.pubTime`
//    against `now()` before acting. RECOMMENDED for production
//    (trustless freshness).
```

## Trust model — why this is "elite"-tier

`fetchVerifiedPrice` reconstructs the merkle root locally from the
fetched leaves and compares to `phoebe.lastRoot` on-chain. A lying
operator can't forge a matching root because they don't hold the
threshold BLS secret that signed the on-chain state. A *stale*
operator (whose snapshot rolled off-chain) gets caught by the same
hash mismatch and the helper falls through to the next operator
automatically.

The operator endpoints in `PHOEBE_MAINNET.operators` ship hardcoded in
the SDK for v0.5; future versions may discover them on-chain via a
phoebe operator registry. Either way, the helper's trust assumption
is the same: only the on-chain root matters.

## Error model

`fetchVerifiedPrice` throws on:

| Reason | Recovery |
|---|---|
| Empty `operators` list | Pass `PHOEBE_MAINNET.operators` or your own. |
| `phoebe.lastRoot === 0n` (no snapshot ever) | Wait ~30s after group key publish for first push. |
| Every operator unreachable / 5xx | Transient — retry. |
| Every operator served a snapshot whose root ≠ on-chain | Operators are mid-roll (between push windows) OR a malicious set served the same lie — retry in ~30s. |
| Snapshot has no leaf for `feedId` | That feed isn't published. Confirm the feedId is in the canonical registry. |
| `ageSec > maxAgeSec` | Snapshot older than your bound — retry or relax `maxAgeSec`. |

Retry with backoff (10-30s between push windows) is the right pattern
for transient failures.

## Multi-feed snapshots — reuse the same operator

A snapshot commits to many feeds in one merkle root. If you need
several feeds in the same atomic read, pin to the operator that
served the first response — its snapshot is what `lastRoot` matches:

```ts
const tonUsd = await fetchVerifiedPrice(phoebe, 0, dep.operators!);
const btcUsd = await fetchVerifiedPrice(phoebe, 1, [
    dep.operators!.find((o) => o.address === tonUsd.sourceOperator)!,
]);
```

This avoids re-verifying against a different snapshot if a push
window rolls between the two calls.

## Pyth-style mode B (update + read)

For sub-heartbeat freshness, Phoebe supports mode B: the consumer
attaches a fresh signed snapshot inline with its `RequestPrice` and
Phoebe verifies + advances the cache + walks the proof against the
fresh root, all in one tx. Costs ~73k gas vs ~16k for mode A.

**This skill covers mode A only.** Mode B requires aggregating BLS
partials off-chain before submission. For the Tolk side of mode B,
see `phoebe-integrate-consumer`. A mode-B TS helper is on the v0.6
roadmap.

## Production checklist

- [ ] Pin `@titon-network/phoebe-sdk` to a specific minor (e.g. `^0.5.0`)
- [ ] Use `PHOEBE_MAINNET.operators` — the SDK bumps this list as new operators come online
- [ ] Don't trust `quote.ageSec` for security; enforce `leaf.pubTime` staleness ON-CHAIN in your consumer contract
- [ ] Catch `fetchVerifiedPrice` errors and retry with backoff (10-30s)
- [ ] If your TPS requires it, cache the verified quote for the current push window in your dapp — don't re-fetch on every read
- [ ] Browser dapps: the endpoint is CORS-open (`Access-Control-Allow-Origin: *`); no proxy needed

## When NOT to use this skill

Load a different skill if:

- The user is writing the Tolk **consumer contract** that receives
  `FulfillPrice` → `phoebe-integrate-consumer`
- The user is running a Phoebe **operator** (signing snapshots,
  managing the daemon) → `phoebe-operator-setup` (in the SDK skills)
- The user is **deploying** a fresh Phoebe instance → `phoebe-deploy-and-admit`
- The user is **debugging a revert** they hit → `phoebe-debug-revert`

## Related types in the SDK

| Type | Purpose |
|---|---|
| `OperatorEndpoint` | `{ address, url }` — one operator's public read endpoint |
| `VerifiedPriceQuote` | `{ feedId, mantissa, expo, confBps, pubTime, snapshotTime, root, proof, leaf, sourceOperator, ageSec }` — the result |
| `FetchVerifiedPriceOptions` | `{ maxAgeSec?, timeoutMs?, nowSec?, fetchImpl? }` — tunables + test injection points |

All exported from `@titon-network/phoebe-sdk` root.
