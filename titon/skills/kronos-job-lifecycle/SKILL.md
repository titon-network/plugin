---
name: kronos-job-lifecycle
description: Update, pause, resume, withdraw from, or cancel an existing Kronos job. Use when the user wants to change a registered job's parameters, top up funds, drain excess balance, temporarily disable, or fully retire a job.
---

# Manage an existing Kronos job

You're helping the user mutate a job that's already registered. Every operation here requires the **job owner wallet** — the wallet that originally signed `RegisterJob`.

## What you need

- `jobId` (bigint) — read it from the `JobRegistered` event when the job was registered, or from `client.jobs.count() - 1n` for the most recent one.
- The owner wallet's `Sender`.
- Each operation needs gas: `value: cfg.minGasReserve` (default 0.05 TON) is the floor.

## Operation cheatsheet

| Op | Effect | Resets `lastExecutedAt`? | Requires owner? |
|----|--------|--------------------------|-----------------|
| `update` | Rewrite reward + interval + gasLimit | No | yes |
| `updateFull` | Rewrite every mutable field | No (preserves executionCount) | yes |
| `pause` | Set `isActive=false`; Execute reverts | No | yes |
| `resume` | Set `isActive=true` | No | yes |
| `fund` | Top up balance | No | no (anyone can fund) |
| `withdraw` | Pull excess balance, keep job alive | No | yes |
| `cancel` | Refund + delete the job entry | gone | yes |

## Update reward / interval / gasLimit

```ts
// CRITICAL: update() rewrites ALL THREE fields. To change one, fetch first.
const current = await client.jobs.get(jobId);
if (!current) throw new Error('job not found');

await client.jobs.update(ownerSender, {
    value: cfg.minGasReserve,
    jobId,
    reward: toNano('0.07'),       // new value
    interval: current.interval,   // preserve
    gasLimit: current.gasLimit,   // preserve
});
```

For broader changes (target, message body, windows, expiry, maxExecutions) use `updateFull` — same trap, fetch all current values you want preserved.

## Pause / resume

```ts
await client.jobs.pause(ownerSender, { value: cfg.minGasReserve, jobId });
// later...
await client.jobs.resume(ownerSender, { value: cfg.minGasReserve, jobId });
```

While paused: Execute reverts with `E_JOB_NOT_ACTIVE` (127). Owner can still cancel/withdraw/update.

## Top up funds

```ts
// Anyone can fund — not owner-restricted.
await client.jobs.fund(anySender, { value: toNano('1'), jobId });
// `value` minus minGasReserve is credited to the job balance.
```

## Withdraw excess (without cancelling)

```ts
// CRITICAL: active jobs reject withdrawals that leave balance < executionCost
// (E_BELOW_MIN_FOR_ACTIVE = 134). Compute the safe ceiling inline — no helper
// in the SDK because it's a two-line clamp.
const job = await client.jobs.get(jobId);
const econ = await client.jobs.economics(jobId);
const max = job!.isActive
    ? (job!.balance > econ!.totalCost ? job!.balance - econ!.totalCost : 0n)
    : job!.balance;

await client.jobs.withdraw(ownerSender, {
    value: cfg.minGasReserve,
    jobId,
    amount: max,        // or any amount <= max
});
```

To drain everything, **pause first** (then withdraw is unrestricted) or **cancel** (refunds + deletes).

## Cancel and verify

```ts
await client.jobs.cancel(ownerSender, { value: cfg.minGasReserve, jobId });

// `cancel` wipes the entire map entry. Verify:
console.log(await client.jobs.exists(jobId));   // false
console.log(await client.jobs.get(jobId));      // null
console.log(await client.jobs.balance(jobId));  // 0n
```

Refund (remaining balance + the message's value minus reserve) goes to the job owner via `BounceMode.NoBounce`.

## Pitfalls — call out to the user

- **`update()` is full-overwrite.** Even if they only want to change reward, you must read + re-supply interval + gasLimit. Otherwise you accidentally "update" them to whatever you guessed.
- **Withdraw on active job has a floor.** `balance - amount >= executionCost` or it reverts (134 BelowMinForActive). To drain everything, pause first or cancel.
- **Cancel is destructive — no undo.** The job entry is wiped. They'd need a new RegisterJob (gets a new jobId).
- **Pause-skip ops still work while paused.** Cancel, withdraw, fund, cleanup, sweep all work on paused jobs. That's by design — owners must always be able to retire a paused job.
- **The owner is the wallet that registered**, not the registry owner. If the user lost that wallet's keys, the job is unmanageable (only housekeeping can clean it up, and only after expireAfter elapses).
- **`fund()` is permissionless.** Anyone — not just the owner — can top up a job. Useful for automaton consortiums or grant-funded automations.

## Related skills

- `/titon:kronos-register-job` — to create the job in the first place.
- `/titon:kronos-debug-exit-code` — when an op reverts (132 NotJobOwner, 134 BelowMinForActive, 127 JobNotActive are the most common here).
- `/titon:kronos-monitor-events` — watch `JobUpdated`, `JobPaused`, `JobResumed`, `JobWithdrawn`, `JobCancelled` to confirm.
