---
name: phoebe-integrate-consumer-python
description: Python equivalent of phoebe-integrate-consumer — pull verified prices from Phoebe (TON's threshold-BLS price oracle) into a Tolk consumer contract, with a Python off-chain driver to trigger the pull. Use when the user says "from Python", "in pytoniq", "snake_case", "I'm using Python", "integrate Phoebe in a Python backend / service / script", "drive RequestPrice from Python", "build a price-aware vault with Python tooling", or otherwise wants the Python flavour of the TS integration recipe. The Tolk receiver code is identical regardless of language. For the TS / sandbox-tested flavour use `phoebe-integrate-consumer`.
---

# Integrate a Phoebe price-oracle consumer (Python)

You're helping a dapp author add verified on-chain price reads with
`titon-network-phoebe-sdk` (PyPI). Phoebe's consumer surface is
intentionally tiny: send `RequestPrice` (0x71) → receive `FulfillPrice`
(0x72). One `BLS_VERIFY` on push + one cell-merkle walk per pull, any
of 256 feeds from a single signed root.

> **Tolk receiver code is identical regardless of language.** Only the
> off-chain driver (deploy, trigger, test) differs. For the Tolk
> skeleton see this file; for full TS sandbox testing patterns see the
> `phoebe-integrate-consumer` sibling.

Two pull modes — pick by freshness need:

- **Mode A — cached fast-path** (~18k gas, default). Phoebe walks your Merkle proof against the cached `lastRoot`. No BLS verify. Use for routine reads.
- **Mode B — Pyth-style update + read** (~73k gas). Attach a fresh signed snapshot inline; Phoebe verifies + advances cache + walks your proof against the fresh root. Permissionless caller (BLS is the auth). Use when sub-heartbeat freshness matters — liquidations, settlement, oracle-divergence checks.

## Install

```bash
pip install titon-network-phoebe-sdk pytoniq
# or with poetry / uv:
poetry add titon-network-phoebe-sdk pytoniq
```

The PyPI name uses dashes; the import path uses underscores
(`from phoebe_sdk import ...`).

## Wire types — copy verbatim from the SDK source

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

**Tolk receiver code is identical to the TS path** — Tolk is
language-neutral. See `phoebe-integrate-consumer` for the same Tolk
skeleton; the only difference is the driver below.

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

## Python driver (off-chain trigger)

```python
import asyncio
import os
from pytoniq import LiteBalancer, WalletV5R1
from phoebe_sdk import (
    Phoebe,
    PhoebeMerkleTree,
    PHOEBE_TESTNET,
    QueryIdStream,
    assert_deployment,
    estimate_pull_value,
)


async def trigger_pull(feed_id: int, latest_leaves: dict):
    client = LiteBalancer.from_testnet_config(trust_level=2)
    await client.start_up()
    try:
        dep = assert_deployment("testnet")
        phoebe = Phoebe.create_from_address(dep.phoebe, client=client)
        cfg = await phoebe.get_config()

        # Off-chain: build a tree from the latest signed leaves (cache + indexer).
        tree = PhoebeMerkleTree.from_sparse_leaves(latest_leaves)
        leaf = latest_leaves[feed_id]

        # Load the consumer wallet from mnemonic.
        mnemonic = os.environ["CONSUMER_MNEMONIC"].split()
        consumer_wallet = await WalletV5R1.from_mnemonic(client, mnemonic)

        stream = QueryIdStream()
        await phoebe.send_request_price(
            consumer_wallet,
            value=estimate_pull_value(pull_fee=cfg.pull_fee),  # mode A
            query_id=stream.next(),
            feed_id=feed_id,
            proof=tree.proof(feed_id),
            leaf=leaf,
            max_staleness=0,                                    # accept any active root
        )
    finally:
        await client.close_all()
```

Phoebe will deliver `FulfillPrice` to your consumer contract on
inclusion; the consumer's Tolk receiver above runs and your domain
logic fires.

## The 5 rules every consumer follows

1. **Pin Phoebe at deploy.** `storage.phoebe: address` set once. Repointing requires a contract upgrade.
2. **Sender-gate the callback.** `assert (sender == storage.phoebe) throw E_NOT_PHOEBE;` at the top of `handleFulfillPrice`. Without this, anyone can deliver fake prices by sending a 0x72 body.
3. **Unique queryId per pending request.** Use a monotone counter on-chain or `QueryIdStream` from the SDK off-chain. (Phoebe doesn't track per-query state, but your callback dispatcher needs it.)
4. **Funded request value.** Floor = `cfg.pull_fee + ~0.03 TON` (mode A) or `+ ~0.05 TON` (mode B). Use `estimate_pull_value(pull_fee=..., with_fresh=...)` — don't hardcode.
5. **Revert-proof callback.** Phoebe sends `FulfillPrice` with `bounce: false` — a revert in your handler does NOT bounce back. Keep the callback cheap and assertion-light; emit an event and process heavy work in a separate message.

## Building the proof off-chain (Python)

```python
from phoebe_sdk import PhoebeMerkleTree, PriceLeaf

# Sparse map of feed_id → PriceLeaf — empty slots get the canonical placeholder.
tree = PhoebeMerkleTree.from_sparse_leaves({
    42: PriceLeaf(feed_id=42, mantissa=1_950_000_000, expo=-9, conf_bps=20, pub_time=1_700_000_000),
})
proof = tree.proof(42)
assert tree.root_as_int() == ... # matches phoebe.last_root after operator pushes this set
```

In production, an off-chain indexer watches `SnapshotPushed` events +
caches the operator-signed leaves. The consumer fetches them at pull
time (often via `fetch_verified_price` — see
`phoebe-consume-price-python`). **Don't reuse a proof across
snapshots** — every root rotation invalidates prior proofs.

## Cost trade-offs at a glance

| Mode | Gas | When to use |
|---|---|---|
| A (cached) | ~18k + pullFee | Routine reads, marks, periodic mark-to-market |
| B (fresh-update) | ~73k + pullFee | Liquidations, settlement, perp closes, options exercise — anywhere stale prices break safety |

Positive externality of mode B: the cache advances atomically, so
subsequent same-block pulls read your advanced cache for free.

## Testing — Python uses live testnet, not a local sandbox

**Important gap from the TS sibling.** pytoniq has no equivalent to
`@ton/sandbox` today, so there's no in-memory "deploy a Phoebe fixture
+ push + pull" loop. Python integration tests run against live testnet
via pytest + pytoniq:

```bash
pip install titon-network-phoebe-sdk pytoniq pytest pytest-asyncio
```

```python
# tests/test_phoebe_pull.py
import os
import pytest
import pytest_asyncio
from pytoniq import LiteBalancer, WalletV5R1
from phoebe_sdk import (
    PHOEBE_TESTNET,
    Phoebe,
    QueryIdStream,
    estimate_pull_value,
    fetch_verified_price,
)


@pytest_asyncio.fixture(scope="module")
async def client():
    c = LiteBalancer.from_testnet_config(trust_level=2)
    await c.start_up()
    yield c
    await c.close_all()


@pytest_asyncio.fixture(scope="module")
async def consumer_wallet(client):
    mnemonic = os.environ["CONSUMER_MNEMONIC"].split()
    return await WalletV5R1.from_mnemonic(client, mnemonic)


@pytest.mark.asyncio
async def test_pull_lands(client, consumer_wallet):
    phoebe = Phoebe.create_from_address(PHOEBE_TESTNET.phoebe, client=client)
    quote = await fetch_verified_price(phoebe, feed_id=0, operators=PHOEBE_TESTNET.operators)
    cfg = await phoebe.get_config()
    stream = QueryIdStream()
    await phoebe.send_request_price(
        consumer_wallet,
        value=estimate_pull_value(pull_fee=cfg.pull_fee),
        query_id=stream.next(),
        feed_id=quote.feed_id,
        proof=quote.proof,
        leaf=quote.leaf,
        max_staleness=0,
    )
    # Allow ~6-10s for testnet inclusion; then walk consumer's external-outs
    # to assert FulfillPrice was delivered.
```

**For sandbox-based testing of the consumer contract** — exit-code
matrix, gas baselines, multi-operator pushes — use the TS skill
(`phoebe-integrate-consumer` with `deployPhoebeFixture`). Python tests
should be a thin smoke layer that exercises the Python driver against
live testnet, not the full revert / gas-baseline matrix.

A pragmatic split most teams adopt:

- **TS sandbox suite** — receiver correctness, exit code matrix, gas baselines (fast, deterministic, in-process)
- **Python pytest suite** — deploy + trigger + fulfill on testnet (slower, validates the Python driver path end-to-end)

## Common pitfalls

- **Don't trust an unauthenticated `FulfillPrice` body.** Sender-pin is non-negotiable.
- **Don't reuse a Merkle proof across snapshots.** Every push rotates `lastRoot`; old proofs revert with `E_BAD_MERKLE_PROOF` (183).
- **Don't pass a leaf that doesn't match the proof.** `leaf.feed_id` must equal `feed_id`; the leaf cell hash must match the proof's claimed leaf.
- **Don't hand-decode the FulfillPrice body.** Use `parse_fulfill_price(body)` — `mantissa` is `int128`, easy to get the widths wrong.
- **Don't forget mode B's permissionless caller.** Mode-B pulls emit `SnapshotPushed` with `submitter` = the consumer address, NOT an operator. Indexers should join against `phoebe.get_operator()` to disambiguate.
- **Don't pass `float` for nanoton amounts.** `pull_fee`, `value`, `mantissa` are all `int`. Use `int(0.05 * 10**9)` (or just the literal `50_000_000`) — never floats.

## Related skills

- `/titon:phoebe-pull-fresh-price-python` — when you need mode B (Pyth-style update + read) from Python.
- `/titon:phoebe-consume-price-python` — `fetch_verified_price` if you don't already have the leaves cached.
- `/titon:phoebe-handle-event-python` — index `PricePulled` + `SnapshotPushed` to track your dapp's pulls.
- `/titon:phoebe-debug-revert-python` — diagnose a failed pull by exit code.
- `/titon:phoebe-deploy-and-admit-python` — stand up Phoebe in your workspace if there's no live deploy yet.
- `/titon:phoebe-integrate-consumer` — TS sibling with full `deployPhoebeFixture` sandbox test patterns.
- `/titon:kronos-register-job-python` — if you want a recurring trigger that calls into your consumer.
