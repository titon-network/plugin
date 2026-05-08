---
name: kronos-target-receiver
description: Write the receiver in a Tolk contract that Kronos will call on schedule. Use when the user is building the target of a Kronos job — opcode dispatch, sender validation, idempotence under retry, gas budget sanity.
---

# Build a Kronos-triggered receiver

You're helping a Tolk contract dev write the **target side** of a Kronos job — the contract the registry calls every N seconds. Idiomatic pattern, common pitfalls, paste-ready code.

## Mental model — three things to get right

1. **Opcode dispatch.** Kronos calls your contract with whatever body the job owner registered. Your contract still sees an `onInternalMessage` with a 32-bit op prefix; you dispatch on that. Kronos does **not** wrap the call in its own envelope — the body is passed through verbatim.
2. **Sender validation (optional but usually wanted).** Kronos forwards from the *registry* address, not the job owner. If only the registry may trigger this path, assert `sender == KRONOS_REGISTRY`.
3. **Idempotence.** Kronos uses `BounceMode.NoBounce` — automatons get paid for sending, not for your success. A failed execution still counts as executed on-chain. Make your handler re-runnable: either idempotent by design (snapshot prices, bump counters) or guarded by a "last processed" watermark.

## The recipe — bare-bones receiver

```tolk
// contracts/my-target.tolk
import "@stdlib/common.tolk";

// The opcode you told Kronos to forward. Job owner stores:
//   beginCell().storeUint(OP_CRON_TICK, 32).endCell()
// when they call RegisterJob, and every Execute the registry forwards this body.
const OP_CRON_TICK = 0x7001;

struct Storage {
    owner: address;
    kronosRegistry: address;
    lastTickAt: uint32;
    tickCount: uint64;
    // ... your own state
}

fun onInternalMessage(in: InMessage) {
    var cs = in.body.beginParse();
    val op = cs.loadUint(32);

    if (op == OP_CRON_TICK) {
        handleCronTick(in.senderAddress);
        return;
    }

    // ... your other receivers

    assert (false) throw 0xFFFF;   // unknown opcode
}

fun handleCronTick(sender: address) {
    var storage = lazy Storage.load();

    // (1) Only Kronos may trigger scheduled ticks.
    assert (sender == storage.kronosRegistry) throw 401;

    // (2) Idempotent watermark — even if the registry retries via its fallback
    //     window, we process each cycle at most once.
    val now_ts = blockchain.now();
    assert (now_ts > storage.lastTickAt) throw 402;

    // (3) Your business logic. Keep it CHEAP — you paid for gasLimit, not
    //     gasLimit × retries.
    doTheWork();

    storage.lastTickAt = now_ts;
    storage.tickCount += 1;
    storage.save();
}
```

## Gas budget — how much to put in `gasLimit`

The automaton forwards exactly `gasLimit` with each call. Under-sizing = silent revert, the automaton gets paid, your contract state doesn't advance.

```ts
// Measure in a sandbox test (see /titon:kronos-gas-sizing).
const result = await target.send(automaton.getSender(), {
    value: toNano('0.5'),
    body: beginCell().storeUint(OP_CRON_TICK, 32).endCell(),
});
const tx = result.transactions.find(t => t.inMessage?.info.dest?.equals(target.address));
const gasUsed = tx.description.computePhase.gasUsed;  // nano-TON
const recommended = (gasUsed * 120n) / 100n;           // 20% buffer
console.log(`gasLimit: toNano('${recommended.toString()}')`);
```

Rule of thumb for a very simple receiver (load + mutate + save): `toNano('0.02')` is safe. Anything that emits events, sends messages, or iterates a map needs `toNano('0.05')` – `toNano('0.1')`.

## Storing the Kronos address (once, at deploy)

```tolk
fun handleSetKronosRegistry(sender: address, newAddr: address) {
    var storage = lazy Storage.load();
    assert (sender == storage.owner) throw 100;   // E_NOT_OWNER
    assert (storage.kronosRegistry.isNone()) throw 101;  // one-shot pin
    storage.kronosRegistry = newAddr;
    storage.save();
}
```

One-shot pin matches how Kronos itself pins `forgeton`. Re-pointing = timelocked code upgrade or redeploy.

## Pitfalls — call out to the user

- **Do not `throw` to reject a cycle.** It reverts the whole receiver; the automaton still got paid; your watermark never advances. Return early silently instead.
- **Do not depend on `random()` for adversarial entropy.** The assigned automaton knows when Kronos will call. Use Fortuna (`/titon:fortuna-integrate-consumer`) for verifiable randomness.
- **Do not `bounceOnInvalid`.** Kronos sends `BounceMode.NoBounce`; a bounce would silently refund *you*, not the automaton.
- **Do not forget to fund `gasLimit` headroom for events.** `createExternalLogMessage` is ~1500 gas on its own; every `emitEvent` pushes the ceiling.
- **Do not check `sender == jobOwner`.** Kronos forwards from the *registry* — the job owner is the one who paid for the job, but they're not the sender. Use `sender == KRONOS_REGISTRY`.

## Testing with the SandboxKronos harness

```ts
import { SandboxKronos } from '@titon-network/kronos-sdk/testing';
import { Blockchain } from '@ton/sandbox';

const blockchain = await Blockchain.create();
const kronos = await SandboxKronos.deploy(blockchain, { owner: owner.address, treasury: owner.address });
const target = blockchain.openContract(MyTarget.createFromConfig({ owner: owner.address, kronosRegistry: kronos.address }, code));
await target.sendDeploy(owner.getSender(), toNano('0.5'));

const jobId = await kronos.register({
    target: target.address,
    message: beginCell().storeUint(OP_CRON_TICK, 32).endCell(),
    interval: 300,
    reward: toNano('0.05'),
    gasLimit: toNano('0.02'),
    via: jobOwner.getSender(),
});

await kronos.fastForward(300);
const result = await kronos.executeNext(jobId, automaton.getSender());
expect(result.transactions).toHaveTransaction({
    to: target.address,
    success: true,
    op: OP_CRON_TICK,
});
```

See `/titon:kronos-integration-test` for the harness's full API.

## Related skills

- `/titon:kronos-gas-sizing` — empirically measure `gasLimit`.
- `/titon:kronos-integration-test` — full sandbox test pattern.
- `/titon:kronos-register-job` — register the job that calls this receiver.
- `/titon:kronos-scaffold` — one-shot generator: receiver + test + register script.
- `/titon:kronos-debug-exit-code` — triage what a reverted execution means.
