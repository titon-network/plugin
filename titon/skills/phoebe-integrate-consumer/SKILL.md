---
name: phoebe-integrate-consumer
description: Pull verified prices from Phoebe — TON's threshold-BLS price oracle. Use when the user wants to integrate a price feed, "use Phoebe in my contract", read BTC/ETH/any price on-chain, build a price-aware vault / lending market / perp / options product, or otherwise consume Phoebe snapshots from a Tolk consumer.
---

# Integrate a Phoebe price-oracle consumer

You're helping a dapp author add verified on-chain price reads. Phoebe's consumer surface is intentionally tiny: send `RequestPrice` (0x71) → receive `FulfillPrice` (0x72). One `BLS_VERIFY` on push + one cell-merkle walk per pull, any of 256 feeds from a single signed root.

Two pull modes — pick by freshness need:

- **Mode A — cached fast-path** (~18k gas, default). Phoebe walks your Merkle proof against the cached `lastRoot`. No BLS verify. Use for routine reads.
- **Mode B — Pyth-style update + read** (~73k gas). Attach a fresh signed snapshot inline; Phoebe verifies + advances cache + walks your proof against the fresh root. Permissionless caller (BLS is the auth). Use when sub-heartbeat freshness matters — liquidations, settlement, oracle-divergence checks.

## The fastest path: clone the worked example

```bash
npm install @titon-network/phoebe-sdk @ton/core
# In your Blueprint project root:
cp -r node_modules/@titon-network/phoebe-sdk/examples/price-aware-vault/* .
npx blueprint test tests/PriceAwareVault.spec.ts
```

The example uses `deployPhoebeFixture` from the SDK testing subpath (no testnet wallet, no funding) — full deploy → push → pull → callback in-process. Adapt the storage + business logic to your domain.

## Or scaffold a fresh starter

```bash
npx phoebe scaffold consumer --out contracts/my-consumer.tolk
```

The template carries the receiver-safety banner, sender-pin assertion, and a placeholder `handleFulfillPrice`. Customize:

1. Replace the placeholder body with your price-consuming logic (settle, liquidate, fill, mark).
2. Inline `RequestPrice` directly inside whatever internal event needs a fresh price (a swap, a settle, a liquidation check).
3. Add a `map<queryId, RequestState>` if multiple pulls can be in flight.

## Wire types — copy verbatim from `@titon-network/phoebe-sdk` source

```tolk
// Outbound to Phoebe.
struct (0x00000071) RequestPrice {
    queryId:        uint64                    // consumer-supplied; echoed back
    feedId:         uint16                    // 0..255 in v1
    proofRef:       cell                      // pruned-branch merkle_proof
    leaf:           PriceLeaf                 // claimed leaf at slot `feedId`
    maxStaleness:   uint32                    // 0 = accept any active root
    freshUpdateRef: Cell<FreshUpdate>?        // null = mode A; set = mode B
}

// Inbound from Phoebe.
struct (0x00000072) FulfillPrice {
    queryId: uint64                           // echoes your request
    leaf:    PriceLeaf                        // verified leaf — trust the fields
}

struct PriceLeaf {
    feedId:   uint16
    mantissa: int128                          // signed; price = mantissa × 10^expo
    expo:     int8
    confBps:  uint16                          // confidence interval (basis points)
    pubTime:  uint32                          // unix seconds when produced off-chain
}
```

Drift breaks the lazy-decoder silently. Re-copy after every Phoebe upgrade.

## Tolk skeleton

```tolk
const E_NOT_PHOEBE: int = 201;

struct ConsumerStorage {
    owner:        address
    phoebe:       address                     // pinned at deploy
    nextQueryId:  uint64
    // ... your business state
}

fun handleFulfillPrice(msg: FulfillPrice, sender: address) {
    var storage = lazy ConsumerStorage.load();
    assert (sender == storage.phoebe) throw E_NOT_PHOEBE;

    // `msg.leaf` is verified. Price = mantissa × 10^expo. confBps is uncertainty.
    val priceMantissa: int128 = msg.leaf.mantissa;
    val priceExpo:     int8   = msg.leaf.expo;

    // ... your settle / liquidate / fill logic ...
    storage.save();
}

fun triggerSettle(sender: address) {
    var storage = lazy ConsumerStorage.load();
    val queryId = storage.nextQueryId;
    storage.nextQueryId = queryId + 1;
    // ... build the proof + body (off-chain via PhoebeMerkleTree, attached as a ref) ...
    storage.save();
}
```

## TypeScript trigger (off-chain or test)

```ts
import { toNano } from '@ton/core';
import {
    Phoebe,
    PhoebeMerkleTree,
    estimatePullValue,
    PHOEBE_TESTNET,
} from '@titon-network/phoebe-sdk';

const phoebe = tonClient.open(Phoebe.createFromAddress(PHOEBE_TESTNET!.phoebe));
const cfg    = await phoebe.getConfig();

// Off-chain: build a tree from the latest signed leaves (cache + indexer).
const tree   = PhoebeMerkleTree.fromSparseLeaves(latestLeaves);
const leaf   = latestLeaves.get(42)!;

await phoebe.sendRequestPrice(consumerSender, {
    value:        estimatePullValue({ pullFee: cfg.pullFee }),    // mode A
    queryId:      nextQueryId(),
    feedId:       42,
    proof:        tree.proof(42),
    leaf,
    maxStaleness: 0,                                              // accept any active root
});
```

## The 5 rules every consumer follows

1. **Pin Phoebe at deploy.** `storage.phoebe: address` set once. Repointing requires a contract upgrade.
2. **Sender-gate the callback.** `assert (sender == storage.phoebe) throw E_NOT_PHOEBE;` at the top of `handleFulfillPrice`. Without this, anyone can deliver fake prices by sending a 0x72 body.
3. **Unique queryId per pending request.** Use a monotone counter on-chain or `QueryIdStream` from the SDK off-chain. (Phoebe doesn't track per-query state, but your callback dispatcher needs it.)
4. **Funded request value.** Floor = `cfg.pullFee + ~0.03 TON` (mode A) or `+ ~0.05 TON` (mode B). Use `estimatePullValue({ pullFee, withFresh })` — don't hardcode.
5. **Revert-proof callback.** Phoebe sends `FulfillPrice` with `bounce: false` — a revert in your handler does NOT bounce back. Keep the callback cheap and assertion-light; emit an event and process heavy work in a separate message.

## Building the proof off-chain

```ts
import { PhoebeMerkleTree } from '@titon-network/phoebe-sdk';

// Sparse map of feedId → leaf — empty slots get the canonical placeholder.
const tree  = PhoebeMerkleTree.fromSparseLeaves(new Map([[42, myLeaf]]));
const proof = tree.proof(42);    // proof.hash() === tree.rootAsBigint()
```

In production, an off-chain indexer watches `EvtSnapshotPushed` events + caches the operator-signed leaves. The consumer fetches them from the indexer at pull time. **Don't reuse a proof across snapshots** — every root rotation invalidates prior proofs.

## Cost trade-offs at a glance

| Mode | Gas | When to use |
|---|---|---|
| A (cached) | ~18k + pullFee | Routine reads, marks, periodic mark-to-market |
| B (fresh-update) | ~73k + pullFee | Liquidations, settlement, perp closes, options exercise — anywhere stale prices break safety |

Positive externality of mode B: the cache advances atomically, so subsequent same-block pulls read your advanced cache for free.

## Testing with the sandbox fixture

```ts
import { Blockchain } from '@ton/sandbox';
import { deployPhoebeFixture } from '@titon-network/phoebe-sdk/testing';

const blockchain = await Blockchain.create();
const fx         = await deployPhoebeFixture(blockchain);

const leaf       = { feedId: 42, mantissa: 100n, expo: 0, confBps: 50, pubTime: 2_000_000_000 };
const { tree }   = await fx.pushSnapshot({ leaves: new Map([[42, leaf]]) });

// Your consumer can now call `requestPrice` against fx.phoebe.address.
```

Full vault round-trip in `node_modules/@titon-network/phoebe-sdk/examples/price-aware-vault/`.

## Common pitfalls

- **Don't trust an unauthenticated `FulfillPrice` body.** Sender-pin is non-negotiable.
- **Don't reuse a Merkle proof across snapshots.** Every push rotates `lastRoot`; old proofs revert with `E_BAD_MERKLE_PROOF` (183).
- **Don't pass a leaf that doesn't match the proof.** `leaf.feedId` must equal `msg.feedId`; the leaf cell hash must match the proof's claimed leaf.
- **Don't hand-decode the FulfillPrice body.** Use `parseFulfillPrice(body)` — `mantissa` is `int128`, easy to get the widths wrong.
- **Don't forget mode B's permissionless caller.** Mode-B pulls emit `EvtSnapshotPushed` with `submitter` = the consumer address, NOT an operator. Indexers should join against `phoebe.getOperator()` to disambiguate.

## Related skills

- `/titon:phoebe-pull-fresh-price` — when you need mode B (Pyth-style update + read).
- `/titon:phoebe-handle-event` — index `EvtPricePulled` + `EvtSnapshotPushed` to track your dapp's pulls.
- `/titon:phoebe-debug-revert` — diagnose a failed pull by exit code.
- `/titon:phoebe-deploy-and-admit` — stand up Phoebe in your workspace if there's no live deploy yet.
- `/titon:kronos-register-job` — if you want a recurring trigger that calls into your consumer.
