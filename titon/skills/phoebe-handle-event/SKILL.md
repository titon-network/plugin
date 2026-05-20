---
name: phoebe-handle-event
description: Build an indexer / dashboard / alerting bot that consumes Phoebe events. Use when the user wants to decode Phoebe events, track snapshot history, surface mode-B pulls, monitor operator activity, build cost telemetry, or otherwise consume the Phoebe event stream as a dapp author or analytics layer.
---

# Index and react to Phoebe events

You're helping a user consume the Phoebe event stream. Phoebe emits events as **external-out messages** (`dest: addr_none`) — the durable on-chain audit surface. Prefer indexing events over polling getters: getters show current state; events show every transition.

## What Phoebe emits

12 typed events in opcode range 0x80–0x8C. Discriminated by `kind` in the SDK's `PhoebeEvent` discriminated union:

| `kind` | Fires on | Useful for |
|---|---|---|
| `SnapshotPushed` | Operator `PushSnapshot` (0x70) OR consumer mode-B `RequestPrice` (0x71) advancing cache | latency, push rate, per-submitter win counts, mode-A/B mix |
| `PricePulled` | Consumer `RequestPrice` succeeded (mode A or B) | per-consumer pull volume, per-feed pull volume, revenue |
| `OperatorMirrored` | Operator activate/deactivate via `AutomatonSync` | operator set health, churn |
| `GroupKeyCached` | Atlas `GroupKeySync` arrived | rotation epoch history |
| `Paused` / `Unpaused` | Owner emergency pause | outage windows |
| `ConfigUpdated` | Owner tunable change | governance audit trail; includes `operatorBps`, `criticalStaleness`, `maxInactivity` |
| `FeesWithdrawn` | Owner drained accumulated owner-share rewards | revenue tracking |
| `RewardClaimed` | Operator drained accrued pull-fee share | per-operator economics |
| `CodeUpgradeProposed` / `CodeUpgradeExecuted` / `CodeUpgradeCancelled` | 3-step timelocked upgrade flow | contract version lineage |

> Phoebe v4 adds `OperatorsEvicted` (0x8D) on the contract side for the permissionless rotation sweep. SDK decoder support tracks the contract revisions; if the kind isn't in your SDK version yet, `tryDecodeEvent` will return `null` for it — bump `@titon-network/phoebe-sdk` to pick it up.

## Decoder cheatsheet

```ts
import {
    decodeEvent,
    decodeEvents,
    tryDecodeEvent,
    summarizeTx,
    summarizeTxs,
} from '@titon-network/phoebe-sdk';

decodeEvent(body)         // PhoebeEvent — throws on unknown opcode
tryDecodeEvent(body)      // PhoebeEvent | null — silent on unknown
decodeEvents(bodies)      // batch — drops unknowns silently

// Higher-level: collapse a Transaction[] into [{ exitCode, events, ... }]
for (const summary of summarizeTxs(txs)) {
    for (const ev of summary.events) {
        // typed PhoebeEvent — switch on ev.kind
    }
}
```

## Discriminated-union switch

```ts
import { decodeEvent } from '@titon-network/phoebe-sdk';

const ev = decodeEvent(body);
switch (ev.kind) {
    case 'SnapshotPushed':
        // ev.timestamp, ev.root, ev.submitter, ev.groupEpoch
        await db.recordPush(ev.timestamp, ev.root, ev.submitter);
        break;
    case 'PricePulled':
        // ev.consumer, ev.queryId, ev.feedId, ev.leaf
        await db.recordPull(ev.consumer, ev.queryId, ev.feedId);
        break;
    case 'OperatorMirrored':
        await db.upsertOperator(ev.operator, ev.isActive);
        break;
    case 'GroupKeyCached':
        await db.recordEpoch(ev.groupEpoch, ev.groupPk);
        break;
    case 'ConfigUpdated':
        await db.recordConfigChange({
            pullFee:           ev.pullFee,
            operatorBps:       ev.operatorBps,
            criticalStaleness: ev.criticalStaleness,
            maxInactivity:     ev.maxInactivity,
        });
        break;
    case 'RewardClaimed':
        await db.recordRewardClaim(ev.operator, ev.amount, ev.to);
        break;
    case 'Paused':
        await alerts.page(`Phoebe paused at ${ev.timestamp}`);
        break;
    case 'CodeUpgradeProposed':
        await alerts.info(`Phoebe upgrade proposed: eta=${new Date(ev.eta * 1000).toISOString()}`);
        break;
}
```

## Distinguish operator-push from consumer mode-B advance

`SnapshotPushed` fires on **two** code paths:

1. Operator's `PushSnapshot` (0x70) — `submitter` is in your ForgeTON-mirrored operator set.
2. Consumer's `RequestPrice` with `freshUpdate` (mode B) — `submitter` is the consumer; permissionless.

To disambiguate, join `submitter` against `phoebe.getOperator()`:

```ts
const op = await phoebe.getOperator(ev.submitter);
if (op && op.isActive) {
    // operator heartbeat push
} else {
    // consumer-driven mode-B advance — they paid the BLS-verify gas to land a fresh root
}
```

Cache `getOperator()` results aggressively — operator-set churn is signaled by `OperatorMirrored`.

## Cross-correlate the pull lifecycle

A successful pull produces one `EvtPricePulled` at Phoebe + (if you also index the consumer's address) one effective state change there. Sequence to expect:

```
EvtSnapshotPushed   { submitter: operator, timestamp, root }     ← operator heartbeat
  ↓ (some time passes, consumer triggers a pull)
EvtPricePulled      { consumer, queryId, feedId, leaf }          ← Phoebe verified + forwarded
  ↓ (consumer's handleFulfillPrice fires)
<your consumer's domain event>                                   ← what your dapp logs
```

For mode-B pulls the sequence is:

```
EvtSnapshotPushed   { submitter: consumer, timestamp, root }    ← cache advanced by consumer
EvtPricePulled      { consumer, queryId, feedId, leaf }          ← same tx
```

Both events fire in the same tx — group them by `(tx.hash, tx.lt)`.

## Schema-drift guard at startup

```ts
import { Phoebe, PHOEBE_STORAGE_VERSION, PHOEBE_TESTNET } from '@titon-network/phoebe-sdk';

const phoebe = tonClient.open(Phoebe.createFromAddress(PHOEBE_TESTNET!.phoebe));
const live   = await phoebe.getSchemaVersions();
if (live.storage !== PHOEBE_STORAGE_VERSION) {
    throw new Error(`SDK/contract schema mismatch: live=${live.storage} SDK=${PHOEBE_STORAGE_VERSION}`);
}
```

Surface as a "rebuild the SDK" alert rather than dropping records — Phoebe upgrades bump the storage version when wire shapes shift.

## Alerting recipe

```ts
import { tryDecodeEvent } from '@titon-network/phoebe-sdk';

for (const body of externalOutBodies) {
    const ev = tryDecodeEvent(body);
    if (!ev) continue;    // unknown opcode → bump SDK
    switch (ev.kind) {
        case 'Paused':
            await pagerDuty.alert('Phoebe paused — investigate');
            break;
        case 'CodeUpgradeProposed':
            await alerts.info(`Phoebe upgrade proposed eta=${ev.eta}`);
            break;
        case 'CodeUpgradeExecuted':
            await alerts.warn('Phoebe code upgraded — verify SDK compatibility');
            break;
    }
}
```

## Heartbeat-stall detection (the most useful alert)

Phoebe's value collapses if operators stop pushing. Track time-since-last `SnapshotPushed`:

```ts
const lastPush = await db.getMostRecentSnapshotPushedTimestamp();
const stale    = Math.floor(Date.now() / 1000) - lastPush;
if (stale > 120) {
    await pagerDuty.alert(`Phoebe heartbeat stalled — ${stale}s since last push`);
}
```

For finer-grained per-operator silence, watch the `submitter` distribution; an operator silent for `maxInactivity` will be skipped in the pull-fee split (mode-A pulls) and is a candidate for permissionless eviction.

## Polling vs push

Public TonCenter only supports polling. Patterns ranked by reliability:

- **Simple:** `client.getTransactions(phoebeAddr, limit=50)` every 5–10s. Dedupe by `tx.lt`.
- **Better:** ton-api-v4 + websocket subscriptions.
- **Best:** Run a TON liteclient + process blocks directly.

Always backfill from the deploy `lt` to current — early `GroupKeyCached` + `OperatorMirrored` events define the schema lineage.

## Useful CLI

```bash
npx phoebe decode <hex-or-base64>          # decode a single event cell
npx phoebe info <phoebe-addr> --testnet    # current contract state
npx phoebe verify --testnet                # SDK vs live-deploy drift check
```

## Common pitfalls

- **Unknown opcodes** — `decodeEvent` throws; `tryDecodeEvent` returns null. Use the latter when streaming live data; log nulls so you spot new events you haven't bumped the SDK to.
- **bigints break `JSON.stringify`.** `queryId`, `root`, `mantissa`, `amount` are bigints. Use a replacer:
  ```ts
  JSON.stringify(ev, (_k, v) => typeof v === 'bigint' ? v.toString() : v)
  ```
- **External-outs vs internal sends** — only `dest: addr_none` messages are events. Internal sends (e.g. `SyncRequest` outbound to Atlas) are NOT events; don't try to decode them.
- **Tx ordering across blocks** — use event timestamps (`ev.timestamp`, `ev.pushedAt`) not tx-hash order.
- **Cause codes on `OperatorMirrored`** — `cause` field distinguishes `CAUSE_FIRST_SYNC` (initial fan-out) from `CAUSE_ACTIVATION_CHANGE` (state transition). Use it to filter "real" activity vs bootstrap noise.

## Related skills

- `/titon:phoebe-debug-revert` — when an event reflects a failure you're tracing.
- `/titon:phoebe-integrate-consumer` — write the consumer that emits the pulls you're tracking.
- `/titon:fortuna-monitor-events` — same indexer shape for Fortuna events; mirror it if you're tracking both.
- `/titon:kronos-monitor-events` — same shape again for Kronos's stream.
