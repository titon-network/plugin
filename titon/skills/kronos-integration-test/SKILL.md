---
name: kronos-integration-test
description: Write a sandbox integration test for a Kronos-triggered contract using the SandboxKronos harness. Use when the user is testing their target contract end-to-end — deploy + register + fast-forward + execute + assert.
---

# Test a Kronos-triggered contract end-to-end

You're helping a dev write a jest test that: (1) deploys their Tolk target, (2) registers a Kronos job against it, (3) fast-forwards past `nextDue`, (4) fires `Execute`, (5) asserts state changed.

Without the harness that's ~30 lines of boilerplate per test. With the harness it's ~5.

## The `SandboxKronos` harness

```ts
import { Blockchain, type TreasuryContract, type SandboxContract } from '@ton/sandbox';
import { toNano, beginCell } from '@ton/core';
import { SandboxKronos } from '@titon-network/kronos-sdk/testing';
import '@ton/test-utils';

import { MyTarget } from '../wrappers/MyTarget';

const OP_CRON_TICK = 0x7001;

describe('MyTarget under Kronos', () => {
    let blockchain: Blockchain;
    let owner: SandboxContract<TreasuryContract>;
    let jobOwner: SandboxContract<TreasuryContract>;
    let automaton: SandboxContract<TreasuryContract>;
    let kronos: SandboxKronos;
    let target: SandboxContract<MyTarget>;

    beforeEach(async () => {
        blockchain = await Blockchain.create();
        owner = await blockchain.treasury('owner');
        jobOwner = await blockchain.treasury('jobOwner');
        automaton = await blockchain.treasury('automaton');

        // Deploy Kronos registry (and optionally ForgeTON + wire).
        kronos = await SandboxKronos.deploy(blockchain, { owner: owner.address, treasury: owner.address });

        // Deploy your target; pin the Kronos registry on it.
        target = blockchain.openContract(
            MyTarget.createFromConfig(
                { owner: owner.address, kronosRegistry: kronos.address },
                MyTarget.code,
            ),
        );
        await target.sendDeploy(owner.getSender(), toNano('0.5'));
    });

    it('fires cron tick on schedule', async () => {
        // Register a job targeting our contract.
        const jobId = await kronos.register({
            target: target.address,
            message: beginCell().storeUint(OP_CRON_TICK, 32).endCell(),
            interval: 300,
            reward: toNano('0.05'),
            gasLimit: toNano('0.02'),
            via: jobOwner.getSender(),
        });

        // First execution is permissionless — no fast-forward needed.
        const first = await kronos.executeNext(jobId, automaton.getSender());
        expect(first.transactions).toHaveTransaction({
            to: target.address,
            op: OP_CRON_TICK,
            success: true,
        });

        // Second execution — fast-forward past nextDue.
        await kronos.fastForward(301);
        const second = await kronos.executeNext(jobId, automaton.getSender());
        expect(second.transactions).toHaveTransaction({
            to: target.address,
            op: OP_CRON_TICK,
            success: true,
        });

        // Assert your target's state advanced.
        const snap = await target.getSnapshot();
        expect(snap.tickCount).toBe(2n);
    });
});
```

## Harness API reference

| Method | Purpose |
|--------|---------|
| `SandboxKronos.deploy(blockchain, { owner, treasury, withForgeton? })` | Deploys the registry (+ optional ForgeTON + wires them). Returns the harness. |
| `harness.address` | `Address` of the deployed registry. |
| `harness.client` | `KronosClient` facade — use for reads / decoding events. |
| `harness.register({ target, message, interval, reward, gasLimit, via, … })` | Registers a job, returns `jobId: bigint`. Auto-funds 100 runs. |
| `harness.fastForward(seconds)` | Bumps `blockchain.now`. |
| `harness.executeNext(jobId, viaSender)` | Sends `Execute(jobId)` with `value: toNano('0.5')`. Returns the sandbox result. |
| `harness.decodeEvents(result)` | Pulls externals from a result and decodes every Kronos event. |
| `harness.stakeAutomaton(via)` | Only if `withForgeton: true`. Stakes and registers the sender as an active automaton. |

## Common patterns

### Assert a specific event fired

```ts
const result = await kronos.executeNext(jobId, automaton.getSender());
const events = kronos.decodeEvents(result);
const executed = events.find(e => e.kind === 'JobExecuted');
expect(executed).toBeDefined();
expect(executed!.jobId).toBe(jobId);
```

### Drive multiple cycles

```ts
for (let i = 0; i < 5; i++) {
    await kronos.fastForward(301);
    await kronos.executeNext(jobId, automaton.getSender());
}
const snap = await target.getSnapshot();
expect(snap.tickCount).toBe(5n);   // first (permissionless) + 4 post-fast-forward runs
```

### Standalone mode (no operator set; for fast-iteration)

By default `SandboxKronos.deploy` runs in **standalone mode** — no operator pool wired up, every `Execute` is permissionless. This is the right default for testing your target contract's business logic. You don't need to know anything about staking or operator rotation to verify your receiver behaves correctly under the trigger.

If you want to additionally exercise the rotation/fallback flow (assigned-automaton windows), the harness supports a wired-pool mode — but that's an operator-side concern. For dapp-side correctness, the snapshot/state assertions above on `executeNext` cover the cases that matter.

## Memory caps

Sandbox specs OOM under parallel jest — the harness doesn't change that.

```ts
// jest.config.ts
export default {
    maxWorkers: 1,
    testTimeout: 30_000,
};
```

And run tests with `--runInBand`.

## Pitfalls — call out to the user

- **First execution is permissionless.** Don't expect `NotAssigned` on the first run regardless of rotation.
- **Fast-forward before `executeNext` for subsequent runs.** Otherwise you'll hit `TooEarly` (exit 129).
- **Auto-funding assumes 100 runs.** Override `funding: toNano('N')` in `register(...)` if you want a different horizon.
- **`withForgeton: false` skips all rotation.** `assignedAutomatonFor(jobId)` returns null until the pool is wired.

## Related skills

- `/titon:kronos-target-receiver` — write the receiver being tested.
- `/titon:kronos-gas-sizing` — measure realistic gas inside the same harness.
- `/titon:kronos-debug-exit-code` — triage a test failure by exit code.
- `/titon:kronos-scaffold` — generate a starting receiver + test in one shot.
