---
name: kronos-gas-sizing-python
description: Empirically measure and size the gas_limit for a Kronos-triggered target contract from Python. Use when the user is picking `gas_limit` for RegisterJob, sees silent Execute reverts on a Python deploy, or wants to add a regression guard in pytest.
---

# Size `gas_limit` for a Kronos job (Python)

You're helping a dev pick the right `gas_limit` for a RegisterJob call from Python. Undersizing is the #1 Kronos integration footgun: the automaton gets paid for sending but the target reverts out-of-gas — state doesn't advance and the user sees "why isn't my job running?"

## The fundamental rule

> **`gas_limit` is forwarded verbatim to the target.** The registry only enforces `gas_limit > 0`; it does not estimate for you. If the target needs 8500 gas and you registered 5000, every execution silently OOGs and you never hear about it on-chain.

## The Python reality: no in-memory sandbox

Unlike the TypeScript SDK (`@ton/sandbox` runs the contract VM in-process), Python doesn't have a comparable harness. Three pragmatic approaches:

### 1. Use the helper functions for off-chain math

Pure helpers in the Python SDK that don't need a VM:

```python
from kronos_sdk import recommended_gas_limit, recommended_register_value

# Wrap a measured gasUsed with a 20% buffer
result = recommended_gas_limit(50_000, buffer_percent=20)
print(f"gas_used={result.gas_used}  recommended={result.recommended}")

# Compute the right RegisterJob value from cfg
value = recommended_register_value(
    reward=int(0.05 * TON),
    gas_limit=result.recommended,
    protocol_fee_bps=cfg.protocol_fee_bps,
    min_gas_reserve=cfg.min_gas_reserve,
    min_funding=cfg.min_funding,
    executions=100,
)
```

These mirror the on-chain math byte-for-byte but don't measure gas — you still need a measured `gas_used` from somewhere.

### 2. Measure on testnet (one-shot Execute, then inspect)

Cheapest end-to-end measurement; takes 5–15 s per cycle:

```python
import asyncio
from pytoniq import LiteBalancer
from kronos_sdk import KRONOS_TESTNET, KronosClient, KronosRegistry

TON = 10**9


async def measure_target_gas(target_address: str, job_id: int):
    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()

    registry = KronosRegistry.create_from_address(KRONOS_TESTNET.registry, client=rpc)
    client = KronosClient(registry=registry)

    # Fire Execute (permissionless on first run)
    await client.jobs.execute(any_wallet, value=int(0.5 * TON), job_id=job_id)

    # Wait a few seconds for inclusion, then pull the target's recent txs
    import time; time.sleep(8)
    txs = await rpc.get_transactions(target_address, count=5)

    # Find the inbound from the registry — its compute phase has gas_used
    for tx in txs:
        in_msg = tx.in_msg
        if in_msg and str(in_msg.info.src) == str(KRONOS_TESTNET.registry):
            gas_used = tx.description.compute_ph.gas_used
            print(f"target gas_used={gas_used}  recommended (20% buffer)={gas_used * 12 // 10}")
            break

    await rpc.close_all()
```

Use this to validate the gas number you got from the TS sandbox before going live.

### 3. Lean on the TS sandbox for tight iteration

For active development of the receiver itself, `@ton/sandbox` + `estimateJobGas` from `@titon-network/kronos-sdk` (TS) is the productive path — sub-second per measurement. Once you've converged on a number, use the Python deploy path (`/titon:kronos-register-job-python`) with that constant.

This isn't a workaround — it reflects real engineering productivity. Python tests on testnet are perfect for "did the deploy land correctly?"; the TS sandbox is perfect for "is the receiver tight?".

## Pick a buffer — 20% is not magic

- **20%** — safe default for stable code paths.
- **40%** — paths with variable work (iterating a map, forking on balance, etc.).
- **Measure the worst case, not the average.** Sample with full state, not an empty target.
- **Re-measure after changes.** Any logic edit to the receiver invalidates the estimate.

## Regression guard — pin the gas in pytest

```python
import pytest
from pytoniq import LiteBalancer
from kronos_sdk import KRONOS_TESTNET, KronosClient, KronosRegistry

@pytest.mark.asyncio
async def test_cron_tick_stays_under_gas_limit(target_address, job_id, any_wallet):
    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()

    registry = KronosRegistry.create_from_address(KRONOS_TESTNET.registry, client=rpc)
    client = KronosClient(registry=registry)

    await client.jobs.execute(any_wallet, value=int(0.5 * TON), job_id=job_id)
    # ...wait + pull tx like above...
    gas_used = ...

    # Fail the build if the receiver grows past 80% of the registered gas_limit.
    GAS_LIMIT_NANO = 20_000_000      # 0.02 TON
    assert gas_used < int(GAS_LIMIT_NANO * 0.8)

    await rpc.close_all()
```

This catches regressions: a refactor that bumps gas from 0.012 → 0.018 passes today but blows up at 0.025.

## What counts toward gas_used

- **Storage reads** — cheap but not free. `lazy` helps (only materialises fields you read).
- **Storage writes** — the expensive one. Every `storage.save()` pays rent-style costs proportional to total bits.
- **Emitting events** — `createExternalLogMessage` is ~1500 gas on its own; every field adds ~50.
- **Outgoing sends** — ~500 gas per action, more if the value > 0.
- **Map operations** — O(log N) in cell depth. A map with 1k entries costs ~5x a map with 10.

## Pitfalls — call out to the user

- **Don't size based on an empty-state target.** Real gas is higher once your maps/refs fill. Measure against realistic state.
- **Don't use `execution_cost` for sizing.** That's the *automaton's* economics (reward + gas + fee). `gas_limit` is what your target receives, which is a separate number.
- **Don't pick `gas_limit = execution_economics.total_cost`.** Those are incompatible units — one is "what the job spends", the other is "what the target gets".
- **Don't forget: 1 TON = 10^9 nano-TON.** `int(0.02 * TON)` = 20_000_000. Gas numbers in `gas_used` are already in nano-TON.
- **Testnet ≠ mainnet gas exactly.** Gas prices are the same in both, but block timing / cell-depth distributions can shift. Pad the buffer if your contract is hot.

## Related skills

- `/titon:kronos-target-receiver-python` — write the receiver that this gas sizes.
- `/titon:kronos-register-job-python` — plug the measured `gas_limit` into RegisterJob.
- `/titon:kronos-debug-exit-code-python` — `explain_error(13)` for OOG symptoms.
