---
name: phoebe-debug-revert
description: Diagnose a failed Phoebe transaction by exit code. Use when a Phoebe tx reverted with a numeric exit code (e.g. 161, 180, 181, 183, 187, 240), when the user pastes a trace and says "Phoebe reverted", "BLS_VERIFY failed", "stale snapshot", "bad merkle proof", "FulfillPrice never landed", or asks "what does Phoebe error X mean?".
---

# Diagnose a Phoebe exit code

You're helping a user understand why a Phoebe transaction reverted. Phoebe's `explainError` covers Phoebe codes (100–249), cross-contract auth codes (200/201), TVM commons, and the upgrade-timelock range.

## Get the code

The exit code lives at:
- Sandbox: `tx.description.computePhase.exitCode`
- TonClient trace: `compute_ph.exit_code`
- tonscan / tonviewer: shown on the failed tx page

If you have a sandbox `Transaction`:

```ts
import { summarizeTx, formatTxSummary } from '@titon-network/phoebe-sdk';

for (const tx of result.transactions) {
    console.log(formatTxSummary(summarizeTx(tx)));
}
```

If you have a thrown SDK error:

```ts
import { PhoebeError } from '@titon-network/phoebe-sdk';

try { ... } catch (e) {
    if (e instanceof PhoebeError) {
        console.error(`[${e.code}] ${e.name}: ${e.message}`);
        if (e.hint) console.error(`hint: ${e.hint}`);
    }
}
```

## Explain it

```ts
import { explainError } from '@titon-network/phoebe-sdk';

const e = explainError(161);
console.log(`${e.origin}:${e.name} — ${e.message}`);
if (e.hint) console.log(`hint: ${e.hint}`);
```

CLI shortcut: `npx phoebe explain 161`.

Full table: [`ERRORS.md`](https://github.com/titon-network/phoebe/blob/main/sdks/typescript/ERRORS.md).

## Phoebe error ranges (current)

| Range | Class | Codes |
|---|---|---|
| 100–109 | Access / lifecycle | 100 NotOwner, 101 OperationPaused, 102 NotPaused, 103 AlreadyPaused |
| 110–119 | Schema / version | 110 BadSchemaVersion |
| 130–131 | Group / threshold | 130 GroupIdNotSupported, 131 InvalidThresholdConfig |
| 140–149 | Operator + group-key monotonicity | 140 OperatorNotFound, 141 OperatorNotActive, 142 GroupEpochNotMonotonic, 143 GroupKeyChangedAtEpoch, 144 WrongDeploymentBinding |
| 160–169 | BLS / crypto | 161 InvalidBlsSignature, 162 InvalidG1Point |
| 180–193 | Phoebe-specific (snapshot / proof / claim / liveness) | 180 GroupKeyNotSet, 181 StaleSnapshot, 182 BadTimestampDrift, 183 BadMerkleProof, 186 FeedIdOutOfRange, 187 FreshNotFresher, 188 NoUnclaimedReward, 189 InvalidOperatorBps, 190 CriticalStaleness, 191 InvalidCriticalStaleness, 192 EvictionNotConfigured, 193 InvalidMaxInactivity |
| 200–209 | Cross-contract auth | 200 NotForgeton (sender ≠ pinned forgeton on AutomatonSync), 201 NotAtlas (sender ≠ pinned atlas on GroupKeySync) |
| 220–229 | Upgrade timelock | 225 UpgradeEtaTooSoon, 226 UpgradeNotReady, 227 UpgradeAlreadyPending, 228 NoPendingUpgrade, 229 EmptyCodeCell |
| 240–249 | Reserve / value | 240 InsufficientValue, 241 ReserveInvariant, 242 AmountExceedsAccumulated, 243 BadWithdrawTarget |
| 333 | TEP convention | 333 WrongWorkchain |
| 0xFFFF | Dispatcher | UnknownOpcode |

## Most common reverts dapp authors hit

### **161 InvalidBlsSignature** — push or mode-B pull reverted post-entry
The BLS aggregate doesn't verify against cached `groupPk`. Causes in order of likelihood:
1. **Wrong DST off-chain.** Use `signMessage(sk, hashBytes)` from the SDK; `@noble/curves` raw defaults to `_NUL_` which Phoebe rejects.
2. **Hash domain drift.** `computeSnapshotHash(phoebeAddress, timestamp, root)` must be called with the EXACT (address, ts, root) you're submitting.
3. **Stale pkShare.** Operator's share is from a prior Atlas rotation epoch; re-register after rotation.
4. **Wrong Phoebe address signed for.** See 144 below — operator endpoint may be signing for a different Phoebe.

For mode-B pulls (consumer-side): the operator endpoint served a bad sig. Re-fetch from a healthy operator; verify their endpoint isn't on a stale group epoch.

### **180 GroupKeyNotSet** — push or mode-B pull reverted on a fresh deploy
Atlas hasn't fanned out the bootstrap `GroupKeySync` to Phoebe. Owner-side:

```ts
await phoebe.sendSyncAtlas(ownerSender, { value: toNano('0.1') });
```

Then verify: `await phoebe.getGroupKey()` should return non-null. Or wait for Atlas's automatic fan-out (triggered by `SetVerifier` admission at Atlas).

### **181 StaleSnapshot** — push OR pull reverted
**On push:** `timestamp <= storage.lastSnapshotTime`. Another operator pushed first; use a strictly newer timestamp.
**On pull:** `maxStaleness` exceeded against the active root. Set `maxStaleness=0` to accept any active root, OR wait for a fresh push, OR attach a mode-B `freshUpdate` to satisfy a tight bound.

### **183 BadMerkleProof** — pull reverted post-entry
Three causes:
1. **Stale proof.** Every push rotates `lastRoot` and invalidates prior proofs. Rebuild with `PhoebeMerkleTree.fromSparseLeaves(latestLeaves).proof(feedId)` against the active root.
2. **Tampered leaf.** Pass the same `PriceLeaf` you used to build the tree — no mutation between build + send.
3. **feedId mismatch.** `leaf.feedId` MUST equal `msg.feedId` (the request's `feedId` field).

For mode-B specifically: build the proof against `freshUpdate.root`, NOT the cached `lastRoot`.

### **187 FreshNotFresher** — mode-B pull reverted post-entry
`freshUpdate.timestamp <= cached lastSnapshotTime`. The fresh-update must be strictly newer. Common causes:
- Operator endpoint is lagging — re-fetch from a healthier operator.
- Another consumer's mode-B pull beat yours into the same block (strict-monotonic; second consumer at same ts reverts).
- An operator's heartbeat push beat you. 

Always re-check `phoebe.getSnapshot().lastSnapshotTime` right before sending; fall back to mode A when fresh isn't fresher. See [`/titon:phoebe-pull-fresh-price`](/titon:phoebe-pull-fresh-price) for the defensive pattern.

### **144 WrongDeploymentBinding** — push OR mode-B pull reverted pre-BLS-verify
Signed payload's `phoebeHash` doesn't match this Phoebe's address-hash. Either:
- Operator endpoint is signing for a DIFFERENT Phoebe deployment (mis-config).
- Someone is replaying a sig from a sibling Phoebe sharing the same Atlas group.

Pass the **destination** `Address` to `computeSnapshotHash`. For consumers: re-fetch from the right operator endpoint for THIS Phoebe.

### **240 InsufficientValue** — almost any receiver reverted on entry
Attached `value` below the receiver's floor:
- `RequestPrice` (mode A): `cfg.pullFee + MIN_GAS_FOR_PULL`. Use `estimatePullValue({ pullFee })`.
- `RequestPrice` (mode B): add `withFresh: true` (+0.05 TON for BLS_VERIFY budget).
- `PushSnapshot`: `MIN_GAS_FOR_PUSH` (~0.05 TON). Use `estimatePushValue()`.
- `ClaimReward`: `MIN_GAS_FOR_CLAIM` (~0.03 TON). Attach 0.05 TON for slack.

Compute precisely: `npx phoebe info <addr>` to see live `pullFee`; multiply with the SDK's estimator.

### **182 BadTimestampDrift** — push or mode-B pull reverted
`|now - timestamp| > maxPushDrift` (300s default). Causes:
- Operator clock skew — sync via NTP.
- Operator endpoint serves a stale fresh-update — re-fetch.
- `maxPushDrift` is too tight for your network's jitter — owner can `UpdateConfig` to raise it.

### **190 CriticalStaleness** (v3) — mode-A pull rejected
Cached root is older than the owner-set `criticalStaleness` window. The contract refuses to serve a stale price. Options:
- Attach a mode-B `freshUpdate` to push the cache forward in the same tx.
- Wait for an operator to push a fresh heartbeat.
- Check `phoebe.getSnapshot()` to confirm the gap — if no operators are pushing, escalate operator-side (see [`/titon:phoebe-handle-event`](/titon:phoebe-handle-event) heartbeat-stall detection).

### **200 NotForgeton / 201 NotAtlas** — cross-contract inbound rejected
Sender ≠ pinned `storage.forgeton` / `storage.atlas`. Phoebe pins both at deploy; repointing requires a code upgrade. Check `phoebe.getOwnership()`; verify `deployments.testnet.json` matches.

### **188 NoUnclaimedReward** — `ClaimReward` rejected
Operator's accrued balance is zero (never accrued, or just drained). Query `phoebe.getUnclaimedReward(addr)` first; only claim when it's > 0. Note: accrual is **pull-time** (when consumers pull), not push-time — if you pushed but no consumer has pulled yet, you have no balance.

## TVM-level errors (not Phoebe-specific)

- **9 CellUnderflow** — wire-struct widths off. Re-copy `RequestPrice` / `FulfillPrice` / `PriceLeaf` from `@titon-network/phoebe-sdk`'s `templates/consumer.tolk`.
- **13 OutOfGas** — bump `value` on the send. For the consumer's `FulfillPrice` handler, increase value on the originating `RequestPrice`.
- **37 NotEnoughTon** — contract balance below reserve invariant. Owner-side: plain transfer to top up.
- **0xFFFF UnknownOpcode** — hit Phoebe's dispatcher `else` branch. Check op constants from `OP` in `@titon-network/phoebe-sdk`.

## "Pull worked but FulfillPrice never landed at my consumer"

Symptom: `RequestPrice` succeeded; consumer's `handleFulfillPrice` never fired.

1. **Consumer reverts on receive.** Phoebe sends with `bounce: false` — revert at consumer does NOT bounce back. Phoebe-side state is already finalized (pullFee retained). Check the consumer's tx exit code.
2. **Wrong opcode.** Consumer's receiver must be `struct (0x00000072) FulfillPrice`. Wrong opcode → 0xFFFF at the consumer.
3. **Wrong workchain.** Phoebe + consumer on different workchains → action 36 at Phoebe's outbound. Pin both to workchain 0.
4. **Insufficient consumer-side gas.** Phoebe forwards what's left after `pullFee` is retained. Heavy callback + tight value → exit 13 (OutOfGas) at consumer. Increase `value` on the originating `RequestPrice`.

## Wrap and re-throw

```ts
import { explainError, PhoebeError } from '@titon-network/phoebe-sdk';

try {
    await phoebe.sendRequestPrice(via, opts);
} catch (raw) {
    const code = extractExitCode(raw);    // your helper
    throw new PhoebeError(code, `RequestPrice failed for queryId=${queryId}`);
}
```

`PhoebeError` has `.code`, `.origin`, `.name`, `.message`, `.hint`.

## Useful CLI

```bash
npx phoebe explain 161                    # what is InvalidBlsSignature
npx phoebe info <addr> --testnet          # live state: pullFee, group key, last snapshot
npx phoebe verify --testnet               # SDK vs live-deploy drift check
npx phoebe decode <hex-or-base64>         # decode an event cell
```

## Watch out for

- **Don't retry blindly.** A failed push or pull burns gas. Diagnose the exit code first.
- **Don't ignore `NotForgeton` / `NotAtlas`.** These mean Phoebe was deployed pointing at the WRONG peer addresses — a deploy-time bug. Don't try to "fix" by re-syncing; you need a timelocked code upgrade to repoint.
- **Don't mix up `timestamp` and `groupEpoch`.** `timestamp` is unix seconds; `groupEpoch` is Atlas's monotonic rotation counter. Different concepts.
- **Don't trust an unauthenticated `FulfillPrice` body.** Sender-pin in your consumer is non-negotiable.

## Related skills

- `/titon:phoebe-integrate-consumer` — if the failure was on a consumer-side `RequestPrice`.
- `/titon:phoebe-pull-fresh-price` — if the failure was on a mode-B fresh-update.
- `/titon:phoebe-handle-event` — when the event log will help trace the root cause.
- `/titon:phoebe-deploy-and-admit` — if `NotForgeton`/`NotAtlas` points at a mis-bound deployment.
