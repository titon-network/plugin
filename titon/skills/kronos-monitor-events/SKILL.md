---
name: kronos-monitor-events
description: Build an indexer / dashboard / alerting bot for Kronos registry events. Use when the user wants to track their job's lifecycle (registration, execution, funding, expiry), reconstruct job history, surface fee accruals, send Discord/Slack alerts on registry events, or otherwise consume the Kronos event stream as a dapp author.
---

# Monitor Kronos events

You're helping a dapp author consume the Kronos registry event stream. The registry emits events as **external-out messages** (`dest: addr_none`) — durable on-chain audit surface. Each body is `storeUint(opcode, 32) || fields...` per `contracts/messages.tolk`.

Prefer indexing events over polling getters: getters show current state only; events show every transition.

## Decoder cheatsheet

```ts
import { decodeEvent, tryDecodeEvent } from '@titon-network/kronos-sdk';

decodeEvent(body)      // returns KronosEvent — throws on unknown opcode
tryDecodeEvent(body)   // returns KronosEvent | null on unknown
```

Discriminated union — switch on `event.kind`:

```ts
const ev = decodeEvent(body);
switch (ev.kind) {
    case 'JobRegistered':
        await db.recordJob(ev.jobId, ev.target, ev.interval, ev.reward);
        break;
    case 'JobExecuted':
        await db.recordExecution(ev.jobId, ev.executionCount, ev.executedAt);
        break;
    case 'JobFunded':
        await db.recordFunding(ev.jobId, ev.amount, ev.newBalance);
        break;
    case 'JobExpired':
    case 'JobCancelled':
        await alerts.info(`job ${ev.jobId} retired (${ev.kind})`);
        break;
}
```

## Registry event kinds (the only ones a dapp author needs)

| Kind | Fires on | Key fields |
|---|---|---|
| `JobRegistered` | Owner registers a job | jobId, owner, target, reward, interval, expireAfter |
| `JobExecuted` | Successful execution | jobId, executionCount, reward, protocolFee, executedAt |
| `JobFunded` | Anyone tops up a job balance | jobId, amount, newBalance |
| `JobUpdated` | Owner changes job config | jobId |
| `JobPaused` / `JobResumed` | Owner toggles | jobId |
| `JobWithdrawn` | Owner pulls excess balance | jobId, amount, remainingBalance |
| `JobCancelled` | Owner cancels (refund issued) | jobId, refund |
| `JobExpired` | Housekeeping cleaned up an expired job | jobId, refund |
| `JobDustSwept` | Housekeeping zero'd out an underfunded job | jobId, amount |
| `JobHousekeepingExecuted` | A housekeeping batch ran | batchStart, cleaned, nextCursor |
| `AssignedAutomatonMissed` | Fallback executor ran an assigned automaton's job | jobId, assigned, executor |
| `ConfigUpdated` | Owner changed registry tunables | …config reply |
| `TreasuryUpdated` | Owner changed fee recipient | treasury |
| `FeesWithdrawn` | Owner drained accumulatedFees | amount, remainingAccumulated |
| `PausedChanged` | Owner toggled registry pause flag | paused |
| `UpgradeProposed` / `UpgradeCancelled` | Timelocked upgrade lifecycle | codeHash, eta |
| `CodeUpdated` | Timelocked upgrade fired | codeHash, oldCodeHash, timestamp |

## Reconstructing job state without polling

`JobRegistered` → `JobExecuted` × N → `JobFunded`/`JobUpdated` → `JobCancelled`/`JobExpired` form a complete state log. Apply them in order; you can skip `getJob(...)` getters entirely after the initial load.

```ts
const job = applyJobEvents(initialState, eventsForThisJob);
```

## Long-running watcher for one job

```ts
import { JobWatcher } from '@titon-network/kronos-sdk';

new JobWatcher(client, jobId)
    .on('executed', (ev) => console.log(`run ${ev.executionCount} at ${new Date(ev.executedAt * 1000)}`))
    .on('low-balance', (ev) => alertOps(ev.jobId, ev.remaining))
    .start();
```

## Capturing events from a sandbox tx

```ts
import { decodeEvents } from '@titon-network/kronos-sdk';

const externals = result.transactions
    .flatMap((tx) => (tx as { externals?: { body: Cell }[] }).externals ?? [])
    .map((e) => e.body);
const events = decodeEvents(externals);    // drops unknowns silently
```

## Polling vs push

Public TonCenter only supports polling. Patterns:

- **Simple:** `client.getTransactions(registry, limit=50)` every 5–10 s; dedupe by `tx.lt`.
- **Better:** ton-api-v4 + websocket subscriptions.
- **Best:** Run a TON liteclient + process blocks directly.

## Alerting recipe

```ts
import { decodeEvent } from '@titon-network/kronos-sdk';

for (const body of bodies) {
    const ev = decodeEvent(body);
    switch (ev.kind) {
        case 'JobExpired':
            await postToSlack(`Kronos: job ${ev.jobId} ran out of funds and was cleaned up`);
            break;
        case 'AssignedAutomatonMissed':
            await alerts.warn(`Kronos: job ${ev.jobId} executor=${ev.executor} not the assigned automaton`);
            break;
        case 'PausedChanged':
            if (ev.paused) await pagerDuty.alert('Kronos registry paused — investigate');
            break;
    }
}
```

> bigints in events (jobId, coins) break `JSON.stringify`. Use a replacer:
> `(_k, v) => typeof v === 'bigint' ? v.toString() : v`

## Useful CLI

```bash
npx kronos-sdk decode <hex-or-base64>   # decode a single event cell
npx kronos-sdk info <registry> --testnet  # current contract state
```

## Common pitfalls

- **Unknown opcodes** — `decodeEvent` throws; `tryDecodeEvent` returns null. Use the latter when streaming mixed sources; log nulls so you spot new events you haven't bumped the SDK to.
- **External-outs vs internal sends** — only `dest: addr_none` messages are events. Internal sends from the registry to the underlying staking pool are NOT events; don't try to decode them.
- **Ordering across blocks isn't deterministic** — use `executedAt` / `availableAt` timestamps and `executionCount` counters; don't rely on tx-hash order.
- **`JobExpired` ≠ failure** — it just means the housekeeping cursor caught a job past `expireAfter` or below one execution cost. The owner already withdrew or deliberately ran it down.

## Related skills

- `/titon:kronos-debug-exit-code` — when an event reflects a failed execution you're tracing.
- `/titon:fortuna-monitor-events` — same pattern for Fortuna's event stream; mirror the indexer shape if you're tracking both.
