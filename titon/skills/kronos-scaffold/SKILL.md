---
name: kronos-scaffold
description: Scaffold the full Kronos integration for a new target contract — receiver Tolk snippet, sandbox test, deploy-and-register script. Use when the user wants a one-shot "make this contract Kronos-triggered" action from {contract-path, opcode, interval}.
---

# Scaffold Kronos integration into an existing contract

You're helping a dev **bolt Kronos onto their existing contract** in one pass. Given:

- A Tolk contract path (the target).
- An opcode they want Kronos to dispatch to.
- A cadence (interval seconds + reward).

You generate: (1) the receiver snippet to paste into the contract, (2) a sandbox test file, (3) a deploy + register script. Then point them at the right follow-up skills.

## Ask the user

| Field | What to extract |
|-------|-----------------|
| Target contract path | e.g. `contracts/my-staker.tolk`. Read it to understand the existing dispatch pattern. |
| Opcode name | PascalCase, e.g. `CronTick`. Suggest a hex value in the `0x7000`-`0x7FFF` user range. |
| Interval | Seconds. Snap to sane values (60, 300, 3600, 86400). |
| Reward | Default `toNano('0.05')`. |
| Gas limit | Default `toNano('0.02')` (simple receivers) — flag that they should measure via `/titon:kronos-gas-sizing`. |
| Kronos registry address | If deploying fresh, scaffold a deploy-and-wire step. Otherwise use `KRONOS_TESTNET.registry` from the SDK. |

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

See `/titon:kronos-target-receiver` for the full pattern (idempotence, gas budget, pitfalls).

## Step 2 — Sandbox test (`tests/Kronos.spec.ts`)

```ts
import { Blockchain, type TreasuryContract, type SandboxContract } from '@ton/sandbox';
import { toNano, beginCell } from '@ton/core';
import { SandboxKronos } from '@titon-network/kronos-sdk/testing';
import '@ton/test-utils';

import { MyTarget } from '../wrappers/MyTarget';

const OP_CRON_TICK = 0x7001;

describe('MyTarget Kronos cron', () => {
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
        kronos = await SandboxKronos.deploy(blockchain, { owner: owner.address, treasury: owner.address });

        target = blockchain.openContract(
            MyTarget.createFromConfig(
                { owner: owner.address, kronosRegistry: kronos.address },
                MyTarget.code,
            ),
        );
        await target.sendDeploy(owner.getSender(), toNano('0.5'));
    });

    it('cron tick updates state', async () => {
        const jobId = await kronos.register({
            target: target.address,
            message: beginCell().storeUint(OP_CRON_TICK, 32).endCell(),
            interval: 300,
            reward: toNano('0.05'),
            gasLimit: toNano('0.02'),
            via: jobOwner.getSender(),
        });

        const result = await kronos.executeNext(jobId, automaton.getSender());
        expect(result.transactions).toHaveTransaction({
            to: target.address,
            op: OP_CRON_TICK,
            success: true,
        });

        const snap = await target.getSnapshot();
        expect(snap.lastTickAt).toBeGreaterThan(0);
    });
});
```

## Step 3 — Deploy + register script (`scripts/registerCron.ts`)

Blueprint style — works with `blueprint run registerCron`:

```ts
import { NetworkProvider } from '@ton/blueprint';
import { Address, beginCell, toNano } from '@ton/core';
import { KronosClient, KronosRegistry, registerJobOpts, KRONOS_TESTNET } from '@titon-network/kronos-sdk';

const OP_CRON_TICK = 0x7001;

export async function run(provider: NetworkProvider) {
    const registry = provider.open(KronosRegistry.createFromAddress(KRONOS_TESTNET.registry));
    const client = new KronosClient({ registry });

    const target = Address.parse(await provider.ui().input('Target address'));
    const cfg = await client.jobs.config();

    const opts = registerJobOpts(
        {
            target,
            message: beginCell().storeUint(OP_CRON_TICK, 32).endCell(),
            interval: 300,
            reward: toNano('0.05'),
            gasLimit: toNano('0.02'),
        },
        cfg,
    );

    await client.jobs.register(provider.sender(), opts);
    console.log(`Registered. Watch for JobRegistered event to get the jobId.`);
}
```

For mainnet, swap `KRONOS_TESTNET` once it's replaced by `KRONOS_MAINNET`.

## Checklist before running

- [ ] Tolk receiver compiles: `pnpm build` (or `blueprint build`).
- [ ] Test passes: `pnpm test` or `jest --runInBand`.
- [ ] Measure actual `gasLimit`: see `/titon:kronos-gas-sizing` — probably higher than the `0.02` default.
- [ ] Pick a stable opcode in `0x7000`-`0x7FFF`. If your contract already uses anything up there, don't collide.
- [ ] Target contract has a way to pin `kronosRegistry` at deploy (storage field + owner-only setter).

## What the skill does NOT generate

- **The `MyTarget` wrapper (`wrappers/MyTarget.ts`).** That's target-specific. If Blueprint is in use, it generates a stub; otherwise hand-write the wrapper from the SDK shape.
- **The actual business logic in `doTheWork()`.** Your job.
- **Mainnet deployment.** Scaffold targets testnet by default; the user is responsible for audit + mainnet cutover.

## Related skills

- `/titon:kronos-target-receiver` — the receiver pattern in detail.
- `/titon:kronos-gas-sizing` — measure `gasLimit` properly before shipping.
- `/titon:kronos-integration-test` — the full `SandboxKronos` harness API.
- `/titon:kronos-register-job` — what `registerJobOpts` does under the hood.
