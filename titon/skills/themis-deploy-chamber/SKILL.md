---
name: themis-deploy-chamber
description: Permissionlessly deploy a Themis chamber for your protocol via DeployChamber (0x90) at the singleton factory. Use when the user says "deploy a Themis chamber", "set up a sealed-bid pipeline for my dapp", "create a chamber bound to my consumer", or "stand up Themis for my protocol".
---

# Deploy a chamber for your protocol

A `ThemisChamber` is the per-protocol commit-reveal pipeline. Every dapp using Themis gets its own chamber, deployed by anyone (you, your protocol's deployer, an end-user) by sending `DeployChamber (0x90)` to the singleton factory. The factory mints a fresh chamber instance with deterministic address, returns its address via the `EvtChamberDeployed (0xB0)` event, and from then on the chamber operates independently — owned by whoever you set as `chamberOwner`, dispatching `RevealCallback` to whatever address you set as `consumer`.

## Prerequisites

You need:
1. **The `ThemisFactory` address** for the network you're deploying on. See [`deployments.{network}.json`](../../../) in the parent repo for whichever networks have been deployed.
2. **A wallet with enough TON** to cover the factory's `deployChamberFee` + `minChamberInit` + gas — ~3 TON is safely over. The SDK does not expose a full `factory.getConfig()` getter (only `getRewardPool()`); use the SDK defaults (`DEFAULT_DEPLOY_CHAMBER_FEE = 0.5 TON`, `DEFAULT_MIN_CHAMBER_INIT = 0.5 TON`) as the expected floor, or over-attach.
3. **Your consumer contract's address** (your dapp). Can be addr_none initially if your consumer needs the chamber's address baked in — wire it afterwards via your consumer's bind-chamber message (see [`themis-integrate-consumer.md`](./themis-integrate-consumer.md) "Option A").
4. **A `groupId`** — `DEFAULT_GROUP_ID = 0` (Atlas has one group). Single-group is enforced; non-zero reverts.

## The DeployChamber message

```tolk
struct (0x00000090) DeployChamber {
    chamberOwner: address       // who owns the chamber post-deploy
    consumer:     address       // where RevealCallback (0x95) gets sent
    groupId:      uint8         // 0 = DEFAULT_GROUP_ID    params:       Cell<ChamberDeployParams>
}

// MIRROR-OF: themis-messages.tolk:DeployChamberParams — exactly 9 fields.
// The SDK's `ChamberDeployParams` TS type has the same 9 fields (no
// chamberOwner / consumer / groupId nested inside; those are at the OUTER
// `DeployChamberArgs` level).
struct ChamberDeployParams {
    commitDuration:  uint32     // seconds; how long bidders have to submit
    revealDuration:  uint32     // seconds; how long operators have to reveal
    submitFee:       coins      // per-bid fee, retained into chamber's rewardPool
    revealerReward:  coins      // paid to the first valid revealer
    advanceReward:   coins      // paid to whoever calls AdvanceRound after revealEta
    callbackGas:     coins      // attached value when dispatching RevealCallback
    maxBidsPerRound: uint32     // commit-window capacity; MUST be ≤ 32 (REFUND_BATCH_CAP)
    minReserve:      coins      // storage-rent floor; chamber asserts balance >= this
    minXcGas:        coins      // cross-contract gas floor (MIN_XC_GAS_FLOOR ~ 0.01 TON)
}
```

## TypeScript — drop-in deploy

```typescript
import { Address, toNano } from '@ton/core';
import { ThemisFactory, EVT_CHAMBER_DEPLOYED, decodeEvent } from '@titon-network/themis-sdk';

async function deployChamberForMyProtocol(opts: {
    factory:      ThemisFactory;
    deployer:     Sender;                  // any wallet
    chamberOwner: Address;                 // your protocol's owner
    consumer:     Address;                 // your dapp contract
}): Promise<{ chamber: Address; serial: bigint }> {
    const beforeSerial = await opts.factory.getNumChambers();

    // Wrapper signature: sendDeployChamber(via, value, DeployChamberArgs).
    // `chamberOwner` + `consumer` + `groupId` are at the OUTER level of args;
    // `params` is the inner 9-field tunable cell.
    const result = await opts.factory.sendDeployChamber(
        opts.deployer,
        toNano('3'),                          // value: generous; refunds excess
        {
            chamberOwner: opts.chamberOwner,
            consumer:     opts.consumer,
            groupId:      0,                  // DEFAULT_GROUP_ID — optional, defaults to 0
            params: {
                commitDuration:  60 * 5,               // 5 min
                revealDuration:  60 * 5,               // 5 min
                submitFee:       toNano('0.05'),
                revealerReward:  toNano('0.05'),
                advanceReward:   toNano('0.02'),
                callbackGas:     toNano('0.05'),
                maxBidsPerRound: 8,                    // wrapper throws if > 32
                minReserve:      toNano('0.1'),
                minXcGas:        toNano('0.01'),
            },
        },
    );

    // The factory emits EvtChamberDeployed with the new chamber's address.
    for (const ext of result.transactions.flatMap((t) => t.outMessages.values())) {
        if (!ext.info || ext.info.type !== 'external-out') continue;
        const ev = decodeEvent(ext.body);
        if (ev?.kind === 'ChamberDeployed' && ev.serial === beforeSerial) {
            return { chamber: ev.chamber, serial: ev.serial };
        }
    }
    throw new Error('DeployChamber did not emit EvtChamberDeployed');
}
```

(Sandbox tests in the parent repo use the `tests/helpers/deploy.ts:deployChamber()` helper that does this for you. Production deploys should mirror the pattern above with proper error-handling.)

## Choosing the tunables

The defaults from `DEFAULT_CHAMBER_PARAMS` (see `tests/helpers/deploy.ts`) are conservative testnet values. Production picks:

| Tunable | Default | Production tuning |
|---------|---------|-------------------|
| `commitDuration` | 60s | **5-15 min** for human-driven bidding; **10-30s** for high-frequency DEX swaps. Trade-off: longer = more accumulation, less staleness pressure. |
| `revealDuration` | 60s | **5-10 min** typically; should exceed your operator quorum's reveal-roundtrip latency by 3-5×. |
| `submitFee` | 0.05 TON | Cover gas + a small revealer reward. Bidders pay this per bid; it accumulates in the pool. |
| `revealerReward` | 0.05 TON | Operator's incentive to be first. Must be ≥ operator's gas cost. |
| `advanceReward` | 0.02 TON | Permissionless bounty — anyone advances a stuck round. Should cover gas + small profit. |
| `callbackGas` | 0.05 TON | Attached to the `RevealCallback` outbound. If your consumer is expensive, bump this. |
| `maxBidsPerRound` | 8 | Practical max ~32 before gas budget bites on `RevealRound`. Tune to your expected per-round volume. |
| `minReserve` | 0.1 TON | Storage-rent floor; chamber asserts balance ≥ this before outbound sends. 0.1 is comfortable for default storage. |
| `minXcGas` | 0.01 TON | Cross-contract gas floor; bumped if downstream consumers are very expensive. |

**Pool economics:** `submitFee × bidsPerRound` flows into the pool every commit window. `revealerReward + (maxBidsPerRound × submitFee_refund_in_advance_branch_A) + advanceReward` flows OUT. If you set `submitFee = revealerReward = 0.05`, the pool breaks even at ~1 reveal per `revealerReward / submitFee = 1` bid. Set `submitFee > revealerReward / maxBidsPerRound` for a profitable pool; the chamber owner can `WithdrawFees` from the surplus.

**Recommendation:** start with defaults on testnet. Measure actual bid volume + reveal latency. Tune up for mainnet.

## Wiring to your consumer

Two patterns — both detailed in [`themis-integrate-consumer.md`](./themis-integrate-consumer.md) under "Wire your consumer to a chamber":

### Option A — owner-only `SetBoundChamber` (recommended)

1. Deploy consumer with `isBound = false`.
2. Deploy chamber with `consumer:` set to your consumer's address.
3. Owner calls `SetBoundChamber` on the consumer with the new chamber's address.

### Option B — predict the chamber address (not yet wrapped)

Themis chamber addresses ARE deterministic from `(factory, serial, code)` on-chain — the factory derives them via `address.fromWorkchainAndHash(0, hashCell(stateInit))`. But **the SDK does not yet expose a `predictChamberAddress` helper**. The only chamber constructor on the wrapper is `ThemisChamber.createFromAddress(addr)`.

If you need address prediction today, replicate the factory's state-init derivation by hand (load `chamberCode` from `loadThemisChamberCode()` if/when it lands, hash `(parent, serial)` into the chamber's data cell, run `contractAddress(0, { code, data })`). Easier path: **use Option A**; the prediction helper lands in a later SDK release.

## After deploy

The chamber is **autonomous** post-deploy:

- Atlas's `GroupKeySync (0x51)` reaches it via the factory's fan-out. First time: bootstrap. Then: every Atlas rotation.
- ForgeTON's `AutomatonSync (0x1A)` reaches it via the factory's fan-out. Operators stake/unstake at ForgeTON → mirrored at the chamber.
- Anyone may `SubmitCiphertext` during commit windows.
- Operators race to submit `RevealRound`.
- Anyone may `AdvanceRound` post-`revealEta`.
- Owner may `Pause`, `Unpause`, `WithdrawFees`, `UpdateConfig`.
- The chamber is NOT directly upgradeable — child code upgrades happen at the factory level via `ProposeChildCodeUpgrade`, which only affects FUTURE chamber deploys.

You don't need to do anything to "start" the chamber. The first `EvtRoundStarted` fires on its first `AdvanceRound` (or implicitly on the first reveal+advance cycle).

### Tell the operators (critical — your chamber is dead until they opt in)

A deployed chamber receives ciphertexts fine, but **reveals don't happen
automatically** — at least one staked, mirrored automaton operator has
to opt into serving your chamber. There is no auto-discovery on the
operator side; operators add chamber addresses to their daemon config
explicitly. If you skip this step, every round expires unrevealed and
`AdvanceRound` refunds your bidders — the protocol is alive but
useless to your dapp.

After your `terraform apply` / chamber-deploy script lands, do one of:

1. **Run your own operator** — install `@titon-network/automaton`,
   stake at ForgeTON, register a BLS share at Atlas, add your chamber
   to `config.themis.chambers`. See
   [`../../../automaton/docs/quickstart.md`](../../../automaton/docs/quickstart.md)
   §"Optional — enable Themis". Cheapest at Lightsail (~$5/mo).
2. **Recruit existing operators** — publish your chamber address to
   the operator community (e.g. the Titon Discord / operator chat).
   Existing automatons add it to `themis_chambers` in their terraform
   var or daemon config and `make apply` to serve it. Each operator
   earns `revealerReward` per round they reveal.

Verification that an operator is serving your chamber: call
`chamber.getOperator(<operatorAddr>)` — returns `{ isActive: true }` once
the factory's `AutomatonSync` has fanned out (bounded by
`cfg.maxFanoutPerSync`; usually within a few cycles after the operator
stakes / your chamber deploys, whichever was last).

> ⚠️ **Fresh chambers start with an empty operator map** (factory's
> `syncCursor` only advances on incoming `AutomatonSync`, not on chamber
> deploy). A freshly-deployed chamber stays
> unmirrored until ForgeTON emits a new `AutomatonSync (0x1A)`, which
> ONLY fires on `Register` / `Unstake-finalize` / `Slash` — NOT on
> `IncreaseStake` / `RequestUnstake` / `CancelUnstake` / opt-in /
> opt-out. So if your operators are already registered + active and
> haven't churned recently, your fresh chamber will stay unmirrored
> until either:
>
> - A new operator registers (or an existing one fully unstakes / gets
>   slashed) — natural propagation, can take days/weeks on a quiet
>   network.
> - Or the ForgeTON owner runs `forgeton.sendForceSync({ automaton:
>   <op> })` once per operator you want mirrored at your chamber.
>   Owner-only; operators can NOT trigger this themselves.
>
> For testnet smoke: bootstrap with `forgeton.sendForceSync` from the
> ForgeTON-owner wallet (`forgeton/.env`'s `WALLET_MNEMONIC` for the
> team's testnet deployment). For mainnet, plan ahead — coordinate with
> the team to run `ForceSync` immediately after your first chamber
> deploys, so operators can serve your reveals on day one.

## Owner ops post-deploy

```typescript
// Pause new submissions + new reveals (operator-side stays open).
await chamber.sendPause(ownerVia, toNano('0.05'));

// Resume.
await chamber.sendUnpause(ownerVia, toNano('0.05'));

// Withdraw the surplus from the reward pool.
// Positional signature: sendWithdrawFees(via, value, to, amount).
const cfg = await chamber.getConfig();         // exposes rewardPool
await chamber.sendWithdrawFees(
    ownerVia,
    toNano('0.05'),                            // value (gas)
    ownerAddress,                              // to — MUST NOT be the chamber's own address
    cfg.rewardPool - toNano('0.1'),            // amount; leave some for future revealer rewards
);
```

(More owner ops in `themis-owner-ops.md` when written — pause/unpause/withdraw/UpdateConfig + chamber-code upgrade flow.)

## Pre-flight checklist

Before sending `DeployChamber` in production:

- [ ] Factory is admitted at ForgeTON (run `forgeton.getConsumer(factoryAddress)` — `isActive: true`).
- [ ] Factory is admitted at Atlas (run `atlas.getVerifier(factoryAddress)` — `isActive: true`, otherwise no GroupKeySync reaches your chamber).
- [ ] `groupId` is `DEFAULT_GROUP_ID = 0` (single-group is enforced).
- [ ] `consumer` address is set and lives on workchain 0.
- [ ] `chamberOwner` is a wallet you control with enough TON for ongoing pause/config/withdraw ops.
- [ ] Tunables match your protocol's volume + latency profile (see "Choosing the tunables" above).
- [ ] At least one operator is mirrored at the factory (otherwise reveals will fail with `E_OPERATOR_NOT_FOUND (170)`).
- [ ] Attached value covers `cfg.deployChamberFee + cfg.minChamberInit + ~0.5 TON gas` (3 TON is safely over).

## Watch out for

- **Don't deploy chambers in bulk without monitoring fan-out cycles.** The factory's fan-out cursor is bounded by `cfg.maxFanoutPerSync` (default 16). With many chambers, an `AutomatonSync` from ForgeTON propagates over several txs. Until the cycle completes, new chambers might miss the latest state. `EvtFanoutCycleComplete (0xB7)` signals end-of-cycle.
- **Don't set `commitDuration < revealDuration / 5`.** Operators need time to threshold-decrypt every bid in the round. Too-tight a commit window relative to reveal capacity = chronically missed reveals = chronic refunds.
- **Don't deploy a chamber expecting to update its `parent`.** The chamber's `parent: address` is **immutable post-deploy**. To repoint, you'd need a new chamber.
- **Don't deploy without setting `consumer`.** `RevealRound` reverts with `E_CONSUMER_NOT_SET (173)` if `consumer == addr_none`. Either deploy with the consumer wired or `Pause` until you can wire it.
- **Don't try to send `DeployChamber` directly without the wrapper.** The params cell layout is finicky (9 fields, mixed `uint32` / `coins`); use `factory.sendDeployChamber(via, value, args)` and let the wrapper serialize. The wrapper also throws on `maxBidsPerRound > 32`, which the contract enforces server-side; catching that mistake client-side saves a tx + revert.
- **Don't reuse a `(chamberOwner, consumer, params)` tuple expecting deduplication.** Themis allows multiple chambers with identical params — the factory increments `serial` on each deploy. If you want one-per-protocol, gate at your dapp layer.

## Cross-references

- [`AGENTS.md`](../AGENTS.md) — full SDK surface.
- [`skills/themis-integrate-consumer.md`](./themis-integrate-consumer.md) — write the consumer that this chamber dispatches callbacks to.
- [`skills/themis-bidder-flow.md`](./themis-bidder-flow.md) — bidder UX for submitting to this chamber.
- [`skills/themis-debug.md`](./themis-debug.md) — error-code lookup for deploy failures.
- [`tests/Factory.spec.ts`](../../../tests/Factory.spec.ts) — `DeployChamber` sandbox specs; production-mirroring.
