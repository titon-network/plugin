---
name: fortuna-debug-exit-code
description: Diagnose a failed Fortuna transaction by exit code. Use when a TON transaction reverted with a numeric exit code (e.g. 120, 124, 161, 240), when the user pastes a transaction trace, or when they ask "what does Fortuna error X mean?".
---

# Diagnose a Fortuna exit code

You're helping a user understand why a Fortuna transaction reverted. Fortuna's `explainError` covers Fortuna codes (100-249), cross-contract auth codes (200/201), and common TVM codes.

## Get the code

The exit code is at:
- Sandbox: `tx.description.computePhase.exitCode`
- TonClient trace: `compute_ph.exit_code`
- TON viewer / tonscan: shown on the failed tx page

If the user pasted a tx hash, walk them through `tonClient.getTransaction` to extract the code, OR ask them to grab it from tonviewer.

## The recipe

```ts
import { explainError } from '@titon-network/fortuna-sdk';
const e = explainError(124);
console.log(`${e.origin}:${e.name} — ${e.message}`);
if (e.hint) console.log(`hint: ${e.hint}`);
```

CLI shortcut: `npx fortuna explain 124`.

## Fortuna error code table (current)

| Range | Class | Codes |
|---|---|---|
| 100-109 | Access / lifecycle | 100 NotOwner, 101 Paused, 102 NotPaused |
| 110-119 | Schema / version | 110 BadSchemaVersion |
| 120-129 | **Request lifecycle** | 120 DuplicateRequest, 121 RequestNotFound, 122 DeadlineNotReached, 123 StaleEpoch, 124 GroupKeyNotSet, 125 TtlOutOfRange |
| 130-139 | Group ID (v2 forward-compat) | 130 GroupIdNotSupported |
| 140-149 | Operator / opt-in | 140 OperatorNotFound, 141 OperatorNotActive, 142 GroupEpochNotMonotonic, 143 OperatorNotOptedIn, 144 OperatorAlreadyOptedIn, 145 OperatorAlreadyOptedOut |
| 160-169 | BLS / crypto | 161 InvalidBlsSignature, 162 InvalidG1Point |
| 200-209 | Cross-contract auth | 200 NotForgeton (sender ≠ pinned forgeton on AutomatonSync), 201 NotAtlas (sender ≠ pinned atlas on GroupKeySync) |
| 220-229 | Upgrade timelock | 225 UpgradeEtaTooSoon, 226 UpgradeNotReady, 227 UpgradeAlreadyPending, 228 NoPendingUpgrade, 229 EmptyCodeCell |
| 240-249 | Reserve / value | 240 InsufficientValue, 241 ReserveInvariant, 242 AmountExceedsAccumulated, 243 BadWithdrawTarget |

## Most common reverts dapp authors hit

### **120 DuplicateRequest** — `RequestRandomness` reverted
You used the same `(consumer, queryId)` while a previous request was still pending.
- Use a monotone counter on-chain, OR `QueryIdStream` from the SDK off-chain.
- Wait for fulfillment / reclaim before reusing the queryId.

### **121 RequestNotFound** — Fulfill / Reclaim / Prune reverted
The `(consumer, queryId)` pair has no pending request. Either it never landed, or it already settled (fulfilled / reclaimed / pruned).
- `await fortuna.getRequest(consumer, queryId)` → `null` confirms no pending state.
- Check the event log: was there an `EvtRequestCreated` event? An `EvtRequestFulfilled` after?

### **122 DeadlineNotReached** — `ReclaimRequest` reverted
TTL hasn't elapsed. The default TTL is 3600 s (1 h).
- `await fortuna.getRequest(consumer, queryId)` returns `{ deadline, ... }` — wait until `now >= deadline`.
- Owner can call `PruneExpiredRequest` for admin-triggered edge cases.

### **123 StaleEpoch** — `FulfillRandomness` reverted
The request was placed before an Atlas group-key rotation and the operator tried to fulfill against the old epoch (D-008).
- Consumer should call `ReclaimRequest` after deadline for a refund.
- Operators: discard pre-rotation requests; current `groupEpoch` is `await fortuna.getGroupKey().groupEpoch`.

### **124 GroupKeyNotSet** — `RequestRandomness` reverted on a fresh deploy
Atlas hasn't fan-out the bootstrap `GroupKeySync` yet.
- Owner: `await fortuna.sendSyncAtlas(ownerSender, { value: toNano('0.1') })` to pull manually.
- Or wait for Atlas's automatic fan-out (happens on `SetVerifier` admission).

### **161 InvalidBlsSignature** — `FulfillRandomness` reverted
The aggregate sig doesn't verify against cached `groupPk` + derived alpha. Causes in order of likelihood:
1. **Wrong DST off-chain.** `@noble/curves` defaults to `_NUL_`; Fortuna requires `BLS_DST_G2_POP`. Import it from the SDK.
2. **Alpha pre-image drift.** Use `computeAlpha(consumer, queryId, seed, creationLt)` from `@titon-network/fortuna-sdk` rather than reimplementing.
3. **Stale pkShare.** Operator's share is from a prior rotation epoch; re-register after rotation.
4. **Threshold subset forgery.** Submitted a sig from fewer than `threshold` signers.

### **240 InsufficientValue** — almost any receiver reverted on entry
Attached `value` below the receiver's gas floor. For `RequestRandomness`, the floor is:

`baseRequestFee + submitterReward + callbackGas + minForwardReserve`

At `FORTUNA_DEFAULTS` with `callbackGas = 0.05`: **0.22 TON** (forward 0.25 for slack).

Compute precisely: `npx fortuna estimate request --callback-gas 0.05`.

### **241 ReserveInvariant** — outbound value send reverted
`balance < minStorageReserve + pendingFeeLocked + feeAccumulated`. Rare; usually means prior accounting drift. Owner: top up Fortuna with a plain transfer to clear.

### **200 NotForgeton / 201 NotAtlas** — inbound auth failed
Someone (not the pinned forgeton/atlas) tried to send `AutomatonSync` (0x1A) or `GroupKeySync` (0x51). Confirm:
- `await fortuna.getOwnership()` returns the addresses Fortuna trusts.
- `deployments.testnet.json` matches.

## TVM-level errors (not Fortuna-specific)

- **9 CellUnderflow** — wire-struct field widths off. Re-copy `RequestRandomness` / `VrfCallback` from `@titon-network/fortuna-sdk`'s `templates/consumer.tolk`.
- **13 OutOfGas** — bump `value` on the send.
- **37 NotEnoughTon** — contract balance too low for the outbound. Check reserve invariant; top up.
- **0xFFFF UnknownOpcode** — hit Fortuna's dispatcher `else` branch. Check op constants from `OP` in `@titon-network/fortuna-sdk`.

## Wrap and re-throw

```ts
import { explainError, FortunaError } from '@titon-network/fortuna-sdk';

try {
    await fortuna.sendFulfillRandomness(via, opts);
} catch (raw) {
    const code = extractExitCode(raw);   // your helper
    throw new FortunaError(code, `FulfillRandomness failed for queryId=${queryId}`);
}
```

`FortunaError` has `.code`, `.origin`, `.name`, `.message`, `.hint`.

## Useful CLI shortcuts

```bash
npx fortuna explain 124                    # what is GroupKeyNotSet
npx fortuna estimate request --callback-gas 0.05   # exact request value floor
npx fortuna info <addr> --testnet          # current contract state
npx fortuna verify --testnet               # SDK vs live-deploy drift check
```

## Related skills

- `/titon:fortuna-integrate-consumer` — if the failure was on a consumer-side `RequestRandomness`.
- `/titon:fortuna-monitor-events` — when the event log will help trace the root cause.
