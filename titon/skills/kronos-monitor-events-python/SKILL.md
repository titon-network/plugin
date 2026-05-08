---
name: kronos-monitor-events-python
description: Build an indexer / dashboard / alerting bot for Kronos registry events from Python. Use when the user wants to track a job's lifecycle (registration, execution, funding, expiry), reconstruct job history, surface fee accruals, send Discord/Slack alerts on registry events, or otherwise consume the Kronos event stream from a Python application.
---

# Monitor Kronos events (Python)

You're helping a dapp author consume the Kronos registry event stream from Python via `titon-network-kronos-sdk`. The registry emits events as **external-out messages** (`dest: addr_none`) — durable on-chain audit surface. Each body is `store_uint(opcode, 32) || fields...`.

Prefer indexing events over polling getters: getters show current state only; events show every transition.

## Decoder cheatsheet

```python
from kronos_sdk import decode_event, decode_events

decode_event(body)        # returns KronosEvent — raises on unknown opcode
decode_events(bodies)     # returns list[KronosEvent] — drops unknowns silently
```

Discriminated union — switch on `event.kind`:

```python
ev = decode_event(body)
match ev.kind:
    case "JobRegistered":
        await db.record_job(ev.job_id, ev.target, ev.interval, ev.reward)
    case "JobExecuted":
        await db.record_execution(ev.job_id, ev.execution_count, ev.executed_at)
    case "JobFunded":
        await db.record_funding(ev.job_id, ev.amount, ev.new_balance)
    case "JobExpired" | "JobCancelled":
        await alerts.info(f"job {ev.job_id} retired ({ev.kind})")
```

## Registry event kinds (the only ones a dapp author needs)

| Kind | Fires on | Key fields (snake_case) |
|---|---|---|
| `JobRegistered` | Owner registers a job | `job_id, owner, target, reward, interval, expire_after` |
| `JobExecuted` | Successful execution | `job_id, execution_count, reward, protocol_fee, executed_at` |
| `JobFunded` | Anyone tops up a job balance | `job_id, amount, new_balance` |
| `JobUpdated` | Owner changes job config | `job_id` |
| `JobPaused` / `JobResumed` | Owner toggles | `job_id` |
| `JobWithdrawn` | Owner pulls excess balance | `job_id, amount, remaining_balance` |
| `JobCancelled` | Owner cancels (refund issued) | `job_id, refund` |
| `JobExpired` | Housekeeping cleaned up an expired job | `job_id, refund` |
| `JobDustSwept` | Housekeeping zero'd out an underfunded job | `job_id, amount` |
| `JobHousekeepingExecuted` | A housekeeping batch ran | `batch_start, cleaned, next_cursor` |
| `AssignedAutomatonMissed` | Fallback executor ran an assigned automaton's job | `job_id, assigned, executor` |
| `ConfigUpdated` | Owner changed registry tunables | …config reply |
| `TreasuryUpdated` | Owner changed fee recipient | `treasury` |
| `FeesWithdrawn` | Owner drained accumulated fees | `amount, remaining_accumulated` |
| `PausedChanged` | Owner toggled registry pause flag | `paused` |
| `UpgradeProposed` / `UpgradeCancelled` | Timelocked upgrade lifecycle | `code_hash, eta` |
| `CodeUpdated` | Timelocked upgrade fired | `code_hash, old_code_hash, timestamp` |

## Long-running watcher for one job

The Python SDK ships `JobWatcher` — a callback-driven, single-job subscription. You provide a `source` that implements `poll_bodies()` (returns recent external-out cells); the watcher decodes, filters, dispatches to your handlers, and synthesizes a `LowBalance` event from the job's runs-remaining health:

```python
import asyncio
from kronos_sdk import (
    JobWatcher,
    JobWatcherOptions,
    KronosClient,
    KronosRegistry,
    KRONOS_TESTNET,
)
from pytoniq import LiteBalancer


class TonCenterSource:
    def __init__(self, rpc, registry_address):
        self._rpc = rpc
        self._registry = registry_address

    async def poll_bodies(self):
        txs = await self._rpc.get_transactions(self._registry, count=20)
        return [
            out.body
            for tx in txs
            for out in tx.out_msgs
            if out.is_external_out      # canonical pytoniq idiom
        ]


async def main():
    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()
    try:
        registry = KronosRegistry.create_from_address(KRONOS_TESTNET.registry, client=rpc)
        client = KronosClient(registry=registry)

        source = TonCenterSource(rpc, KRONOS_TESTNET.registry)
        watcher = JobWatcher(
            client,
            job_id,
            JobWatcherOptions(source=source, poll_seconds=10, low_balance_threshold=10),
        )
        watcher.on("JobExecuted", lambda ev: print(f"run {ev.execution_count} by {ev.automaton}"))
        watcher.on("LowBalance", lambda ev: print(f"low balance: {ev.runs_remaining} runs left"))

        stop = watcher.start()
        try:
            await asyncio.sleep(3600)   # run for an hour
        finally:
            await stop()
    finally:
        await rpc.close_all()


asyncio.run(main())
```

The polling pattern below is also fine — `JobWatcher` is just a higher-level wrapper for callback-driven dapps that don't want to write their own dispatch loop.

## Polling pattern (without JobWatcher)

```python
import asyncio
from kronos_sdk import decode_events, KRONOS_TESTNET
from pytoniq import LiteBalancer


async def poll_loop():
    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()

    last_lt = 0
    while True:
        txs = await rpc.get_transactions(KRONOS_TESTNET.registry, count=50)
        new_externals = []
        for tx in txs:
            if tx.lt <= last_lt:
                continue
            for out in tx.out_msgs:
                if out.is_external_out:     # canonical pytoniq idiom
                    new_externals.append(out.body)
            last_lt = max(last_lt, tx.lt)

        for ev in decode_events(new_externals):
            handle_event(ev)

        await asyncio.sleep(8)
```

`decode_events` drops unknown opcodes silently. To detect new event types you haven't bumped the SDK to, use `decode_event` inside a try/except instead.

## Reconstructing job state without polling

`JobRegistered` → `JobExecuted` × N → `JobFunded`/`JobUpdated` → `JobCancelled`/`JobExpired` form a complete state log. Apply them in order; you can skip `await client.jobs.get(...)` getters entirely after the initial load.

## Capturing events from a single tx (e.g. inside a test)

```python
from kronos_sdk import decode_events

# After awaiting a send_*, pull the registry's recent txs and grab externals
txs = await rpc.get_transactions(KRONOS_TESTNET.registry, count=5)
bodies = [out.body for tx in txs for out in tx.out_msgs if out.is_external_out]
events = decode_events(bodies)
```

## Alerting recipe

```python
from kronos_sdk import decode_event

for body in bodies:
    ev = decode_event(body)
    match ev.kind:
        case "JobExpired":
            await post_to_slack(f"Kronos: job {ev.job_id} ran out of funds and was cleaned up")
        case "AssignedAutomatonMissed":
            await alerts.warn(f"Kronos: job {ev.job_id} executor={ev.executor} not the assigned automaton")
        case "PausedChanged":
            if ev.paused:
                await pager_duty.alert("Kronos registry paused — investigate")
```

## JSON serialization gotcha

`int` fields that exceed JS safe range (`2^53`) cause issues if you forward to a JS frontend or a JSON column with implicit `Number` casting. In Python they're fine; just be careful at the boundary.

```python
import json

# Dataclasses serialize cleanly; for events use dataclasses.asdict()
from dataclasses import asdict
print(json.dumps(asdict(ev), default=str))   # `default=str` handles Address
```

## Polling vs push

Public TonCenter only supports polling. Patterns ranked by reliability:

- **Simple:** `rpc.get_transactions(addr, count=50)` every 5–10 s; dedupe by `tx.lt`.
- **Better:** ton-api-v4 + websocket (no native Python client; use `aiohttp` against the WS endpoint).
- **Best:** Run a TON liteclient + process blocks directly. `pytoniq.LiteClient` supports this.

## Common pitfalls

- **Unknown opcodes** — `decode_event` raises; `decode_events` drops silently. Wrap individual decodes in try/except when streaming mixed sources, so you log nulls and spot new events you haven't bumped the SDK to.
- **External-outs vs internal sends** — only `dest=None` messages are events. Internal sends from the registry to the staking pool are NOT events; don't try to decode them.
- **Ordering across blocks isn't deterministic** — use `executed_at` / `available_at` timestamps and `execution_count` counters; don't rely on tx-hash order.
- **`JobExpired` ≠ failure** — it just means housekeeping caught a job past `expire_after` or below one execution cost. The owner already withdrew or deliberately ran it down.
- **CLI tools are TS-only.** `npx kronos-sdk decode <hex>` exists; there's no `python -m kronos_sdk` equivalent yet. For one-off cell decoding from a shell, install the npm package or use `pytoniq_core.Cell.one_from_boc(bytes.fromhex(...))` + `decode_event`.

## Related skills

- `/titon:kronos-debug-exit-code-python` — when an event reflects a failed execution you're tracing.
- `/titon:fortuna-monitor-events-python` — same pattern for Fortuna's event stream; mirror the indexer shape if you're tracking both.
