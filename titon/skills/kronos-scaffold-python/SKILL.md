---
name: kronos-scaffold-python
description: Scaffold the full Kronos integration for a new target contract using a Python toolchain — receiver Tolk snippet, pytest integration test, deploy-and-register Python script. Use when the user wants a one-shot "make this contract Kronos-triggered" action and their stack is Python rather than TypeScript.
---

# Scaffold Kronos integration into an existing contract (Python stack)

You're helping a dev **bolt Kronos onto their existing contract** in one pass, with Python tooling. Given:

- A Tolk contract path (the target).
- An opcode they want Kronos to dispatch to.
- A cadence (interval seconds + reward).

You generate: (1) the receiver snippet to paste into the contract, (2) a pytest integration test, (3) a deploy + register Python script. Then point them at the right follow-up skills.

## Ask the user

| Field | What to extract |
|-------|-----------------|
| Target contract path | e.g. `contracts/my-staker.tolk`. Read it to understand the existing dispatch pattern. |
| Opcode name | PascalCase, e.g. `CronTick`. Suggest a hex value in the `0x7000`-`0x7FFF` user range. |
| Interval | Seconds. Snap to sane values (60, 300, 3600, 86400). |
| Reward | Default `int(0.05 * TON)`. |
| Gas limit | Default `int(0.02 * TON)` (simple receivers) — flag that they should measure via `/titon:kronos-gas-sizing-python`. |
| Kronos registry address | Use `KRONOS_TESTNET.registry` from the SDK by default. |

## Step 1 — Receiver snippet (paste into their contract)

```tolk
// Add near top of contracts/<their-contract>.tolk:
const OP_CRON_TICK = 0x7001;           // pick from 0x7000-0x7FFF range

// Add to the Storage struct (if not already present):
struct Storage {
    // ... their existing fields ...
    kronosRegistry: address;
    lastTickAt: uint32;
}

// Add to onInternalMessage dispatch:
if (op == OP_CRON_TICK) {
    handleCronTick(in.senderAddress);
    return;
}

// New receiver:
fun handleCronTick(sender: address) {
    var storage = lazy Storage.load();
    assert (sender == storage.kronosRegistry) throw 401;
    val now_ts = blockchain.now();
    assert (now_ts > storage.lastTickAt) throw 402;

    // TODO: your business logic here
    doTheWork();

    storage.lastTickAt = now_ts;
    storage.save();
}
```

See `/titon:kronos-target-receiver-python` for the full pattern (idempotence, gas budget, pitfalls).

## Step 2 — pytest integration test (`tests/test_kronos.py`)

```python
import asyncio
import os
import pytest
import pytest_asyncio
from pytoniq import LiteBalancer, WalletV5R1
from pytoniq_core import Address, begin_cell
from kronos_sdk import (
    KRONOS_TESTNET,
    JobOptsInput,
    KronosClient,
    KronosRegistry,
    register_job_opts,
)

TON = 10**9
OP_CRON_TICK = 0x7001


@pytest_asyncio.fixture(scope="module")
async def rpc():
    client = LiteBalancer.from_testnet_config(trust_level=2)
    await client.start_up()
    yield client
    await client.close_all()


@pytest.mark.asyncio
async def test_cron_tick_advances_state(rpc, owner_wallet, target_address):
    registry = KronosRegistry.create_from_address(KRONOS_TESTNET.registry, client=rpc)
    client = KronosClient(registry=registry)
    cfg = await client.jobs.config()

    opts = register_job_opts(
        JobOptsInput(
            target=Address(target_address),
            message=begin_cell().store_uint(OP_CRON_TICK, 32).end_cell(),
            interval=300,
            reward=int(0.05 * TON),
            gas_limit=int(0.02 * TON),
        ),
        cfg,
    )

    await client.jobs.register(owner_wallet, **opts.as_kwargs())
    await asyncio.sleep(8)

    job_id = (await client.jobs.count()) - 1

    # First execution is permissionless
    await client.jobs.execute(owner_wallet, value=int(0.5 * TON), job_id=job_id)
    await asyncio.sleep(8)

    job = await client.jobs.get(job_id)
    assert job is not None
    assert job.execution_count >= 1

    # Cleanup
    await client.jobs.cancel(owner_wallet, value=cfg.min_gas_reserve, job_id=job_id)
```

## Step 3 — Deploy + register script (`scripts/register_cron.py`)

```python
"""Register the Kronos cron job for our target contract.

Usage:
    KRONOS_OWNER_MNEMONIC="word word ..." python scripts/register_cron.py <target-address>
"""

import asyncio
import os
import sys
from pytoniq import LiteBalancer, WalletV5R1
from pytoniq_core import Address, begin_cell
from kronos_sdk import (
    KRONOS_TESTNET,
    JobOptsInput,
    KronosClient,
    KronosRegistry,
    register_job_opts,
)

TON = 10**9
OP_CRON_TICK = 0x7001


async def main():
    if len(sys.argv) != 2:
        print(f"usage: {sys.argv[0]} <target-address>")
        sys.exit(1)
    target_address = sys.argv[1]

    mnemonic = os.environ["KRONOS_OWNER_MNEMONIC"].split()

    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()
    try:
        owner_wallet = await WalletV5R1.from_mnemonic(rpc, mnemonic)

        registry = KronosRegistry.create_from_address(KRONOS_TESTNET.registry, client=rpc)
        client = KronosClient(registry=registry)
        cfg = await client.jobs.config()

        opts = register_job_opts(
            JobOptsInput(
                target=Address(target_address),
                message=begin_cell().store_uint(OP_CRON_TICK, 32).end_cell(),
                interval=300,
                reward=int(0.05 * TON),
                gas_limit=int(0.02 * TON),
            ),
            cfg,
        )

        await client.jobs.register(owner_wallet, **opts.as_kwargs())
        print("Registered. Watch for the JobRegistered event to get the jobId.")
    finally:
        await rpc.close_all()


if __name__ == "__main__":
    asyncio.run(main())
```

For mainnet, swap `KRONOS_TESTNET` once it's replaced by `KRONOS_MAINNET`.

## Checklist before running

- [ ] Tolk receiver compiles. Use Blueprint (TS) or your project's Tolk build chain.
- [ ] Test passes: `pytest tests/test_kronos.py`.
- [ ] Measure actual `gas_limit`: see `/titon:kronos-gas-sizing-python` — probably higher than the `0.02` default.
- [ ] Pick a stable opcode in `0x7000`-`0x7FFF`. If your contract already uses anything up there, don't collide.
- [ ] Target contract has a way to pin `kronosRegistry` at deploy (storage field + owner-only setter).

## What the skill does NOT generate

- **The Tolk build pipeline.** Python doesn't compile Tolk — use `@ton/blueprint` (TypeScript) or your existing toolchain to produce `MyTarget.compiled.json`. The Python SDK consumes the compiled output, it doesn't build it.
- **The actual business logic in `doTheWork()`.** Your job.
- **Mainnet deployment.** Scaffold targets testnet by default; the user is responsible for audit + mainnet cutover.
- **A wallet wrapper for the target.** Python doesn't have a one-line "wrapper" generator like Blueprint. Construct the target's `StateInit` from your compiled bytecode + the constructor params manually, or use `pytoniq_core` primitives.

## Related skills

- `/titon:kronos-target-receiver-python` — the receiver pattern in detail.
- `/titon:kronos-gas-sizing-python` — measure `gas_limit` properly before shipping.
- `/titon:kronos-integration-test-python` — full pytest pattern.
- `/titon:kronos-register-job-python` — what `register_job_opts` does under the hood.
