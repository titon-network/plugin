---
name: kronos-gas-sizing
description: Empirically measure and size the gasLimit for a Kronos-triggered target contract. Use when the user is picking `gasLimit` for RegisterJob, sees silent Execute reverts, or wants to add a regression guard.
---

# Size `gasLimit` for a Kronos job

You're helping a dev pick the right `gasLimit` for a RegisterJob call. Undersizing is the #1 Kronos integration footgun: the automaton gets paid for sending but the target reverts out-of-gas — state doesn't advance and the user sees "why isn't my job running?"

## The fundamental rule

> **`gasLimit` is forwarded verbatim to the target.** The registry only enforces `gasLimit > 0`; it does not estimate for you. If the target needs 8500 gas and you registered 5000, every execution silently OOGs and you never hear about it on-chain.

## Three ways to size it

### 1. Use the SDK's `estimateJobGas` (fastest)

Two modes — pick the one that matches your state:

**a) `initCode` — fresh deploy in an isolated sandbox (one-liner, no setup):**

```ts
import { estimateJobGas } from '@titon-network/kronos-sdk';
import { beginCell, toNano } from '@ton/core';

const est = await estimateJobGas({
    initCode: MyTarget.code,         // compiled BoC
    initData: MyTarget.initialData,  // optional, defaults to empty cell
    body: beginCell().storeUint(OP_CRON_TICK, 32).endCell(),
});
console.log(`gasUsed=${est.gasUsed}  recommended=${est.recommended}`);
```

**b) `target` — measure against a contract you've already deployed on a sandbox you control:**

```ts
import { Blockchain } from '@ton/sandbox';

const blockchain = await Blockchain.create();
const target = blockchain.openContract(MyTarget.createFromConfig(...));
await target.sendDeploy(via, toNano('0.5'));

const est = await estimateJobGas({
    blockchain,                    // REQUIRED when using `target`
    target: target.address,
    body: beginCell().storeUint(OP_CRON_TICK, 32).endCell(),
});
```

Pass `blockchain` so the estimator runs on YOUR sandbox where the target exists. `target` without `blockchain` throws upfront — the estimator does not fork live mainnet state, so an isolated blockchain wouldn't have your target's code.

Both modes apply the default 20% buffer (override with `bufferPercent`) and return nano-TON bigints.

### 2. Measure in your existing sandbox test

```ts
import { Blockchain } from '@ton/sandbox';
import { toNano, beginCell } from '@ton/core';

const result = await target.sendInternal({
    from: KRONOS_REGISTRY_LIKE_ADDRESS,
    value: toNano('0.5'),
    body: beginCell().storeUint(OP_CRON_TICK, 32).endCell(),
});

const tx = result.transactions.find(t =>
    t.inMessage?.info.type === 'internal' &&
    t.inMessage.info.dest?.equals(target.address),
);
const gasUsed = tx!.description.type === 'generic' &&
    tx!.description.computePhase.type === 'vm'
    ? tx!.description.computePhase.gasUsed
    : 0n;

const recommended = (gasUsed * 120n) / 100n;
console.log(`gasUsed=${gasUsed}  recommended=${recommended}`);
```

Use this when your test already deploys the target and you want one inline measurement.

### 3. Back-calculate from a real testnet run

If the job is already live and you suspect OOG:

```ts
// Look at the last JobExecuted event — the reward is a deterministic pay-out,
// but the tx chain to the target can be inspected on tonviewer to read
// computePhase.gasUsed of the forwarded call. If gasUsed == gasLimit, you OOG'd.
```

Pair with `explainError(13)` — that's the TVM out-of-gas code.

## Pick a buffer — 20% is not magic

- **20%** — safe default for stable code paths.
- **40%** — paths with variable work (iterating a map, forking on balance, etc.).
- **Measure the worst case, not the average.** Sample with full state, not an empty sandbox.
- **Re-measure after changes.** Any logic edit to the receiver invalidates the estimate.

## Regression guard — pin the gas in a test

```ts
it('cron tick stays under gasLimit', async () => {
    const result = await target.sendInternal({
        from: KRONOS_REGISTRY_LIKE,
        value: toNano('0.5'),
        body: beginCell().storeUint(OP_CRON_TICK, 32).endCell(),
    });
    const tx = result.transactions.find(t => t.inMessage?.info.dest?.equals(target.address));
    const gasUsed = (tx!.description as any).computePhase.gasUsed as bigint;

    // Fail the build if the receiver grows past 80% of gasLimit.
    expect(gasUsed).toBeLessThan(toNano('0.016'));  // 80% of 0.02 TON gasLimit
});
```

This catches regressions: a refactor that bumps gas from 0.012 → 0.018 passes today but blows up at 0.025.

## What counts toward gasUsed

- **Storage reads** — cheap but not free. `lazy` helps (only materialises fields you read).
- **Storage writes** — the expensive one. Every `storage.save()` pays rent-style costs proportional to total bits.
- **Emitting events** — `createExternalLogMessage` is ~1500 gas on its own; every field adds ~50.
- **Outgoing sends** — ~500 gas per action, more if the value > 0.
- **Map operations** — O(log N) in cell depth. A map with 1k entries costs ~5x a map with 10.

## Pitfalls — call out to the user

- **Don't size based on an empty-state sandbox.** Real gas is higher once your maps/refs fill. Measure against realistic state.
- **Don't use `executionCost` for sizing.** That's the *automaton's* economics (reward + gas + fee). `gasLimit` is what your target receives, which is a separate number.
- **Don't pick `gasLimit = executionEconomics.totalCost`.** Those are incompatible units — one is "what the job spends", the other is "what the target gets".
- **Don't forget: 1 TON = 10^9 nano-TON.** `toNano('0.02')` = 20_000_000. Gas numbers in `gasUsed` are already in nano-TON.

## Related skills

- `/titon:kronos-target-receiver` — write the receiver that this gas sizes.
- `/titon:kronos-register-job` — plug the measured gasLimit into RegisterJob.
- `/titon:kronos-debug-exit-code` — `explainError(13)` for OOG symptoms.
