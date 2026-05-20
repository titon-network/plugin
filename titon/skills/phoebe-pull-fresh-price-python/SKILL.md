---
name: phoebe-pull-fresh-price-python
description: Python equivalent of phoebe-pull-fresh-price — pull a sub-heartbeat-fresh price from Phoebe via mode B (Pyth-style update + read) from a Python service. Use when the user says "from Python", "in pytoniq", "snake_case", "I'm using Python", "Pyth-style update from Python", "attach a fresh signed snapshot in Python", "my liquidations need sub-heartbeat freshness", or otherwise wants the Python flavour of mode B. The Tolk consumer receiver is identical regardless of language. For the TS / @titon-network/phoebe-sdk side use `phoebe-pull-fresh-price`.
---

# Pull a fresh price from Phoebe (mode B, Python)

You're helping a dapp author bypass Phoebe's ~30s heartbeat by
attaching a freshly-signed snapshot to the pull, from a Python
service. Phoebe BLS-verifies the fresh aggregate under Atlas's cached
`groupPk`, advances its own cache atomically, and walks your Merkle
proof against the **fresh** root. Permissionless caller — the BLS
aggregate is the auth.

> **Tolk receiver code is identical regardless of language.** Mode B
> only changes the off-chain trigger (the consumer still receives the
> same `FulfillPrice` callback). See `phoebe-integrate-consumer-python`
> for the receiver skeleton.

## Install

```bash
pip install titon-network-phoebe-sdk pytoniq aiohttp
```

## When to use mode B (not mode A)

| Symptom | Pick |
|---|---|
| Routine mark / display / interest-accrual / non-safety read | mode A (cached, ~18k gas) |
| Liquidation, settlement, perp close, options exercise, oracle-divergence check | mode B (fresh, ~73k gas) |
| "The cache could be 28s old and I'll lose money if so" | mode B |
| "I want the freshest possible price even at 4× the gas cost" | mode B |

**Cost callout:** ~73k gas vs ~18k for cached. Don't pay for
freshness you don't need.

## The recipe

```python
import asyncio
import os
import time
from pytoniq import LiteBalancer, WalletV5R1
from phoebe_sdk import (
    Phoebe,
    PhoebeMerkleTree,
    PHOEBE_TESTNET,
    QueryIdStream,
    compute_snapshot_hash,
    estimate_pull_value,
)


async def pull_fresh(feed_id: int):
    client = LiteBalancer.from_testnet_config(trust_level=2)
    await client.start_up()
    try:
        phoebe = Phoebe.create_from_address(PHOEBE_TESTNET.phoebe, client=client)
        cfg = await phoebe.get_config()

        # 1. Check cache age first. If cache is fresh enough, drop down to mode A.
        snapshot = await phoebe.get_snapshot()
        last_snapshot_time = snapshot.last_snapshot_time
        cache_age = int(time.time()) - last_snapshot_time

        if cache_age < 5:
            return await pull_cached(feed_id, cfg)    # mode A is cheaper; cache is fresher than any fresh-update can be

        # 2. Fetch the operator's currently-signed snapshot from their endpoint.
        #    The endpoint should return { timestamp, root, sig, leaves } — the same
        #    payload the operator BLS-signed for the most recent push.
        fresh = await operator_client.get_signed_snapshot()  # your helper

        # 3. Defensive: fresh.timestamp MUST be strictly newer than the cache,
        #    else Phoebe reverts E_FRESH_NOT_FRESHER (187) and you eat the gas.
        if fresh["timestamp"] <= last_snapshot_time:
            return await pull_cached(feed_id, cfg)    # operator endpoint is lagging; fall back

        # 4. Build the proof against the FRESH root (not the cached one).
        fresh_tree = PhoebeMerkleTree.from_sparse_leaves(fresh["leaves"])
        leaf = fresh["leaves"][feed_id]

        # 5. Build the signed-data blob for the fresh update. Phoebe expects
        #    the canonical hash domain — `compute_snapshot_hash` returns the
        #    68-byte blob (phoebeHash | timestamp | root) that operators sign.
        signed_data = compute_snapshot_hash(
            phoebe.address,
            fresh["timestamp"],
            fresh["root"],
        )
        sig = fresh["sig"]                                 # 96-byte BLS-G2 aggregate

        # Load the consumer wallet.
        mnemonic = os.environ["CONSUMER_MNEMONIC"].split()
        consumer_wallet = await WalletV5R1.from_mnemonic(client, mnemonic)

        # 6. Send the pull with `fresh_update` attached + `with_fresh=True` value.
        stream = QueryIdStream()
        await phoebe.send_request_price(
            consumer_wallet,
            value=estimate_pull_value(pull_fee=cfg.pull_fee, with_fresh=True),
            query_id=stream.next(),
            feed_id=feed_id,
            proof=fresh_tree.proof(feed_id),
            leaf=leaf,
            max_staleness=0,
            fresh_update=(signed_data, sig),
        )
    finally:
        await client.close_all()


asyncio.run(pull_fresh(feed_id=42))
```

## What the operator endpoint serves

A mode-B pull needs the **exact** BLS-signed payload the operator
already produced for their most recent push. The endpoint shape
(operator-defined, but conventionally):

```python
# Type sketch — actual transport is JSON over HTTP.
SignedSnapshot = {
    "timestamp": int,                          # unix seconds — the push's timestamp
    "root":      int,                          # 256-bit Merkle root (decimal-string in JSON)
    "sig":       bytes,                        # 96-byte BLS-G2 aggregate signature
    "leaves":    dict[int, PriceLeaf],         # feed_id → PriceLeaf used to build the tree
}
```

The endpoint typically caches the most recent push and serves it via
HTTP. Operators publish the URL in their config (see
`/titon:phoebe-deploy-and-admit-python`).

> **Off-chain BLS sanity check.** If you maintain the signing infra
> yourself, the canonical hash domain is
> `compute_snapshot_hash(phoebe_address, timestamp, root)` and the
> signing call is `sign_message(secret_key, hash_bytes)` — both from
> `phoebe_sdk`. Drift in this domain produces 161
> `InvalidBlsSignature` on-chain.

## Defensive flow that ALWAYS works

```python
import time


async def pull_fresh_or_fallback(phoebe: Phoebe, feed_id: int):
    cfg = await phoebe.get_config()
    snapshot = await phoebe.get_snapshot()
    fresh = await operator_client.get_signed_snapshot()

    # If fresh isn't actually fresher, fall back to cached path.
    if fresh["timestamp"] <= snapshot.last_snapshot_time:
        return await pull_cached(feed_id, cfg)

    fresh_tree = PhoebeMerkleTree.from_sparse_leaves(fresh["leaves"])
    signed_data = compute_snapshot_hash(phoebe.address, fresh["timestamp"], fresh["root"])
    stream = QueryIdStream()

    await phoebe.send_request_price(
        consumer_wallet,
        value=estimate_pull_value(pull_fee=cfg.pull_fee, with_fresh=True),
        query_id=stream.next(),
        feed_id=feed_id,
        proof=fresh_tree.proof(feed_id),
        leaf=fresh["leaves"][feed_id],
        max_staleness=0,
        fresh_update=(signed_data, fresh["sig"]),
    )
```

## Gas + value math

`estimate_pull_value` factors in:

- `pull_fee` (retained by Phoebe — fee for the read).
- `MIN_GAS_FOR_PULL` (compute reserve at Phoebe for proof walk + callback forward).
- `+0.05 TON` extra when `with_fresh=True` (covers the `BLS_VERIFY` budget).

```python
# Mode A (cached):
value = estimate_pull_value(pull_fee=cfg.pull_fee)

# Mode B (fresh):
value = estimate_pull_value(pull_fee=cfg.pull_fee, with_fresh=True)
```

Hardcoded fallback if you can't read `cfg.pull_fee` at call time:
`150_000_000` (0.15 TON) covers mode B at default tunables with ample
slack.

## Positive externality

When your mode-B pull succeeds, Phoebe writes the fresh root +
timestamp into its own cache **atomically**. Subsequent same-block
(and beyond) pulls — yours or anyone else's — read your advanced
cache for free via mode A. You paid the BLS-verify gas; the rest of
the system gets the freshness.

## Common pitfalls

- **`E_FRESH_NOT_FRESHER` (187).** `fresh.timestamp <= last_snapshot_time` at the moment Phoebe processed your tx. Either: (a) your operator endpoint is lagging; (b) another consumer's mode-B pull beat yours into the same block; (c) an operator's heartbeat push beat you. Always re-check `await phoebe.get_snapshot()` right before sending, and fall back to mode A when fresh isn't fresher.
- **`E_INVALID_BLS_SIGNATURE` (161).** Three causes: (1) wrong DST off-chain — use `sign_message` from `phoebe_sdk`, which pins `BLS_DST_G2_POP`; (2) hash domain drift — `compute_snapshot_hash` must be called with the EXACT (phoebe address, timestamp, root) the operator signed; (3) stale `pk_share` from a pre-rotation Atlas epoch.
- **`E_WRONG_DEPLOYMENT_BINDING` (144).** Fresh-update's signed `phoebeHash` doesn't match this Phoebe's address-hash. Operator endpoint is signing for a different Phoebe deployment (mis-config) or someone is replaying a sig from a sibling Phoebe sharing the same Atlas group. Re-fetch from the right operator.
- **`E_BAD_MERKLE_PROOF` (183).** Build the proof against the FRESH root, NOT the cached `last_root`. Mode-B verifies the proof against the just-verified fresh root.
- **Don't omit the freshness check.** If you blindly send mode B every time, you'll burn 4× the gas of mode A on calls where the cache was already fresh enough.
- **Don't forget bounce-safety.** The callback still arrives `bounce: false`; keep `handleFulfillPrice` cheap.
- **Sig type.** `fresh_update` is a `(signed_data: bytes, sig: bytes)` tuple — exactly 68 + 96 bytes. The SDK's `build_request_price_body` asserts the sizes; truncated or padded blobs revert at the wire layer before the BLS verify.

## Inspecting after the fact

If your mode-B pull lands but you want to confirm the cache advanced
as expected:

```python
snapshot_before = await phoebe.get_snapshot()
# ... send mode B ...
await asyncio.sleep(8)
snapshot_after = await phoebe.get_snapshot()
assert snapshot_after.last_snapshot_time > snapshot_before.last_snapshot_time
assert snapshot_after.last_root == fresh["root"]
```

For the event-stream view (which fires `SnapshotPushed` with
`submitter` = your consumer, NOT an operator), see
`phoebe-handle-event-python`.

## Related skills

- `/titon:phoebe-integrate-consumer-python` — write the Tolk consumer + receiver. Start here if you haven't yet.
- `/titon:phoebe-debug-revert-python` — diagnose 161 / 187 / 183 / 144 when a mode-B pull reverts.
- `/titon:phoebe-handle-event-python` — `SnapshotPushed` fires on mode-B pulls (with `submitter` = consumer); useful for indexers.
- `/titon:phoebe-pull-fresh-price` — TS sibling with the same defensive flow.
