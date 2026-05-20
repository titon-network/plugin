---
name: themis-debug
description: Debug a Themis revert — map TVM exit codes to root causes, common gotchas, and one-line fixes. Use when the user says "my Themis call reverted", "exit code 161 / 170 / etc.", "BLS_VERIFY failed", "submit ciphertext rejected", "operator not active", "round didn't advance", "AEAD decrypt failed", or "what does error N mean".
---

# Debug Themis

Themis reverts are deliberately specific — every assertion in the chamber + factory throws a numeric code that maps to one fix-it action. This skill is the reverse-lookup: starting from the symptom (revert, missing event, silent settlement, MAC failure), find the cause + the line that needs to change.

## The 60-second triage

1. **Get the exit code.** Sandbox: `await result.toHaveTransaction({ exitCode: 161 })`; production: `summarizeTx(tx)` (if available) or read the tx in an explorer. Each step in a multi-tx flow has its own exit code.
2. **Run `explainError(code)`** — returns a one-line prose hint.
3. **Locate the receiver.** Most reverts are receiver-specific: `SubmitCiphertext`, `RevealRound`, `AdvanceRound`, `DeployChamber`, the owner admin ops. Identify which one bounced.
4. **Cross-reference the table below.** Symptoms grouped by integration scenario — find yours and follow the fix.

## Symptom → cause → fix

### "My bid was rejected at SubmitCiphertext"

| Exit | Likely cause | Fix |
|------|--------------|-----|
| 101 `E_PAUSED` | Chamber owner paused new submissions | Wait for owner to `Unpause`, or check `chamber.getPaused()`. Owner-admin sends still work under pause; only `SubmitCiphertext` + `RevealRound` are blocked. |
| 152 `E_COMMIT_WINDOW_CLOSED` | `now >= commitEta` | Wait for next round. Check `chamber.getCurrentRound()` for the new `commitEta`. Don't queue submits during reveal phase. |
| 157 `E_INVALID_CIPHERTEXT` | `c1` failed `BLS_G1_INGROUP` subgroup check, OR malformed AEAD payload | You constructed `c1` outside `encryptBid` (or used `r=0`). Use `encryptBid` from the SDK — never hand-build the ciphertext. |
| 158 `E_BIDS_PER_ROUND_EXCEEDED` | Round is full (`bidCount == maxBidsPerRound`) | Wait for next round, OR petition chamber owner to `UpdateConfig` with higher `maxBidsPerRound`. |
| 164 `E_GROUP_KEY_NOT_SET` | Chamber never received its bootstrap `GroupKeySync` from Atlas | Atlas hasn't admitted the factory as a verifier yet, OR the bootstrap fan-out hasn't propagated. Owner can call `factory.sendSyncAtlas(...)` to re-trigger. |
| 240 `E_INSUFFICIENT_VALUE` | Attached `value < cfg.submitFee + gas floor` | Read `cfg.submitFee` live, attach `submitFee + toNano('0.05')`. Don't hardcode. |
| 333 `E_WRONG_WORKCHAIN` | Sender address is on workchain ≠ 0 | Themis is wc=0-only. Use a wc=0 wallet. |

### "RevealRound was rejected"

| Exit | Likely cause | Fix |
|------|--------------|-----|
| 153 `E_COMMIT_WINDOW_OPEN` | `now < commitEta`, can't reveal yet | Wait for `commitEta`. Off-chain operator's clock may be ahead of TVM's. |
| 154 `E_REVEAL_DEADLINE_PASSED` | `now > revealEta`, operator quorum was too slow | Anyone calls `AdvanceRound` → refunds bidders → round resets. Operators: tighten your reveal pipeline; verify quorum coordination. |
| 156 `E_ROUND_ALREADY_REVEALED` | Another operator's reveal landed first (this is the race the protocol intends), OR `signed.roundId != chamber.currentRound` | If `signed.roundId == chamber.currentRound`: the race was lost. Operator wasted gas (no on-chain cryptographic slashing). If signed roundId differs: you signed a stale payload — re-sign against the current round. |
| 159 `E_WRONG_CHAMBER_BINDING` | `signed.chamberBinding != chamberBindingHash(this.address)` | Off-chain signer signed for a different chamber. Use `chamberBindingHashTs(chamber.address)` — `chamber.address` MUST be the destination chamber, not the factory. |
| 160 `E_INVALID_BLS_SIGNATURE` | `BLS_VERIFY(groupPk, signedDataRef, sig)` failed | (1) Wrong DST — must be `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_`. Use `signAlpha` / `thresholdSign` from the SDK; the default `_NUL_` silently fails. (2) Stale group epoch — re-fetch `groupEpoch` and re-sign. (3) Signed bytes drift — recompute via `buildSignedRevealBytes()` and verify the 76-byte layout (chamberBinding32 ‖ roundIdBE8 ‖ groupEpochBE4 ‖ decryptionsRoot32). |
| 162 `E_INVALID_G1_POINT` | A `D_j` (decryption share) in the entries map failed `BLS_G1_INGROUP` | One of the operator's decryption shares is malformed. Operators: re-run `groupDecrypt(c1, sk)` for that share. |
| 166 `E_GROUP_EPOCH_STALE_SIGNED` | `signed.groupEpoch < chamber.groupEpoch` | Atlas rotated between when the operator signed and when the reveal landed. Re-fetch `groupEpoch` via `chamber.getGroupKey()` and re-sign. |
| 170 `E_OPERATOR_NOT_FOUND` | Sender is not in the mirrored operator map | Operator hasn't staked at ForgeTON yet, OR factory's `AutomatonSync` hasn't fanned out to the chamber yet. Stake → `EvtFanoutCycleComplete` → retry. |
| 171 `E_OPERATOR_NOT_ACTIVE` | Operator mirrored but `isActive: false` | Operator was deactivated at ForgeTON (unstake-to-zero, slash, or explicit deactivate). Re-stake → `AutomatonSync(isActive: true)` propagates. |
| 173 `E_CONSUMER_NOT_SET` | Chamber's `consumer` is addr_none | Chamber was deployed without a consumer. Owner can re-deploy a new chamber with `consumer` set; the current one is dead-letter. |
| 240 `E_INSUFFICIENT_VALUE` | Attached value < callback gas + cross-contract gas | Use the wrapper's `sendRevealRound` value floor (computes from `cfg.callbackGas + cfg.minXcGas + slack`). |

### "AdvanceRound didn't refund"

| Exit | Likely cause | Fix |
|------|--------------|-----|
| 155 `E_REVEAL_DEADLINE_NOT_PASSED` | `now <= revealEta`, can't advance yet | Wait. Branch A (refund) only runs after `revealEta`. |
| 156 `E_ROUND_ALREADY_REVEALED` | Round was already revealed | Already settled — no advance needed. Check `chamber.getCurrentRound()`; you're advancing the wrong round. |
| 179 `E_REWARD_POOL_UNDERFLOW` | Pool can't cover all `bidCount × submitFee` refunds | Pool was drained by prior `WithdrawFees` or `revealerReward` overpayments. Branch A caps refund at `min(totalRefund, rewardPool)`; check `getConfig().rewardPool` and inspect the per-bid refund logic. |

### "My consumer never got a callback"

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Chamber tx success but no inbound at consumer | `consumer` address mismatch | Deploy script wired wrong address. Re-check the `consumer:` field in the `DeployChamberArgs` passed to `factory.sendDeployChamber(via, value, args)` against your consumer's actual address. |
| Inbound arrives but reverts at consumer | Wrong opcode in your `match` arm | Verify `RevealCallback` struct has `(0x00000095)` prefix and is the first arm of `AllowedMessage`. |
| Inbound arrives, opcode parses, then reverts | Your sender pin doesn't match | `assert (sender == storage.boundChamber)`. The `sender` is the chamber's address — make sure you bound the right one. |
| Inbound arrives but no settlement happens | `bidCount == 0` (no bids in this round) | The chamber still calls back on empty rounds. Your handler should no-op for zero-bid rounds. SealedAMM does this with `if (bidCount == 0) { storage.lastFilledRound = …; return; }`. |

### "My settler decrypted but got gibberish / MAC failed"

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `decryptBid` throws "AEAD authentication failed" | Wrong `chamber.address` passed to decrypt | The AEAD key is bound to `(chamber.hash, roundIdBE)`. Pass the SAME chamber address that the bidder passed to `encryptBid`. The `RevealCallback.chamber` field is authoritative — use it. |
| Same as above, but consumer is bound to the right chamber | Wrong `roundId` passed | Same — must match the `roundId` from `RevealCallback`. Pull it from `decodeRevealCallback(body).roundId`. |
| MAC fails for some bids but not others | Dishonest `D` from operators on those bids (integrity-by-AEAD working as designed) | Skip those bids; chamber state unaffected; settlement just doesn't fire for them. **This is the trust model.** No on-chain cryptographic slashing — operators self-deactivate by being detected as dishonest off-chain. `ChallengeReveal` is RESERVED for a future revision. |
| MAC fails for ALL bids in a round | Atlas rotation happened between encrypt and decrypt | The `groupEpoch` at encrypt time differs from the `groupEpoch` at reveal time. Operators signed under the new epoch; bidders encrypted under the old. **Pattern is inherent** — bidders should re-encrypt if their bid hasn't yet been submitted across an epoch boundary. |

### "DeployChamber failed"

| Exit | Likely cause | Fix |
|------|--------------|-----|
| 101 `E_PAUSED` | Factory is paused | Owner unpaused it, OR retry on a different network. |
| 176 `E_INVALID_CONFIG` | Tunables would brick the chamber: zero `commitDuration` / `revealDuration`, `maxBidsPerRound == 0`, etc. | Read the anti-brick rules in [`themis-deploy-chamber.md`](./themis-deploy-chamber.md). Common: setting `commitDuration: 0` for testing — use `1` instead. |
| 240 `E_INSUFFICIENT_VALUE` | Attached < `deployChamberFee + minChamberInit + gas` | Attach `toNano('3')` — refunds excess. |

### "Atlas / ForgeTON sync didn't reach my chamber"

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Factory has cached groupKey but chamber doesn't | Fan-out cursor hasn't reached your chamber yet | Wait for `EvtFanoutCycleComplete (0xB7)` events. With many chambers, fan-out propagates over multiple txs. Owner can `factory.sendSyncAtlas(...)` to re-trigger. |
| Chamber's `getGroupKey()` reverts (or returns zeroed entry) on a fresh deploy | Factory wasn't admitted at Atlas yet, OR fan-out hasn't propagated | Atlas owner needs to call `atlas.sendSetVerifier(via, { value, contract: factory.address, isActive: true })`. That triggers the bootstrap GroupKeySync at the factory, which fans out to chambers. **NB:** the chamber's `getGroupKey()` is non-nullable — pre-bootstrap, getters revert on the load. Only the factory's `getGroupKey(groupId)` is nullable. |
| Operators mirrored at factory but not at chamber | Same — fan-out hadn't completed before chamber was deployed | Subsequent `AutomatonSync` from ForgeTON will reach the new chamber. For an immediate fix, ForgeTON owner can `forgeton.sendForceSync(via, { value, automaton })` (positional opts; re-fans-out that automaton's state to ALL consumers including the factory, which then fans to chambers). |
| Operators mirrored at factory but not at fresh chamber, AND no recent operator-state churn | ForgeTON's `AutomatonSync (0x1A)` is emitted ONLY on `Register (0x10)` / `Unstake-finalize (0x13)` / `Slash (0x14)` — NOT on `IncreaseStake` / `RequestUnstake` / `CancelUnstake` / opt-in / opt-out. So a chamber deployed AFTER all your operators registered will stay unmirrored indefinitely on a quiet network. | The ONLY operator-permissioned trigger is to fully unregister + re-register (24h cooldown — bad). The practical path is `forgeton.sendForceSync` from the ForgeTON owner, once per operator. Coordinate with the team. |

### "Wrapper getter returned wrong answer"

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `SealedAMM.getIsBound()` returns `true` even on a freshly-deployed (unbound) consumer | Wrapper bug in `wrappers/SealedAMM.ts` — the getter `try`s `getBoundChamber` and returns `true` on no-throw, but `getBoundChamber` returns the addr_none placeholder (not a revert) when unbound. | Don't trust `getIsBound()` for boundedness checks. Read the address directly via `getBoundChamber()` and compare to `new Address(0, Buffer.alloc(32, 0))`. |

## Common pitfalls (read these once)

### BLS DST drift

The single most common BLS bug:

```typescript
// WRONG — default DST is _NUL_, silently fails TVM's BLS_VERIFY.
import { bls12_381 as bls } from '@noble/curves/bls12-381';
const sig = bls.longSignatures.sign(msg, sk);

// RIGHT — use the SDK's signAlpha / thresholdSign, which pin BLS_DST_G2_POP.
import { signAlpha } from '@titon-network/themis-sdk';
const sig = signAlpha(sk, signedBytes);
```

If `BLS_VERIFY` fails with code 160 and you swear your sig is "correct", check the DST first.

### Chamber-binding crossing

```typescript
// WRONG — signed against a stand-in chamber, sent to a different one.
const built = buildReveal({ chamber: otherChamber, ... });
await actualChamber.sendRevealRound(via, built);

// RIGHT — always sign against the destination chamber's address.
const built = buildReveal({ chamber: actualChamber.address, ... });
await actualChamber.sendRevealRound(via, built);
```

The chamber computes `chamberBindingHash(this.address)` and compares to `signed.chamberBinding`. Mismatch → `E_WRONG_CHAMBER_BINDING (159)`.

### Stale `groupPk` in bidder UX

Read fresh, every bid. See [`themis-bidder-flow.md`](./themis-bidder-flow.md) "(1) Always read groupPk fresh".

### Encoder-side `c1` issues

`encryptBid` returns a 48-byte `Buffer` for `c1`. Don't:
- Slice/copy a smaller buffer (will revert at `E_INVALID_CIPHERTEXT`).
- Re-use the same `ephemeral:` across submits (ElGamal one-time-pad collapses).
- Hand-construct `c1` outside the SDK.

### Multi-op signing setup

For multi-op tests, `simulateDkg(n)` produces a `SimDkg` with all secret shares + the aggregate `groupPk`. The factory's cached `groupPk` MUST match `dkg.groupPk` — set it via `sendGroupKeySync(mockAtlas, factory.address, { groupPk: dkg.groupPk, ... })` in test setup.

If you forget this: the operator's threshold-sig verifies under `dkg.groupPk` but the chamber verifies under the cached `groupPk` (which doesn't match) → `E_INVALID_BLS_SIGNATURE (160)`.

### Workchain-0 only

Themis is wc=0-only across the stack. Any address received in a message (sender, automaton, recipient) is workchain-checked. Address from wc=-1 (basechain → masterchain forwarding) → `E_WRONG_WORKCHAIN (333)`.

If you're integrating from a wc=-1 wallet, deploy a wc=0 wallet first. Most TON SDKs default to wc=0.

## Inspecting state from the SDK

```typescript
// Chamber state.
await chamber.getOwnership();    // { chamberOwner, consumer }
await chamber.getPaused();       // boolean
await chamber.getCurrentRound(); // { roundId, commitEta, revealEta } — 3 fields only
await chamber.getGroupKey();     // { entryVersion, groupPk, groupEpoch, threshold, memberCount, cachedAt }
                                 //   non-nullable; reverts if never bootstrapped.
await chamber.getConfig();       // 11 fields incl. tunables + rewardPool (see ThemisChamber.ts)
await chamber.getOperator(addr); // { isActive } | null    (null = not mirrored)
await chamber.getSchemaVersions(); // { storage, config }   — 2 fields only
// Factory state.
await factory.getOwnership();    // { factoryOwner, forgeton, atlasAddress }
await factory.getPaused();
await factory.getNumChambers();
await factory.getChildCodeHash();
await factory.getSchemaVersions(); // { storage, config }
await factory.getRewardPool();   // bigint — reads through cfg blob
await factory.getGroupKey(0);    // { entryVersion, groupPk, …, groupId } | null
                                 //   null if no groupKey cached yet.
```

If a getter returns unexpected state (e.g. `factory.getGroupKey(0)` is null after admission, or `chamber.getGroupKey()` reverts), the most likely cause is fan-out not having propagated. Wait for the next `EvtFanoutCycleComplete`, or trigger via `factory.sendSyncAtlas(via, value)` from the owner.

## When a code is missing from this table

Run `explainError(code)` from the SDK for any code — it's the canonical mapping. If `explainError` doesn't have it, it's either a TVM standard code (9 = CellUnderflow, 13 = OutOfGas, 37 = NotEnoughTon) or a code added in a version newer than this skill file. Check [`ERRORS.md`](../ERRORS.md) for the full latest table.

## Cross-references

- [`AGENTS.md`](../AGENTS.md) — protocol overview, opcode + error ranges, surface.
- [`ERRORS.md`](../ERRORS.md) — every `E_*` with prose hint.
- [`OPCODES.md`](../OPCODES.md) — wire-format reference.
- [`skills/themis-integrate-consumer.md`](./themis-integrate-consumer.md) — consumer-side patterns.
- [`skills/themis-bidder-flow.md`](./themis-bidder-flow.md) — bidder-side patterns.
- [`skills/themis-deploy-chamber.md`](./themis-deploy-chamber.md) — deploy-time failures.
- `wrappers/errors.ts` (parent repo) — source of truth for the error table.
