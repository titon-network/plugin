---
name: kronos-job-lifecycle-python
description: Update, pause, resume, withdraw from, or cancel an existing Kronos job from Python. Use when the user wants to change a registered job's parameters, top up funds, drain excess balance, temporarily disable, or fully retire a job from a Python application.
---

# Manage an existing Kronos job (Python)

You're helping the user mutate a job that's already registered, from a Python service or script using `titon-network-kronos-sdk`. Every operation here requires the **job owner wallet** — the wallet that originally signed `RegisterJob`.

## What you need

- `job_id` (`int`) — read it from the `JobRegistered` event when the job was registered, or from `await client.jobs.count() - 1` for the most recent one.
- The owner's `pytoniq.WalletV5R1` instance.
- Each operation needs gas: `value=cfg.min_gas_reserve` (default 0.05 TON) is the floor.

## Operation cheatsheet

| Op | Method | Effect | Resets `last_executed_at`? | Requires owner? |
|----|--------|--------|----------------------------|-----------------|
| update | `client.jobs.update(...)` | Rewrite reward + interval + gas_limit | No | yes |
| update_full | `client.jobs.update_full(...)` | Rewrite every mutable field | No (preserves execution_count) | yes |
| pause | `client.jobs.pause(...)` | Set `is_active=False`; Execute reverts | No | yes |
| resume | `client.jobs.resume(...)` | Set `is_active=True` | No | yes |
| fund | `client.jobs.fund(...)` | Top up balance | No | no (anyone can fund) |
| withdraw | `client.jobs.withdraw(...)` | Pull excess balance, keep job alive | No | yes |
| cancel | `client.jobs.cancel(...)` | Refund + delete the job entry | gone | yes |

## Setup boilerplate

```python
from pytoniq import LiteBalancer
from kronos_sdk import KRONOS_TESTNET, KronosClient, KronosRegistry

TON = 10**9

rpc = LiteBalancer.from_testnet_config(trust_level=2)
await rpc.start_up()

registry = KronosRegistry.create_from_address(KRONOS_TESTNET.registry, client=rpc)
client = KronosClient(registry=registry)
cfg = await client.jobs.config()
```

## Update reward / interval / gas_limit

```python
# CRITICAL: update() rewrites ALL THREE fields. To change one, fetch first.
current = await client.jobs.get(job_id)
if current is None:
    raise RuntimeError("job not found")

await client.jobs.update(
    owner_wallet,
    value=cfg.min_gas_reserve,
    job_id=job_id,
    reward=int(0.07 * TON),       # new value
    interval=current.interval,    # preserve
    gas_limit=current.gas_limit,  # preserve
)
```

For broader changes (target, message body, windows, expiry, max_executions) use `client.jobs.update_full(...)` — same trap, fetch all current values you want preserved.

## Pause / resume

```python
await client.jobs.pause(owner_wallet, value=cfg.min_gas_reserve, job_id=job_id)
# later...
await client.jobs.resume(owner_wallet, value=cfg.min_gas_reserve, job_id=job_id)
```

While paused: Execute reverts with `E_JOB_NOT_ACTIVE` (127). Owner can still cancel/withdraw/update.

## Top up funds

```python
# Anyone can fund — not owner-restricted.
await client.jobs.fund(any_wallet, value=int(1 * TON), job_id=job_id)
# `value` minus min_gas_reserve is credited to the job balance.
```

## Withdraw excess (without cancelling)

```python
# CRITICAL: active jobs reject withdrawals that leave balance < execution_cost
# (E_BELOW_MIN_FOR_ACTIVE = 134). Compute the safe ceiling inline — no helper
# in the SDK because it's a two-line clamp.
job = await client.jobs.get(job_id)
econ = await client.jobs.economics(job_id)
assert job is not None and econ is not None

if job.is_active:
    max_amount = max(0, job.balance - econ.total_cost)
else:
    max_amount = job.balance

await client.jobs.withdraw(
    owner_wallet,
    value=cfg.min_gas_reserve,
    job_id=job_id,
    amount=max_amount,        # or any amount <= max_amount
)
```

To drain everything, **pause first** (then withdraw is unrestricted) or **cancel** (refunds + deletes).

## Ensure-funded helper (top up to a horizon if needed)

The Python SDK includes a one-shot helper that combines balance check + fund-to-target:

```python
from kronos_sdk import EnsureFundedOptions

result = await client.jobs.ensure_funded(
    owner_wallet,
    job_id,
    options=EnsureFundedOptions(min_runs=20, top_up_runs=50),
)
print(result)  # EnsureFundedResult(sent, value, runs_before)
```

Useful for a periodic monitor that keeps jobs topped up automatically.

## Cancel and verify

```python
await client.jobs.cancel(owner_wallet, value=cfg.min_gas_reserve, job_id=job_id)

# `cancel` wipes the entire map entry. Verify:
print(await client.jobs.exists(job_id))   # False
print(await client.jobs.get(job_id))      # None
print(await client.jobs.balance(job_id))  # 0
```

Refund (remaining balance + the message's value minus reserve) goes to the job owner with bounce disabled.

## Pitfalls — call out to the user

- **`update()` is full-overwrite.** Even if they only want to change reward, you must read + re-supply interval + gas_limit. Otherwise you accidentally "update" them to whatever you guessed.
- **Withdraw on active job has a floor.** `balance - amount >= execution_cost` or it reverts (134 BelowMinForActive). To drain everything, pause first or cancel.
- **Cancel is destructive — no undo.** The job entry is wiped. They'd need a new RegisterJob (gets a new jobId).
- **Pause-skip ops still work while paused.** Cancel, withdraw, fund, cleanup, sweep all work on paused jobs. That's by design — owners must always be able to retire a paused job.
- **The owner is the wallet that registered**, not the registry owner. If the user lost that wallet's keys, the job is unmanageable (only housekeeping can clean it up, and only after `expire_after` elapses).
- **`fund()` is permissionless.** Anyone — not just the owner — can top up a job. Useful for automaton consortiums or grant-funded automations.

## Related skills

- `/titon:kronos-register-job-python` — to create the job in the first place.
- `/titon:kronos-debug-exit-code-python` — when an op reverts (132 NotJobOwner, 134 BelowMinForActive, 127 JobNotActive are the most common here).
- `/titon:kronos-monitor-events-python` — watch `JobUpdated`, `JobPaused`, `JobResumed`, `JobWithdrawn`, `JobCancelled` to confirm.
