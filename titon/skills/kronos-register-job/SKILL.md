---
name: kronos-register-job
description: Build and submit a recurring on-chain job to the Kronos registry. Use when the user wants to schedule a periodic contract call, automate a price update, set up a recurring payment, or otherwise needs a job registered with the Kronos protocol on TON.
---

# Build a Kronos recurring job

You're helping a developer register a recurring on-chain job. The Kronos registry stores the schedule; permissionless automatons execute it on-time and earn the configured reward.

## What you need from the user

| Field | What to ask |
|-------|------------|
| Target contract | The address that should be called each cycle. |
| Body cell | The opaque message body to forward (schema is target-defined). If they don't have one, suggest `beginCell().storeUint(OP, 32).endCell()` with their op. |
| Interval | "How often?" Convert to seconds. Reject < 60 (`minInterval`) or > 86_400 (`maxInterval`) by default. |
| Reward | Per-execution automaton reward. Default `toNano('0.05')` — anything below registry's `minReward` is rejected on-chain. |
| Gas budget | Forwarded to target. Default `toNano('0.02')` for simple targets; bump to `toNano('0.1')` for complex ones. Empirically size via `/titon:kronos-gas-sizing`. |

If the user is vague, suggest defaults and confirm.

## The recipe

```ts
import { Address, beginCell, toNano } from '@ton/core';
import { registerJobOpts, KronosClient, KronosRegistry, KRONOS_TESTNET } from '@titon-network/kronos-sdk';

// Live testnet address comes from the SDK — don't hardcode.
const registry = tonClient.open(KronosRegistry.createFromAddress(KRONOS_TESTNET.registry));
const client = new KronosClient({ registry });

// Pull live limits (minInterval, minReward, etc.) for auto-funding + validation.
const cfg = await client.jobs.config();

const opts = registerJobOpts(
    {
        target: targetAddress,
        message: beginCell().storeUint(0, 32).endCell(),
        interval: 3600,                        // seconds
        reward: toNano('0.05'),
        gasLimit: toNano('0.02'),
        windowBefore: 30,                      // optional
        windowAfter: 600,
        maxExecutions: 0,                      // 0 = unlimited
    },
    cfg,                                       // auto-funds 100 runs
);

const result = await registry.sendRegisterJob(walletSender, opts);

// Read the new jobId from the JobRegistered event:
const events = client.events.decodeAll(
    result.transactions
        .flatMap((tx) => tx.externals ?? [])
        .map((e) => e.body),
);
const registered = events.find((e) => e.kind === 'JobRegistered');
console.log(`new job id: ${registered.jobId}`);
```

## Funding math (so user doesn't get rejected)

- Per-execution cost = `reward + gasLimit + reward * protocolFeeBps / 10_000`.
- Min attached value to RegisterJob = at least `minFunding + minGasReserve` from registry config.
- `registerJobOpts(input, cfg)` auto-funds 100 runs; override by passing `funding: toNano('5')` in `input` for a different horizon.
- Show the user the projected burn rate: `cost-per-run * (86400/interval)` = TON per day.

## Pitfalls — call out to the user

- **Body cell is target-specific.** If the target rejects the body (TVM exit 0xFFFF or 9), the automaton still gets paid — the job just no-ops on-chain. Confirm the body matches what the target accepts.
- **Don't set `expireAfter` < `now + interval`.** The job can't run if it expires before the first window opens.
- **Once registered, only the owner can pause/update/cancel.** The wallet that signed RegisterJob is the owner.
- **First execution is permissionless.** Anyone can fire it. Subsequent runs go through assigned-automaton rotation if the registry is wired to a ForgeTON pool.
- **Size `gasLimit` empirically.** The default 0.02 TON works only for trivial receivers. Use `/titon:kronos-gas-sizing` to measure with `estimateJobGas` before going live.

## Related skills

- `/titon:kronos-target-receiver` — write the Tolk receiver this job will call.
- `/titon:kronos-gas-sizing` — measure the right `gasLimit` for that receiver.
- `/titon:kronos-debug-exit-code` if RegisterJob reverts.
- `/titon:kronos-monitor-events` to watch for `JobExecuted`, `JobFunded`, `JobExpired`.
- `/titon:kronos-job-lifecycle` once the job exists and you need to update / pause / cancel it.
