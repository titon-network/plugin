---
name: kronos-target-receiver-python
description: Write the receiver in a Tolk contract that Kronos will call on schedule, and exercise it from Python. Use when the user is building the target of a Kronos job and wants to drive integration tests / one-off triggers from Python — opcode dispatch, sender validation, idempotence under retry, gas budget sanity.
---

# Build a Kronos-triggered receiver (Python perspective)

You're helping a Tolk contract dev write the **target side** of a Kronos job — the contract the registry calls every N seconds. The Tolk part is identical regardless of language; this skill covers the Tolk pattern + the Python-side tooling for triggering / inspecting it.

## Mental model — three things to get right (Tolk side)

1. **Opcode dispatch.** Kronos calls your contract with whatever body the job owner registered. Your contract still sees an `onInternalMessage` with a 32-bit op prefix; you dispatch on that. Kronos does **not** wrap the call in its own envelope — the body is passed through verbatim.
2. **Sender validation (optional but usually wanted).** Kronos forwards from the *registry* address, not the job owner. If only the registry may trigger this path, assert `sender == KRONOS_REGISTRY`.
3. **Idempotence.** Kronos uses `BounceMode.NoBounce` — automatons get paid for sending, not for your success. A failed execution still counts as executed on-chain. Make your handler re-runnable: either idempotent by design (snapshot prices, bump counters) or guarded by a "last processed" watermark.

## The Tolk recipe — bare-bones receiver

```tolk
// contracts/my-target.tolk
import "@stdlib/common.tolk";

const OP_CRON_TICK = 0x7001;

struct Storage {
    owner: address;
    kronosRegistry: address;
    lastTickAt: uint32;
    tickCount: uint64;
}

fun onInternalMessage(in: InMessage) {
    var cs = in.body.beginParse();
    val op = cs.loadUint(32);

    if (op == OP_CRON_TICK) {
        handleCronTick(in.senderAddress);
        return;
    }

    assert (false) throw 0xFFFF;   // unknown opcode
}

fun handleCronTick(sender: address) {
    var storage = lazy Storage.load();

    assert (sender == storage.kronosRegistry) throw 401;

    val now_ts = blockchain.now();
    assert (now_ts > storage.lastTickAt) throw 402;

    doTheWork();

    storage.lastTickAt = now_ts;
    storage.tickCount += 1;
    storage.save();
}
```

## Python-side: trigger an Execute by hand (testnet smoke test)

Kronos in production runs the Execute on a schedule, but you'll often want to fire one manually for testing. Use `client.jobs.execute(...)`:

```python
import asyncio
from pytoniq import LiteBalancer
from kronos_sdk import KRONOS_TESTNET, KronosClient, KronosRegistry

TON = 10**9


async def trigger():
    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()

    registry = KronosRegistry.create_from_address(KRONOS_TESTNET.registry, client=rpc)
    client = KronosClient(registry=registry)

    # Permissionless first run — any wallet can fire it.
    await client.jobs.execute(any_wallet, value=int(0.5 * TON), job_id=job_id)

    # Read the new state
    job = await client.jobs.get(job_id)
    print(f"execution_count={job.execution_count}  last_executed_at={job.last_executed_at}")

    await rpc.close_all()


asyncio.run(trigger())
```

## Storing the Kronos address (once, at deploy)

```tolk
fun handleSetKronosRegistry(sender: address, newAddr: address) {
    var storage = lazy Storage.load();
    assert (sender == storage.owner) throw 100;            // E_NOT_OWNER
    assert (storage.kronosRegistry.isNone()) throw 101;    // one-shot pin
    storage.kronosRegistry = newAddr;
    storage.save();
}
```

One-shot pin matches how Kronos itself pins `forgeton`. Re-pointing = timelocked code upgrade or redeploy.

## Gas budget — measuring from Python

Python doesn't have an in-memory sandbox like `@ton/sandbox`. Two options to size `gas_limit`:

1. **Run a one-shot Execute on testnet, then read back the trace.**
   ```python
   await client.jobs.execute(any_wallet, value=int(0.5 * TON), job_id=job_id)

   # Pull recent transactions for the target — gasUsed is in the compute phase.
   txs = await rpc.run_get_method(target_address, ...)  # or rpc.get_transactions(...)
   # Inspect compute_phase.gas_used and pick gas_limit = gas_used * 1.2
   ```

2. **Use the TS sandbox for tight iteration, then port the chosen `gas_limit` to Python deploy.**
   `@ton/sandbox` runs in a fraction of a second; `pytoniq` against testnet takes 5–10 s per Execute. For development, the TS sandbox is the productive path; Python is for production / ops.

See `/titon:kronos-gas-sizing-python` for the full pattern.

## Pitfalls — call out to the user

- **Do not `throw` to reject a cycle.** It reverts the whole receiver; the automaton still got paid; your watermark never advances. Return early silently instead.
- **Do not depend on `random()` for adversarial entropy.** The assigned automaton knows when Kronos will call. Use Fortuna (`/titon:fortuna-integrate-consumer-python`) for verifiable randomness.
- **Do not `bounceOnInvalid`.** Kronos sends `BounceMode.NoBounce`; a bounce would silently refund *you*, not the automaton.
- **Do not forget to fund `gas_limit` headroom for events.** `createExternalLogMessage` is ~1500 gas on its own; every `emitEvent` pushes the ceiling.
- **Do not check `sender == jobOwner`.** Kronos forwards from the *registry* — the job owner is the one who paid for the job, but they're not the sender. Use `sender == KRONOS_REGISTRY`.
- **No Python sandbox.** Integration tests against Python live on testnet (slow + needs funded wallet) or rely on the TS `SandboxKronos` harness for tight loops, then validate Python deploy paths with a smoke run.

## Related skills

- `/titon:kronos-gas-sizing-python` — empirically measure `gas_limit` against testnet.
- `/titon:kronos-integration-test-python` — testnet integration test pattern with pytest.
- `/titon:kronos-register-job-python` — register the job that calls this receiver.
- `/titon:kronos-scaffold-python` — one-shot generator: receiver + register script.
- `/titon:kronos-debug-exit-code-python` — triage what a reverted execution means.
