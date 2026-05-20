---
name: phoebe-deploy-and-admit
description: Deploy a fresh Phoebe instance and wire it to its peers — pin owner/forgeton/atlas at deploy, admit Phoebe as an Atlas verifier, admit Phoebe as a ForgeTON consumer, then verify. Use when the user says "deploy Phoebe", "set up Phoebe on testnet", "admit Phoebe as a verifier", "wire Phoebe to its peers", or "stand up a new oracle".
---

# Deploy Phoebe and admit it at its peers

You're standing up a new Phoebe deployment. Three contracts need to know about each other:

- **Phoebe** — pinned to one ForgeTON + one Atlas at deploy (immutable post-deploy).
- **ForgeTON** — admits Phoebe as a consumer; thereafter mirrors operator state into Phoebe.
- **Atlas** — admits Phoebe as a verifier; thereafter fans out group-key state into Phoebe.

The order matters: deploy → admit at Atlas (bootstrap groupPk) → admit at ForgeTON (mirror operators).

## Prerequisites

- Funded owner wallet (≥ 5 TON for deploy + admit + initial reward-pool seed).
- Live Atlas + ForgeTON addresses for the target network (`@titon-network/atlas-sdk` + `@titon-network/forgeton-sdk` ship the constants).
- Sandbox-test your specific deploy script before sending real TON.

## Step 0 — sanity-check the bundled artifact

```bash
npx phoebe hash
# → hex:    <bundled-code-hash>
#   base64: <bundled-code-hash-b64>
```

Compare against the contract you reviewed. If you're deploying from the in-repo build (not the published SDK), run `pnpm run build` from the Phoebe repo first.

## Step 1 — derive the address + deploy

```ts
import { Address, toNano } from '@ton/core';
import {
    newPhoebe,
    loadPhoebeCode,
    Phoebe,
    PHOEBE_DEFAULTS,
} from '@titon-network/phoebe-sdk';
import { ATLAS_TESTNET }    from '@titon-network/atlas-sdk';
import { FORGETON_TESTNET } from '@titon-network/forgeton-sdk';

const ownerAddr    = Address.parse('0Q...');
const forgetonAddr = FORGETON_TESTNET!.forgeton;
const atlasAddr    = ATLAS_TESTNET!.atlas;

// `newPhoebe` is the factory shortcut — equivalent to `Phoebe.createFromConfig`
// with `loadPhoebeCode()` and the default tunables.
const phoebe = tonClient.open(newPhoebe({
    owner:    ownerAddr,
    forgeton: forgetonAddr,
    atlas:    atlasAddr,
    // tunables: { pullFee: toNano('0.02'), maxPushDrift: 300 },    // override defaults if needed
}));

// Compute the Phoebe address BEFORE sending the deploy — useful for funding
// the address ahead of time + for recording in your deployments.json:
const phoebeAddr = phoebe.address;
console.log('Phoebe will deploy at', phoebeAddr.toString({ testOnly: true, bounceable: false }));

// Send the deploy. Generous funding — the contract holds rewardPool
// (accumulated pull fees) + minStorageReserve (0.1 TON default).
await phoebe.sendDeploy(ownerSender, toNano('1'));
```

> **Manual init.** If you prefer not to use `newPhoebe`, the equivalent is `Phoebe.createFromConfig({ owner, forgeton, atlas, tunables: PHOEBE_DEFAULTS }, loadPhoebeCode())`.

## Step 2 — admit Phoebe at Atlas (Atlas owner sends)

This triggers the bootstrap `GroupKeySync` fan-out into Phoebe's cache.

```ts
import { Atlas } from '@titon-network/atlas-sdk';

const atlas = tonClient.open(Atlas.createFromAddress(atlasAddr));

await atlas.sendSetVerifier(atlasOwnerSender, {
    value:    toNano('0.5'),       // covers Atlas compute + per-verifier fan-out
    contract: phoebeAddr,
    isActive: true,
});
```

Verify Phoebe cached the key:

```ts
const groupKey = await phoebe.getGroupKey();
console.log(groupKey);
// → { groupId: 0, groupEpoch: 1, threshold: 1, memberCount: 1,
//     groupPk: <48-byte buffer>, cachedAt: <ts> }
```

If `groupKey === null`, Atlas hasn't fanned out yet — owner can manually re-trigger:

```ts
await phoebe.sendSyncAtlas(ownerSender, { value: toNano('0.1') });
```

## Step 3 — admit Phoebe at ForgeTON (ForgeTON owner sends)

This fans out `AutomatonSync` for every active operator into Phoebe's mirror.

```ts
import { ForgeTON } from '@titon-network/forgeton-sdk';

const forgeton = tonClient.open(ForgeTON.createFromAddress(forgetonAddr));

await forgeton.sendSetConsumer(forgetonOwnerSender, {
    value:    toNano('0.1'),
    contract: phoebeAddr,
    isActive: true,
});
```

Active operators are now mirrored. Verify by polling Phoebe:

```ts
const op = await phoebe.getOperator(someOperatorAddr);
console.log(op);    // { isActive: true } if mirrored
```

## Step 4 — verify the deployment end-to-end

```bash
npx phoebe info <phoebe-addr> --testnet
```

Expected output:

```
phoebe @ 0:abcd…
  owner:    0:dead…
  forgeton: 0:beef…
  atlas:    0:cafe…
  schema:   storage=1 config=4
  pullFee:  0.0100 TON
  reserve:  0.1000 TON  (rewardPool: 0.0000 TON)
  drift:    ±300s
  groupKey: epoch=1 t=1/n=1 pk=AABBCCDDEE…
  snapshot: NEVER PUSHED
```

`snapshot: NEVER PUSHED` is correct — operators haven't pushed yet. Once the first push lands, `snapshot:` shows the timestamp + root.

For automated drift check:

```bash
npx phoebe verify --testnet
```

## Sandbox dry-run before mainnet

Always exercise the full deploy → admit → push → pull pipeline in-process first:

```ts
import { Blockchain } from '@ton/sandbox';
import { deployPhoebeFixture } from '@titon-network/phoebe-sdk/testing';

const blockchain = await Blockchain.create();
const fx         = await deployPhoebeFixture(blockchain);

// fx.phoebe, fx.atlas, fx.forgeton are all wired + admitted.
// fx.pushSnapshot({ leaves }) lets you exercise the operator side.
// fx.requestPrice({ tree, feedId, leaf }) lets you exercise the consumer side.
```

If the fixture works, your deploy script should work — the fixture uses the same `loadPhoebeCode()` + `Phoebe.createFromConfig` path under the hood.

## Watch out for

- **Address pinning is permanent.** `storage.forgeton` and `storage.atlas` are immutable post-deploy. Repointing requires a timelocked code upgrade. Triple-check the addresses before sending.
- **Workchain mismatch.** All three contracts MUST be on the same workchain (typically 0). A cross-workchain fan-out fails with action 36 at the sender.
- **Bootstrap order matters.** Step 2 (admit at Atlas) MUST happen before step 3 (admit at ForgeTON). Without a cached `groupPk`, operator pushes will hit `E_GROUP_KEY_NOT_SET` (180).
- **Mainnet checklist.** Never deploy to mainnet without: (a) external audit cleared (Phoebe targets TSA's zero-findings posture matching Fortuna / Atlas / ForgeTON / Kronos); (b) 24h timelocked upgrade in place; (c) ≥ 3 operators committed.
- **The owner is not the operator.** The wallet that signs `sendDeploy` becomes Phoebe's owner (admin ops, code upgrade, fee withdrawal). Operators are mirrored via ForgeTON; they only need to stake there.

## Common follow-up tasks

- **Symbols / feed display** — Phoebe doesn't carry an on-chain feed registry. Symbol display lives in operator-published config files (consumers fetch the `feedId → symbol` map from the operator's URL). Document your registry next to your operator endpoint.
- **First push** — once admitted at both peers, the operator daemon (`automaton` with `phoebe` product enabled) starts heartbeating snapshots. First push lands at `EvtSnapshotPushed`.
- **First pull** — once a snapshot is on-chain, consumers can call `RequestPrice`. Smoke-test with the SDK's bundled CoinFlip-equivalent (the price-aware-vault example).

## Cross-references

- `RECIPES.md#1` — the canonical deploy-and-admit walkthrough in the SDK docs.
- `phoebe-integrate-consumer.md` SDK skill — what dapps do once Phoebe is live.
- `phoebe-debug.md` SDK skill — when the admit or first push doesn't land.

## Related skills

- `/titon:phoebe-integrate-consumer` — what to build on top once Phoebe is live.
- `/titon:phoebe-handle-event` — set up an indexer / heartbeat-stall alert at deploy time.
- `/titon:phoebe-debug-revert` — diagnose admit failures (200/201/130/131) or first-push failures (180/161).
