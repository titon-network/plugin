---
name: kronos-register-job-python
description: Build and submit a recurring on-chain job to the Kronos registry from Python. Use when the user wants to schedule a periodic contract call, automate a price update, set up a recurring payment, or otherwise needs a Kronos job registered from a Python application or service.
---

# Build a Kronos recurring job (Python)

You're helping a developer register a recurring on-chain job from Python via `titon-network-kronos-sdk`. The Kronos registry stores the schedule; permissionless automatons execute it on-time and earn the configured reward.

## What you need from the user

| Field | What to ask |
|-------|------------|
| Target contract | The address that should be called each cycle. |
| Body cell | The opaque message body to forward (schema is target-defined). If they don't have one, suggest `begin_cell().store_uint(OP, 32).end_cell()` with their op. |
| Interval | "How often?" Convert to seconds. Reject < 60 (`minInterval`) or > 86_400 (`maxInterval`) by default. |
| Reward | Per-execution automaton reward. Default `to_nano('0.05')` — anything below registry's `minReward` is rejected on-chain. |
| Gas budget | Forwarded to target. Default `to_nano('0.02')` for simple targets; bump to `to_nano('0.1')` for complex ones. Empirically size via `/titon:kronos-gas-sizing-python`. |

If the user is vague, suggest defaults and confirm.

## Install

```bash
pip install titon-network-kronos-sdk pytoniq
# or with poetry / uv:
poetry add titon-network-kronos-sdk pytoniq
```

## The recipe

```python
import asyncio
import os
from pytoniq import LiteBalancer, WalletV5R1
from pytoniq_core import Address, begin_cell
from kronos_sdk import (
    KRONOS_TESTNET,
    JobOptsInput,
    KronosClient,
    KronosRegistry,
    register_job_opts,
)

TON = 10**9  # 1 TON = 10^9 nanotons


async def main():
    # Live testnet address comes from the SDK — don't hardcode.
    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()
    try:
        registry = KronosRegistry.create_from_address(
            KRONOS_TESTNET.registry, client=rpc
        )
        client = KronosClient(registry=registry)

        # Pull live limits (min_interval, min_reward, etc.) for auto-funding + validation.
        cfg = await client.jobs.config()

        # Build the typed opts. JobOptsInput is a dataclass — IDE-friendly + validated.
        # register_job_opts() returns a SendRegisterJobOpts with all fields resolved.
        opts = register_job_opts(
            JobOptsInput(
                target=Address(target_address),
                message=begin_cell().store_uint(0, 32).end_cell(),
                interval=3600,                  # seconds
                reward=int(0.05 * TON),
                gas_limit=int(0.02 * TON),
                window_before=30,               # optional
                window_after=600,
                max_executions=0,               # 0 = unlimited
            ),
            cfg,                                # auto-funds 100 runs
        )

        # Load the owner wallet from a 24-word mnemonic.
        mnemonic = os.environ["KRONOS_OWNER_MNEMONIC"].split()
        owner_wallet = await WalletV5R1.from_mnemonic(rpc, mnemonic)

        # Spread the resolved opts as kwargs (the SDK's canonical pattern).
        await client.jobs.register(owner_wallet, **opts.as_kwargs())

        # The register send doesn't synchronously return the new jobId — read
        # the JobRegistered event from the registry's recent externals after
        # inclusion. (Allow a few seconds for testnet inclusion.)
        await asyncio.sleep(8)
        txs = await rpc.get_transactions(KRONOS_TESTNET.registry, count=20)
        bodies = [
            out.body
            for tx in txs
            for out in tx.out_msgs
            if out.is_external_out      # external-out = event (canonical pytoniq idiom)
        ]
        events = client.events.decode_all(bodies)
        registered = next(
            (e for e in events if e.kind == "JobRegistered" and e.owner == owner_wallet.address),
            None,
        )
        if registered is not None:
            print(f"new job id: {registered.job_id}")
    finally:
        await rpc.close_all()


asyncio.run(main())
```

## Funding math (so user doesn't get rejected)

- Per-execution cost = `reward + gas_limit + reward * protocol_fee_bps // 10_000`.
- Min attached value to RegisterJob = at least `min_funding + min_gas_reserve` from registry config.
- `register_job_opts(input, cfg)` auto-funds 100 runs; override by setting `funding=int(5 * TON)` on the `JobOptsInput` for a different horizon.
- Show the user the projected burn rate: `cost_per_run * (86400 // interval)` = TON per day.

For pure cost preview without sending:

```python
from kronos_sdk import preview_job_cost, PreviewJobCostInput

preview = preview_job_cost(PreviewJobCostInput(
    reward=int(0.05 * TON),
    gas_limit=int(0.02 * TON),
    interval=3600,
    cfg=cfg,
    horizon_runs=100,
))
print(f"per-run: {preview.per_run_cost / TON} TON")
print(f"100-run funding: {preview.recommended_funding / TON} TON")
```

## Pitfalls — call out to the user

- **Body cell is target-specific.** If the target rejects the body (TVM exit 0xFFFF or 9), the automaton still gets paid — the job just no-ops on-chain. Confirm the body matches what the target accepts.
- **Don't set `expire_after` < `now + interval`.** The job can't run if it expires before the first window opens.
- **Once registered, only the owner can pause/update/cancel.** The wallet that signed RegisterJob is the owner.
- **First execution is permissionless.** Anyone can fire it. Subsequent runs go through assigned-automaton rotation if the registry is wired to a ForgeTON pool.
- **Size `gas_limit` empirically.** The default 0.02 TON works only for trivial receivers. Use `/titon:kronos-gas-sizing-python` to measure before going live.
- **`int` math, not `float`.** All amounts are nanotons (`int`). `int(0.05 * TON)` is OK at small magnitudes; for production-grade cost math prefer `decimal.Decimal` or the SDK's helpers.

## Related skills

- `/titon:kronos-target-receiver-python` — write the Tolk receiver this job will call (Python-side perspective on testing it).
- `/titon:kronos-gas-sizing-python` — measure the right `gas_limit` for that receiver.
- `/titon:kronos-debug-exit-code-python` if RegisterJob reverts.
- `/titon:kronos-monitor-events-python` to watch for `JobExecuted`, `JobFunded`, `JobExpired`.
- `/titon:kronos-job-lifecycle-python` once the job exists and you need to update / pause / cancel it.
