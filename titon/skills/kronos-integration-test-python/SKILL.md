---
name: kronos-integration-test-python
description: Write a pytest integration test for a Kronos-triggered contract from Python — testnet-driven, pytoniq-based, no in-memory sandbox. Use when the user is testing their target end-to-end from Python — deploy + register + Execute + assert.
---

# Test a Kronos-triggered contract end-to-end (Python)

You're helping a dev write a pytest integration test that: (1) deploys their Tolk target (or assumes it's already on testnet), (2) registers a Kronos job against it, (3) fires `Execute`, (4) asserts state changed.

**Important difference from TS:** Python doesn't have an in-memory sandbox like `@ton/sandbox`. Tests run against TON testnet using `pytoniq`. Slower (5–15 s per cycle) but realistic — every byte of behaviour matches mainnet.

## When to use Python tests vs the TS sandbox

| Need | Use |
|------|-----|
| Tight iteration on the receiver's logic / gas | TS `SandboxKronos` harness — sub-second per cycle |
| Verify a Python deploy / ops script works end-to-end | Python pytest against testnet |
| Test that the on-chain state matches what your dapp expects | Either, but Python is closer to your prod stack |
| CI regression for receiver gas budgets | Either |

For most teams: write the receiver-correctness suite in TS sandbox, then add a thin Python smoke suite that exercises the deploy + register + execute path against testnet.

## Install

```bash
pip install titon-network-kronos-sdk pytoniq pytest pytest-asyncio
```

## The pytest pattern

```python
# tests/test_kronos_integration.py
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


@pytest_asyncio.fixture(scope="module")
async def owner_wallet(rpc):
    mnemonic = os.environ["KRONOS_TEST_MNEMONIC"].split()
    return await WalletV5R1.from_mnemonic(rpc, mnemonic)


@pytest_asyncio.fixture(scope="module")
async def client(rpc):
    registry = KronosRegistry.create_from_address(KRONOS_TESTNET.registry, client=rpc)
    yield KronosClient(registry=registry)


@pytest.mark.asyncio
async def test_register_and_execute(client, owner_wallet, target_address):
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

    # Register a job (spread the resolved opts as kwargs)
    await client.jobs.register(owner_wallet, **opts.as_kwargs())

    # Wait for inclusion, then read the most recently registered job_id
    await asyncio.sleep(8)
    job_id = (await client.jobs.count()) - 1

    # First execution is permissionless
    await client.jobs.execute(owner_wallet, value=int(0.5 * TON), job_id=job_id)
    await asyncio.sleep(8)

    # Assert the job's execution_count incremented
    job = await client.jobs.get(job_id)
    assert job is not None
    assert job.execution_count >= 1

    # Cleanup
    await client.jobs.cancel(owner_wallet, value=cfg.min_gas_reserve, job_id=job_id)
```

## Configuration

Test mnemonic + funded testnet wallet are required. Common conftest pattern:

```python
# conftest.py
import os
import pytest


def pytest_collection_modifyitems(config, items):
    if not os.environ.get("KRONOS_TEST_MNEMONIC"):
        skip_marker = pytest.mark.skip(reason="set KRONOS_TEST_MNEMONIC to run integration tests")
        for item in items:
            if "integration" in item.keywords:
                item.add_marker(skip_marker)


@pytest.fixture(scope="session")
def target_address():
    addr = os.environ.get("KRONOS_TEST_TARGET")
    if not addr:
        pytest.skip("set KRONOS_TEST_TARGET to a deployed target address")
    return addr
```

Run with:

```bash
KRONOS_TEST_MNEMONIC="word word word ..." \
KRONOS_TEST_TARGET="EQ..." \
pytest tests/test_kronos_integration.py -v
```

## Common patterns

### Drive multiple cycles

```python
import time

for i in range(3):
    await asyncio.sleep(310)   # past nextDue
    await client.jobs.execute(any_wallet, value=int(0.5 * TON), job_id=job_id)
    await asyncio.sleep(8)

job = await client.jobs.get(job_id)
assert job.execution_count == 3
```

> Slow! 3 cycles + 5 min interval = 15+ minutes. For tight loops use TS sandbox; reserve testnet for "does the deploy work" rather than "does the receiver work".

### Decode events from a tx

```python
from kronos_sdk import decode_event, decode_events

# Pull recent registry txs
txs = await rpc.get_transactions(KRONOS_TESTNET.registry, count=5)
bodies = [out.body for tx in txs for out in tx.out_msgs if out.is_external_out]
events = decode_events(bodies)   # drops unknown opcodes
for ev in events:
    print(ev.kind, ev)
```

### Pin the gas budget as a regression guard

```python
@pytest.mark.asyncio
async def test_target_under_gas_limit(client, owner_wallet, target_address, job_id):
    await client.jobs.execute(owner_wallet, value=int(0.5 * TON), job_id=job_id)
    await asyncio.sleep(8)

    txs = await rpc.get_transactions(target_address, count=3)
    for tx in txs:
        if tx.in_msg and str(tx.in_msg.info.src) == str(KRONOS_TESTNET.registry):
            assert tx.description.compute_ph.gas_used < int(0.016 * TON)   # 80% of 0.02 limit
            return
    pytest.fail("could not find target tx after Execute")
```

## Pitfalls — call out to the user

- **First execution is permissionless.** Don't expect `NotAssigned` on the first run regardless of rotation.
- **Wait between Execute calls.** Testnet block time is 5 s; assertions before the tx settles will read stale state. `asyncio.sleep(8)` is a safe default.
- **No fast-forward.** You can't bump testnet's clock. Tests that need multiple cycles must literally wait `interval` seconds between Executes.
- **Tests share state.** Unlike sandbox, testnet keeps every job you register forever. `cancel` at the end of each test is good hygiene.
- **Funded wallet required.** Tests cost gas (~0.5 TON per cycle if you cancel). Top up the test wallet from the faucet between runs.

## Related skills

- `/titon:kronos-target-receiver-python` — write the receiver being tested.
- `/titon:kronos-gas-sizing-python` — measure realistic gas inside the same harness.
- `/titon:kronos-debug-exit-code-python` — triage a test failure by exit code.
- `/titon:kronos-scaffold-python` — generate a starting receiver + test pair.
