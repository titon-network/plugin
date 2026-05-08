---
name: fortuna-monitor-events
description: Build an indexer / dashboard / alerting bot that consumes Fortuna events. Use when the user wants to track VRF requests, watch fulfillments, surface stalls, build cost telemetry, or otherwise consume the Fortuna event stream as a dapp author or analytics layer.
---

# Monitor Fortuna events

You're helping a user consume the Fortuna event stream. Fortuna emits events as **external-out messages** (`dest: addr_none`) — durable on-chain audit surface. Prefer indexing events over polling getters: getters show current state only; events show every transition.

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

> **Fortuna doesn't slash.** D-011: positive-incentive economics only. There's no slash event in the stream. Operators that miss deadlines just don't earn `submitterReward`; consumers reclaim for refund.

## Decoder cheatsheet

```ts
import { decodeEvent, decodeEvents, tryDecodeEvent, summarizeTxs } from '@titon-network/fortuna-sdk';

decodeEvent(body)         // FortunaEvent — throws on unknown opcode
tryDecodeEvent(body)      // FortunaEvent | null — silent on unknown
decodeEvents(bodies)      // batch — drops unknowns silently

// Higher-level: collapse a Transaction[] into [{ exitCode, events, ... }]
for (const summary of summarizeTxs(txs)) {
    for (const ev of summary.events) {
        // typed FortunaEvent — switch on ev.kind
    }
}
```

## Discriminated-union switch

```ts
const ev = decodeEvent(body);
switch (ev.kind) {
    case 'RequestCreated':
        await db.recordRequest(ev.queryId, ev.consumer, ev.deadline, ev.alpha);
        break;
    case 'RequestFulfilled':
        await db.recordFulfillment(ev.queryId, ev.beta, ev.submitter);
        break;
    case 'RequestReclaimed':
        await alerts.warn(`reclaimed queryId=${ev.queryId} reason=${ev.reason}`);
        break;
    case 'OperatorMirrored':
        await db.upsertOperator(ev.operator, ev.isActive);
        break;
    case 'GroupKeyCached':
        await db.recordEpoch(ev.groupEpoch, ev.groupPk);
        break;
    case 'Paused':
        await alerts.page(`Fortuna paused at ${ev.timestamp}`);
        break;
}
```

## Cross-correlate the request lifecycle

Every successful VRF round produces a 2-event chain:

```
RequestCreated   { queryId, consumer, deadline, alpha, … }
       ↓ (operators sign + submit)
RequestFulfilled { queryId, beta, submitter, … }
```

If no fulfillment lands by deadline:

```
RequestCreated   { queryId, consumer, deadline, … }
       ↓ (TTL expires)
RequestReclaimed { queryId, reason: 'deadline' | 'owner', … }
```

Join by `(consumer, queryId)` — the canonical request key. After an Atlas rotation, in-flight requests abort with `E_STALE_EPOCH`; their reclaim reason is `'deadline'`.

## Schema-drift guard at startup

Events inherit the contract's wire layout. After a Fortuna upgrade, field widths may change. Assert the SDK's expected schema version once:

```ts
import { Fortuna, FORTUNA_STORAGE_VERSION, FORTUNA_TESTNET } from '@titon-network/fortuna-sdk';

const fortuna = tonClient.open(Fortuna.createFromAddress(FORTUNA_TESTNET!.fortuna));
const live = await fortuna.getSchemaVersions();
if (live.storage !== FORTUNA_STORAGE_VERSION) {
    throw new Error(`SDK/contract schema mismatch: live=${live.storage} SDK=${FORTUNA_STORAGE_VERSION}`);
}
```

The SDK getters throw `SchemaDriftError` on parse mismatch — surface as a "rebuild the SDK" alert rather than dropping records.

## Polling vs push

Public TonCenter only supports polling. Patterns ranked by reliability:

- **Simple:** `client.getTransactions(fortuna, limit=50)` every 5–10 s. Dedupe by `tx.lt`.
- **Better:** ton-api-v4 + websocket subscriptions.
- **Best:** Run a TON liteclient + process blocks directly.

Always backfill from the deploy lt to current — early events define the schema lineage.

## Alerting recipe

```ts
import { decodeEvent } from '@titon-network/fortuna-sdk';

for (const body of externalOutBodies) {
    const ev = decodeEvent(body);
    switch (ev.kind) {
        case 'RequestReclaimed':
            await postToSlack(`Fortuna stall: queryId=${ev.queryId} reason=${ev.reason}`);
            break;
        case 'Paused':
            await pagerDuty.alert('Fortuna paused — investigate');
            break;
        case 'CodeUpgradeProposed':
            await alerts.info(`Fortuna upgrade proposed: eta=${new Date(ev.eta * 1000).toISOString()}`);
            break;
    }
}
```

> **Tip:** bigints in events (queryId, beta, coins) break `JSON.stringify`. Use a replacer:
> ```ts
> JSON.stringify(ev, (_k, v) => typeof v === 'bigint' ? v.toString() : v)
> ```

## Useful CLI

```bash
npx fortuna decode <hex-or-base64>      # decode a single event cell
npx fortuna info <fortuna-addr> --testnet   # current contract state
npx fortuna verify --testnet            # SDK vs live-deploy drift check
```

## Common pitfalls

- **Unknown opcodes** — `decodeEvent` throws; `tryDecodeEvent` returns null. Use the latter when streaming mixed sources; log nulls so you spot new events you haven't bumped the SDK to.
- **External-outs vs internal sends** — only `dest: addr_none` messages are events. Internal sends (`SyncRequest` outbound to atlas, etc.) are NOT events; don't try to decode them.
- **Tx ordering across blocks** — use event timestamps (`ev.timestamp`, `ev.deadline`) not tx-hash order.
- **Don't watch for `Slash` events** — Fortuna doesn't slash (D-011). If you see code in tutorials referencing slash mechanics for Fortuna, it's stale.

## Related skills

- `/titon:fortuna-debug-exit-code` — when an event reflects a failure you're tracing.
- `/titon:kronos-monitor-events` — same pattern for Kronos's event stream; mirror the indexer shape if you're tracking both products.
