---
name: phoebe-pull-fresh-price
description: Pull a sub-heartbeat-fresh price from Phoebe via mode B (Pyth-style update + read). Use when the user says "I need a fresher price than the cache", "Pyth-style update", "the heartbeat is too slow for my liquidations", "attach a fresh signed snapshot", or otherwise needs to read a price newer than Phoebe's cached `lastSnapshotTime`.
---

# Pull a fresh price from Phoebe (mode B)

You're helping a dapp author bypass Phoebe's ~30s heartbeat by attaching a freshly-signed snapshot to the pull. Phoebe BLS-verifies the fresh aggregate under Atlas's cached `groupPk`, advances its own cache atomically, and walks your Merkle proof against the **fresh** root. Permissionless caller — the BLS aggregate is the auth.

## When to use mode B (not mode A)

| Symptom | Pick |
|---|---|
| Routine mark / display / interest-accrual / non-safety read | mode A (cached, ~18k gas) |
| Liquidation, settlement, perp close, options exercise, oracle-divergence check | mode B (fresh, ~73k gas) |
| "The cache could be 28s old and I'll lose money if so" | mode B |
| "I want the freshest possible price even at 4× the gas cost" | mode B |

**Cost callout:** ~73k gas vs ~18k for cached. Don't pay for freshness you don't need.

## The recipe

```ts
import { toNano } from '@ton/core';
import {
    Phoebe,
    PhoebeMerkleTree,
    computeSnapshotHash,
    estimatePullValue,
    PHOEBE_TESTNET,
} from '@titon-network/phoebe-sdk';

const phoebe = tonClient.open(Phoebe.createFromAddress(PHOEBE_TESTNET!.phoebe));
const cfg    = await phoebe.getConfig();

// 1. Check cache age first. If cache is fresh enough, drop down to mode A.
const { lastSnapshotTime } = await phoebe.getSnapshot();
const cacheAge             = Math.floor(Date.now() / 1000) - lastSnapshotTime;

if (cacheAge < 5) {
    return await pullCached();    // mode A is cheaper; cache is fresher than any fresh-update can be
}

// 2. Fetch the operator's currently-signed snapshot from their endpoint.
//    The endpoint should return { timestamp, root, sig, leaves } — the same
//    payload the operator BLS-signed for the most recent push.
const fresh = await operatorClient.getSignedSnapshot();

// 3. Defensive: fresh.timestamp MUST be strictly newer than the cache,
//    else Phoebe reverts E_FRESH_NOT_FRESHER (187) and you eat the gas.
if (fresh.timestamp <= lastSnapshotTime) {
    return await pullCached();    // operator endpoint is lagging; fall back
}

// 4. Build the proof against the FRESH root (not the cached one).
const freshTree = PhoebeMerkleTree.fromSparseLeaves(fresh.leaves);
const leaf      = fresh.leaves.get(42)!;

// 5. Send the pull with `freshUpdate` attached + `withFresh: true` value.
await phoebe.sendRequestPrice(consumerSender, {
    value:        estimatePullValue({ pullFee: cfg.pullFee, withFresh: true }),
    queryId:      nextQueryId(),
    feedId:       42,
    proof:        freshTree.proof(42),
    leaf,
    maxStaleness: 0,
    freshUpdate: {
        signedData: { timestamp: fresh.timestamp, root: fresh.root },
        sig:        fresh.sig,
    },
});
```

## What the operator endpoint serves

A mode-B pull needs the **exact** BLS-signed payload the operator already produced for their most recent push. The endpoint shape (operator-defined, but conventionally):

```ts
type SignedSnapshot = {
    timestamp: number;    // unix seconds — the push's timestamp
    root:      bigint;    // 256-bit Merkle root
    sig:       Buffer;    // 96-byte BLS-G2 aggregate signature
    leaves:    Map<number, PriceLeaf>;    // the leaves the operator built the tree from
};
```

The endpoint typically caches the most recent push and serves it via HTTP. Operators publish the URL in their config (see [`/titon:phoebe-deploy-and-admit`](/titon:phoebe-deploy-and-admit)).

> **Off-chain BLS sanity check.** If you maintain the signing infra yourself, the canonical hash domain is `computeSnapshotHash(phoebeAddress, timestamp, root)` and the signing call is `signMessage(secretKey, hashBytes)` — both from `@titon-network/phoebe-sdk`. Drift in this domain produces 161 `InvalidBlsSignature` on-chain.

## Defensive flow that ALWAYS works

```ts
async function pullFreshOrFallback(feedId: number): Promise<void> {
    const cfg                  = await phoebe.getConfig();
    const { lastSnapshotTime } = await phoebe.getSnapshot();
    const fresh                = await operatorClient.getSignedSnapshot();

    // If fresh isn't actually fresher, fall back to cached path.
    if (fresh.timestamp <= lastSnapshotTime) {
        return await pullCached(feedId, cfg);
    }

    const freshTree = PhoebeMerkleTree.fromSparseLeaves(fresh.leaves);
    await phoebe.sendRequestPrice(consumerSender, {
        value:        estimatePullValue({ pullFee: cfg.pullFee, withFresh: true }),
        queryId:      nextQueryId(),
        feedId,
        proof:        freshTree.proof(feedId),
        leaf:         fresh.leaves.get(feedId)!,
        maxStaleness: 0,
        freshUpdate: {
            signedData: { timestamp: fresh.timestamp, root: fresh.root },
            sig:        fresh.sig,
        },
    });
}
```

## Gas + value math

`estimatePullValue` factors in:

- `pullFee` (retained by Phoebe — fee for the read).
- `MIN_GAS_FOR_PULL` (compute reserve at Phoebe for proof walk + callback forward).
- `+0.05 TON` extra when `withFresh: true` (covers the `BLS_VERIFY` budget).

Hardcoded fallback if you can't read `cfg.pullFee` at call time: `toNano('0.15')` covers mode B at default tunables with ample slack.

## Positive externality

When your mode-B pull succeeds, Phoebe writes the fresh root + timestamp into its own cache **atomically**. Subsequent same-block (and beyond) pulls — yours or anyone else's — read your advanced cache for free via mode A. You paid the BLS-verify gas; the rest of the system gets the freshness.

## Common pitfalls

- **`E_FRESH_NOT_FRESHER` (187).** `fresh.timestamp <= lastSnapshotTime` at the moment Phoebe processed your tx. Either: (a) your operator endpoint is lagging; (b) another consumer's mode-B pull beat yours into the same block; (c) an operator's heartbeat push beat you. Always re-check `lastSnapshotTime` right before sending, and fall back to mode A when fresh isn't fresher.
- **`E_INVALID_BLS_SIGNATURE` (161).** Three causes: (1) wrong DST off-chain — use `signMessage` from the SDK, never `@noble/curves` raw; (2) hash domain drift — `computeSnapshotHash` must be called with the EXACT (phoebe address, timestamp, root) the operator signed; (3) stale `pkShare` from a pre-rotation Atlas epoch.
- **`E_WRONG_DEPLOYMENT_BINDING` (144).** Fresh-update's signed `phoebeHash` doesn't match this Phoebe's address-hash. Operator endpoint is signing for a different Phoebe deployment (mis-config) or someone is replaying a sig from a sibling Phoebe sharing the same Atlas group. Re-fetch from the right operator.
- **`E_BAD_MERKLE_PROOF` (183).** Build the proof against `fresh.root`, NOT the cached `lastRoot`. Mode-B verifies the proof against the just-verified fresh root.
- **Don't omit the freshness check.** If you blindly send mode B every time, you'll burn 4× the gas of mode A on calls where the cache was already fresh enough.
- **Don't forget bounce-safety.** The callback still arrives `bounce: false`; keep `handleFulfillPrice` cheap.

## CLI helpers

```bash
npx phoebe info <phoebe-addr> --testnet     # see lastSnapshotTime + pullFee
npx phoebe explain 187                      # what E_FRESH_NOT_FRESHER means
```

## Related skills

- `/titon:phoebe-integrate-consumer` — write the Tolk consumer + receiver. Start here if you haven't yet.
- `/titon:phoebe-debug-revert` — diagnose 161 / 187 / 183 / 144 when a mode-B pull reverts.
- `/titon:phoebe-handle-event` — `EvtSnapshotPushed` fires on mode-B pulls (with `submitter` = consumer); useful for indexers.
