---
name: fortuna-monitor-events-python
description: Build an indexer / dashboard / alerting bot that consumes Fortuna events from Python. Use when the user wants to track VRF requests, watch fulfillments, surface stalls, build cost telemetry, or otherwise consume the Fortuna event stream from a Python application.
---

# Monitor Fortuna events (Python)

You're helping a user consume the Fortuna event stream from Python via `titon-network-fortuna-sdk`. Fortuna emits events as **external-out messages** (`dest: addr_none`) — durable on-chain audit surface. Prefer indexing events over polling getters: getters show current state only; events show every transition.

## What Fortuna emits

12 typed events in opcode range 0x60–0x6D. Discriminated by `kind` in the SDK's `FortunaEvent` union:

| `kind` | Fires on | Useful for |
|---|---|---|
| `RequestCreated` | `RequestRandomness` lands | queue length, request rate, deadline visibility |
| `RequestFulfilled` | First valid `FulfillRandomness` | throughput, per-submitter win counts, latency |
| `RequestReclaimed` | TTL expired + consumer reclaimed (or owner pruned) | stall rate, consumer-side cost telemetry |
| `OperatorMirrored` | Operator activate/deactivate via `AutomatonSync` | operator set health |
| `OperatorOptedIn` / `OperatorOptedOut` | Operator product-level participation toggle | per-product participation |
| `GroupKeyCached` | Atlas `GroupKeySync` arrived | rotation epoch history |
| `Paused` / `Unpaused` | Owner-triggered emergency pause | outage windows |
| `ConfigUpdated` | Owner tunable change | governance audit trail |
| `FeesWithdrawn` | Owner withdrew accumulated fees | revenue tracking |
| `CodeUpgradeProposed` / `CodeUpgradeExecuted` / `CodeUpgradeCancelled` | 3-step timelocked upgrade flow | contract version lineage |

> **Fortuna doesn't slash.** D-011: positive-incentive economics only. There's no slash event in the stream. Operators that miss deadlines just don't earn `submitter_reward`; consumers reclaim for refund.

## Decoder cheatsheet

```python
from fortuna_sdk import decode_event, decode_events, try_decode_event, summarize_tx

decode_event(body)        # FortunaEvent — raises on unknown opcode
try_decode_event(body)    # FortunaEvent | None — silent on unknown
decode_events(bodies)     # batch — drops unknowns silently

# Higher-level: collapse a list of pytoniq Transactions into structured summaries
for summary in summarize_txs(txs):
    for ev in summary.events:
        # typed FortunaEvent — match on ev.kind
        pass
```

## Discriminated-union match

```python
ev = decode_event(body)
match ev.kind:
    case "RequestCreated":
        await db.record_request(ev.query_id, ev.consumer, ev.deadline, ev.alpha)
    case "RequestFulfilled":
        await db.record_fulfillment(ev.query_id, ev.beta, ev.submitter)
    case "RequestReclaimed":
        await alerts.warn(f"reclaimed query_id={ev.query_id} reason={ev.reason}")
    case "OperatorMirrored":
        await db.upsert_operator(ev.operator, ev.is_active)
    case "GroupKeyCached":
        await db.record_epoch(ev.group_epoch, ev.group_pk)
    case "Paused":
        await alerts.page(f"Fortuna paused at {ev.timestamp}")
```

## Cross-correlate the request lifecycle

Every successful VRF round produces a 2-event chain:

```
RequestCreated   { query_id, consumer, deadline, alpha, … }
       ↓ (operators sign + submit)
RequestFulfilled { query_id, beta, submitter, … }
```

If no fulfillment lands by deadline:

```
RequestCreated   { query_id, consumer, deadline, … }
       ↓ (TTL expires)
RequestReclaimed { query_id, reason: 'deadline' | 'owner', … }
```

Join by `(consumer, query_id)` — the canonical request key. After an Atlas rotation, in-flight requests abort with `E_STALE_EPOCH`; their reclaim reason is `'deadline'`.

## Schema-drift guard at startup

Events inherit the contract's wire layout. After a Fortuna upgrade, field widths may change. Assert the SDK's expected schema once:

```python
from fortuna_sdk import (
    Fortuna,
    FORTUNA_STORAGE_VERSION,
    FORTUNA_TESTNET,
    assert_deployment,
)
from pytoniq import LiteBalancer


async def main():
    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()

    # One-shot: confirm SDK's expected schema matches the pinned testnet snapshot
    dep = assert_deployment("testnet")

    # Live check: query the contract's actual schema_versions
    fortuna = Fortuna.create_from_address(dep.fortuna, client=rpc)
    live = await fortuna.get_schema_versions()
    if live.storage != FORTUNA_STORAGE_VERSION:
        raise RuntimeError(
            f"SDK/contract schema mismatch: live={live.storage} SDK={FORTUNA_STORAGE_VERSION}"
        )
```

The SDK getters raise `SchemaDriftError` on parse mismatch — surface as a "rebuild the SDK" alert rather than dropping records.

## Polling pattern

```python
import asyncio
from fortuna_sdk import decode_events, FORTUNA_TESTNET
from pytoniq import LiteBalancer


async def event_loop():
    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()

    last_lt = 0
    while True:
        txs = await rpc.get_transactions(FORTUNA_TESTNET.fortuna, count=50)
        new_externals = []
        for tx in txs:
            if tx.lt <= last_lt:
                continue
            for out in tx.out_msgs:
                if out.is_external_out:     # canonical pytoniq idiom
                    new_externals.append(out.body)
            last_lt = max(last_lt, tx.lt)

        for ev in decode_events(new_externals):
            await handle_event(ev)

        await asyncio.sleep(8)
```

Always backfill from the deploy lt to current — early events define the schema lineage.

## Alerting recipe

```python
from fortuna_sdk import decode_event

for body in external_out_bodies:
    ev = decode_event(body)
    match ev.kind:
        case "RequestReclaimed":
            await post_to_slack(f"Fortuna stall: query_id={ev.query_id} reason={ev.reason}")
        case "Paused":
            await pager_duty.alert("Fortuna paused — investigate")
        case "CodeUpgradeProposed":
            from datetime import datetime, timezone
            eta = datetime.fromtimestamp(ev.eta, tz=timezone.utc).isoformat()
            await alerts.info(f"Fortuna upgrade proposed: eta={eta}")
```

## JSON serialization

Python `int` handles bigints natively, so unlike JS there's no JSON serialization gotcha. But `bytes` and `Address` need conversion:

```python
import json
from dataclasses import asdict
from pytoniq_core import Address

def default(o):
    if isinstance(o, bytes):
        return o.hex()
    if isinstance(o, Address):
        return str(o)
    raise TypeError

json.dumps(asdict(ev), default=default)
```

## Common pitfalls

- **Unknown opcodes** — `decode_event` raises; `try_decode_event` returns `None`. Use the latter when streaming mixed sources; log nones so you spot new events you haven't bumped the SDK to.
- **External-outs vs internal sends** — only `dest=None` messages are events. Internal sends (`SyncRequest` outbound to atlas, etc.) are NOT events; don't try to decode them.
- **Tx ordering across blocks** — use event timestamps (`ev.timestamp`, `ev.deadline`) not tx-hash order.
- **Don't watch for `Slash` events** — Fortuna doesn't slash (D-011). If you see code in tutorials referencing slash mechanics for Fortuna, it's stale.
- **CLI tools are TS-only.** `npx fortuna decode <hex>` exists; the Python SDK doesn't ship a CLI yet. For one-off cell decoding from a shell:
  ```bash
  python3 -c "from fortuna_sdk import decode_event; from pytoniq_core import Cell; print(decode_event(Cell.one_from_boc(bytes.fromhex('...'))))"
  ```

## Related skills

- `/titon:fortuna-debug-exit-code-python` — when an event reflects a failure you're tracing.
- `/titon:kronos-monitor-events-python` — same pattern for Kronos's event stream; mirror the indexer shape if you're tracking both products.
