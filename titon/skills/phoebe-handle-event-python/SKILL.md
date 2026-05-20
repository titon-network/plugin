---
name: phoebe-handle-event-python
description: Python equivalent of phoebe-handle-event — build an indexer / dashboard / alerting bot in Python that consumes Phoebe's event stream. Use when the user says "from Python", "in pytoniq", "snake_case", "I'm using Python", "decode Phoebe events in Python", "watch SnapshotPushed from a Python service", "Python indexer for Phoebe", "alert on heartbeat stall in Python", or otherwise wants the Python flavour of the indexing recipe. For the TS / @titon-network/phoebe-sdk side use `phoebe-handle-event`.
---

# Index and react to Phoebe events (Python)

You're helping a user consume the Phoebe event stream from a Python
service using `titon-network-phoebe-sdk`. Phoebe emits events as
**external-out messages** (`dest: addr_none`) — the durable on-chain
audit surface. Prefer indexing events over polling getters: getters
show current state; events show every transition.

## What Phoebe emits

12 typed events in opcode range 0x80–0x8E. Discriminated by `kind` in
the SDK's `PhoebeEvent` union:

| `kind` | Fires on | Useful for |
|---|---|---|
| `SnapshotPushed` | Operator `PushSnapshot` (0x70) OR consumer mode-B `RequestPrice` (0x71) advancing cache | latency, push rate, per-submitter win counts, mode-A/B mix |
| `PricePulled` | Consumer `RequestPrice` succeeded (mode A or B) | per-consumer pull volume, per-feed pull volume, revenue |
| `OperatorMirrored` | Operator activate/deactivate via `AutomatonSync` | operator set health, churn |
| `GroupKeyCached` | Atlas `GroupKeySync` arrived | rotation epoch history |
| `Paused` / `Unpaused` | Owner emergency pause | outage windows |
| `ConfigUpdated` | Owner tunable change | governance audit trail; includes `operator_bps`, `max_inactivity` |
| `FeesWithdrawn` | Owner drained accumulated owner-share rewards | revenue tracking |
| `RewardClaimed` | Operator drained accrued pull-fee share | per-operator economics |
| `CodeUpgradeProposed` / `CodeUpgradeExecuted` / `CodeUpgradeCancelled` | 3-step timelocked upgrade flow | contract version lineage |
| `OperatorPruned` | Permissionless `GcOperator` swept a deactivated operator | bookkeeping for the sparse-map prune story |
| `SyncAtlasRequested` | Owner-triggered explicit resync | rare; useful for audit timelines |

## Decoder cheatsheet

```python
from phoebe_sdk import (
    decode_event,
    decode_events,
    try_decode_event,
    summarize_tx,
    summarize_txs,
)

ev  = decode_event(body)         # PhoebeEvent — raises on unknown opcode
ev  = try_decode_event(body)     # PhoebeEvent | None — silent on unknown
evs = decode_events(bodies)      # batch — drops unknowns silently

# Higher-level: collapse a Transaction sequence into TxSummary objects
for summary in summarize_txs(txs):
    for ev in summary.events:
        # typed PhoebeEvent — pattern-match on ev.kind
        ...
```

## Match-style dispatch

```python
from phoebe_sdk import decode_event

ev = decode_event(body)
match ev.kind:
    case "SnapshotPushed":
        # ev.timestamp, ev.root, ev.submitter, ev.group_epoch
        await db.record_push(ev.timestamp, ev.root, ev.submitter)
    case "PricePulled":
        # ev.consumer, ev.query_id, ev.feed_id, ev.leaf
        await db.record_pull(ev.consumer, ev.query_id, ev.feed_id)
    case "OperatorMirrored":
        await db.upsert_operator(ev.operator, ev.is_active)
    case "GroupKeyCached":
        await db.record_epoch(ev.group_epoch, ev.group_pk)
    case "ConfigUpdated":
        await db.record_config_change(
            pull_fee=ev.pull_fee,
            operator_bps=ev.operator_bps,
            max_inactivity=ev.max_inactivity,
        )
    case "RewardClaimed":
        await db.record_reward_claim(ev.operator, ev.amount, ev.to)
    case "Paused":
        await alerts.page(f"Phoebe paused at {ev.timestamp}")
    case "CodeUpgradeProposed":
        await alerts.info(f"Phoebe upgrade proposed: eta={ev.eta}")
```

(If you're on Python < 3.10 use `if/elif` on `ev.kind` instead of `match`.)

## Distinguish operator-push from consumer mode-B advance

`SnapshotPushed` fires on **two** code paths:

1. Operator's `PushSnapshot` (0x70) — `submitter` is in your ForgeTON-mirrored operator set.
2. Consumer's `RequestPrice` with `fresh_update` (mode B) — `submitter` is the consumer; permissionless.

To disambiguate, join `submitter` against `phoebe.get_operator()`:

```python
op = await phoebe.get_operator(ev.submitter)
if op is not None and op.is_active:
    # operator heartbeat push
    ...
else:
    # consumer-driven mode-B advance — they paid the BLS-verify gas to land a fresh root
    ...
```

Cache `get_operator()` results aggressively — operator-set churn is
signaled by `OperatorMirrored`. A small in-memory dict keyed by
`Address` with TTL ~60s is plenty.

## Cross-correlate the pull lifecycle

A successful pull produces one `PricePulled` at Phoebe + (if you also
index the consumer's address) one effective state change there.
Sequence to expect:

```
SnapshotPushed   { submitter: operator, timestamp, root }     ← operator heartbeat
  ↓ (some time passes, consumer triggers a pull)
PricePulled      { consumer, query_id, feed_id, leaf }        ← Phoebe verified + forwarded
  ↓ (consumer's handleFulfillPrice fires)
<your consumer's domain event>                                ← what your dapp logs
```

For mode-B pulls the sequence is:

```
SnapshotPushed   { submitter: consumer, timestamp, root }    ← cache advanced by consumer
PricePulled      { consumer, query_id, feed_id, leaf }       ← same tx
```

Both events fire in the same tx — group them by `(tx.hash, tx.lt)`.

## Schema-drift guard at startup

```python
from phoebe_sdk import (
    PHOEBE_CONFIG_BLOB_VERSION,
    PHOEBE_STORAGE_VERSION,
    PHOEBE_TESTNET,
    Phoebe,
    SchemaDriftError,
)

phoebe = Phoebe.create_from_address(PHOEBE_TESTNET.phoebe, client=client)
try:
    live = await phoebe.get_schema_versions()
except SchemaDriftError as e:
    # SDK ships the drift detector — surface as alert + halt indexing.
    raise SystemExit(f"phoebe schema drift detected: {e}")

assert live.storage == PHOEBE_STORAGE_VERSION
assert live.config == PHOEBE_CONFIG_BLOB_VERSION
```

Surface as a "rebuild the SDK" alert rather than dropping records —
Phoebe upgrades bump the storage version when wire shapes shift.

## Alerting recipe

```python
from phoebe_sdk import try_decode_event

for body in external_out_bodies:
    ev = try_decode_event(body)
    if ev is None:
        continue                                # unknown opcode → bump SDK
    if ev.kind == "Paused":
        await pager_duty.alert("Phoebe paused — investigate")
    elif ev.kind == "CodeUpgradeProposed":
        await alerts.info(f"Phoebe upgrade proposed eta={ev.eta}")
    elif ev.kind == "CodeUpgradeExecuted":
        await alerts.warn("Phoebe code upgraded — verify SDK compatibility")
```

## Heartbeat-stall detection (the most useful alert)

Phoebe's value collapses if operators stop pushing. Track
time-since-last `SnapshotPushed`:

```python
import time

last_push = await db.get_most_recent_snapshot_pushed_timestamp()
stale = int(time.time()) - last_push
if stale > 120:
    await pager_duty.alert(f"Phoebe heartbeat stalled — {stale}s since last push")
```

For finer-grained per-operator silence, watch the `submitter`
distribution; an operator silent for `max_inactivity` will be skipped
in the pull-fee split (mode-A pulls) and is a candidate for
permissionless eviction.

## Polling vs push

pytoniq's `LiteBalancer` only supports polling against the liteservers.
Patterns ranked by reliability:

- **Simple:** `await client.get_transactions(phoebe.address, count=50)` every 5–10s. Dedupe by `tx.lt`.
- **Better:** ton-api-v4 + websocket subscriptions (requires a separate HTTP client; see `aiohttp` or `httpx`).
- **Best:** Run a TON liteclient directly and process blocks; pytoniq exposes the lower-level `LiteClient` for this.

Always backfill from the deploy `lt` to current — early
`GroupKeyCached` + `OperatorMirrored` events define the schema lineage.

## Polling loop skeleton

```python
import asyncio
from phoebe_sdk import PHOEBE_TESTNET, Phoebe, try_decode_event
from pytoniq import LiteBalancer


async def indexer():
    client = LiteBalancer.from_testnet_config(trust_level=2)
    await client.start_up()
    try:
        phoebe = Phoebe.create_from_address(PHOEBE_TESTNET.phoebe, client=client)
        seen_lt = await db.get_indexer_cursor() or 0

        while True:
            txs = await client.get_transactions(phoebe.address, count=100)
            new_txs = sorted([t for t in txs if t.lt > seen_lt], key=lambda t: t.lt)
            for tx in new_txs:
                for out in (tx.out_msgs or []):
                    if not getattr(out, "is_external_out", False):
                        continue
                    ev = try_decode_event(out.body)
                    if ev is None:
                        await db.record_unknown_event(tx.hash, out.body)
                        continue
                    await dispatch(ev, tx)
                seen_lt = tx.lt
            await db.set_indexer_cursor(seen_lt)
            await asyncio.sleep(8)
    finally:
        await client.close_all()


asyncio.run(indexer())
```

## Common pitfalls

- **Unknown opcodes** — `decode_event` raises; `try_decode_event` returns `None`. Use the latter when streaming live data; log nulls so you spot new events you haven't bumped the SDK to.
- **`int` types are fine** — unlike TS, Python ints can be arbitrary-precision. `query_id`, `root`, `mantissa`, `amount` are plain `int`. JSON serialization works without a custom encoder.
- **External-outs vs internal sends** — only `dest: addr_none` messages are events. Internal sends (e.g. `SyncRequest` outbound to Atlas) are NOT events; don't try to decode them. Filter on `msg.is_external_out`.
- **Tx ordering across blocks** — use event timestamps (`ev.timestamp`, `ev.pushed_at`) not tx-hash order.
- **Cause codes on `OperatorMirrored`** — `cause` field distinguishes `CAUSE_FIRST_SYNC` (initial fan-out) from `CAUSE_ACTIVATION_CHANGE` (state transition). Use it to filter "real" activity vs bootstrap noise.
- **Connection lifecycle** — `LiteBalancer` keeps long-lived sockets; ALWAYS `await client.close_all()` in a `finally` block, or wrap the indexer in a contextmanager helper.

## Related skills

- `/titon:phoebe-debug-revert-python` — when an event reflects a failure you're tracing.
- `/titon:phoebe-integrate-consumer-python` — write the consumer that emits the pulls you're tracking.
- `/titon:fortuna-monitor-events-python` — same indexer shape for Fortuna events; mirror it if you're tracking both.
- `/titon:kronos-monitor-events-python` — same shape again for Kronos's stream.
- `/titon:phoebe-handle-event` — TS sibling indexer recipe.
