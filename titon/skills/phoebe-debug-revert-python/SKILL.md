---
name: phoebe-debug-revert-python
description: Python equivalent of phoebe-debug-revert — diagnose a failed Phoebe transaction by exit code from a Python script / service / pytest run. Use when the user says "from Python", "in pytoniq", "snake_case", "I'm using Python", "what does Phoebe error X mean in Python", "PhoebeError thrown by phoebe_sdk", "explain_error", or pastes a pytoniq trace and asks for the root cause. For the TS / @titon-network/phoebe-sdk side use `phoebe-debug-revert`.
---

# Diagnose a Phoebe exit code (Python)

You're helping a user understand why a Phoebe transaction reverted
from a Python tool / service / test. Phoebe's `explain_error` covers
Phoebe codes (100–249), cross-contract auth codes (200/201), TVM
commons, and the upgrade-timelock range.

## Get the code

The exit code lives at:
- pytoniq tx: `tx.description.compute_ph.exit_code` (and `tx.description.action.result_code` for the action phase)
- pytoniq `run_get_method` raises `LiteClientError` containing the code on getter reverts
- tonscan / tonviewer: shown on the failed tx page

If you have a pytoniq `Transaction`:

```python
from phoebe_sdk import summarize_tx, format_tx_summary

for tx in transactions:
    summary = summarize_tx(tx)
    print(format_tx_summary(summary))
    # → "[fail] exit 161 InvalidBlsSignature — BLS_VERIFY rejected the aggregate sig"
```

If you have a thrown SDK error:

```python
from phoebe_sdk import PhoebeError

try:
    ...
except PhoebeError as e:
    print(f"[{e.origin} exit {e.code}] {e}")
    if e.hint:
        print(f"hint: {e.hint}")
```

## Explain it

```python
from phoebe_sdk import explain_error, format_error_explanation

e = explain_error(161)
print(f"{e.origin}:{e.name} — {e.message}")
if e.hint:
    print(f"hint: {e.hint}")

# Or one-line:
print(format_error_explanation(e))
```

CLI shortcut from the TS SDK if available: `npx phoebe explain 161`.

Full table: see `ERRORS.md` in the SDK repo or `explain_error` source.

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
1. **Wrong DST off-chain.** Use `sign_message(sk, hash_bytes)` from `phoebe_sdk`; `py_ecc` raw defaults can drift from the `BLS_DST_G2_POP` Phoebe expects.
2. **Hash domain drift.** `compute_snapshot_hash(phoebe_address, timestamp, root)` must be called with the EXACT (address, ts, root) you're submitting.
3. **Stale pkShare.** Operator's share is from a prior Atlas rotation epoch; re-register after rotation.
4. **Wrong Phoebe address signed for.** See 144 below — operator endpoint may be signing for a different Phoebe.

For mode-B pulls (consumer-side): the operator endpoint served a bad sig. Re-fetch from a healthy operator; verify their endpoint isn't on a stale group epoch.

### **180 GroupKeyNotSet** — push or mode-B pull reverted on a fresh deploy
Atlas hasn't fanned out the bootstrap `GroupKeySync` to Phoebe. Owner-side:

```python
await phoebe.send_sync_atlas(owner_wallet, value=100_000_000)  # 0.1 TON
```

Then verify: `await phoebe.get_group_key()` should return non-None. Or wait for Atlas's automatic fan-out (triggered by `SetVerifier` admission at Atlas).

### **181 StaleSnapshot** — push OR pull reverted
**On push:** `timestamp <= storage.last_snapshot_time`. Another operator pushed first; use a strictly newer timestamp.
**On pull:** `max_staleness` exceeded against the active root. Set `max_staleness=0` to accept any active root, OR wait for a fresh push, OR attach a mode-B `fresh_update` to satisfy a tight bound.

### **183 BadMerkleProof** — pull reverted post-entry
Three causes:
1. **Stale proof.** Every push rotates `last_root` and invalidates prior proofs. Rebuild with `PhoebeMerkleTree.from_sparse_leaves(latest_leaves).proof(feed_id)` against the active root.
2. **Tampered leaf.** Pass the same `PriceLeaf` you used to build the tree — no mutation between build + send.
3. **feed_id mismatch.** `leaf.feed_id` MUST equal the `feed_id` argument (the request's `feedId` field). The SDK pre-validates this in `build_request_price_body`, so this fires mainly when the proof was built against a different leaf set.

For mode-B specifically: build the proof against `fresh_update`'s root, NOT the cached `last_root`.

### **187 FreshNotFresher** — mode-B pull reverted post-entry
`fresh_update.timestamp <= cached last_snapshot_time`. The fresh-update must be strictly newer. Common causes:
- Operator endpoint is lagging — re-fetch from a healthier operator.
- Another consumer's mode-B pull beat yours into the same block (strict-monotonic; second consumer at same ts reverts).
- An operator's heartbeat push beat you.

Always re-check `await phoebe.get_snapshot()` right before sending; fall back to mode A when fresh isn't fresher. See `/titon:phoebe-pull-fresh-price-python` for the defensive pattern.

### **144 WrongDeploymentBinding** — push OR mode-B pull reverted pre-BLS-verify
Signed payload's `phoebeHash` doesn't match this Phoebe's address-hash. Either:
- Operator endpoint is signing for a DIFFERENT Phoebe deployment (mis-config).
- Someone is replaying a sig from a sibling Phoebe sharing the same Atlas group.

Pass the **destination** `Address` to `compute_snapshot_hash`. For consumers: re-fetch from the right operator endpoint for THIS Phoebe.

### **240 InsufficientValue** — almost any receiver reverted on entry
Attached `value` below the receiver's floor:
- `RequestPrice` (mode A): `cfg.pull_fee + MIN_GAS_FOR_PULL`. Use `estimate_pull_value(pull_fee=...)`.
- `RequestPrice` (mode B): add `with_fresh=True` (+0.05 TON for BLS_VERIFY budget).
- `PushSnapshot`: `MIN_GAS_FOR_PUSH` (~0.05 TON). Use `estimate_push_value()`.
- `ClaimReward`: `MIN_GAS_FOR_CLAIM` (~0.03 TON). Use `estimate_claim_value()`.

Read live `pull_fee` via `await phoebe.get_config()` — don't hardcode.

### **182 BadTimestampDrift** — push or mode-B pull reverted
`|now - timestamp| > max_push_drift` (300s default). Causes:
- Operator clock skew — sync via NTP.
- Operator endpoint serves a stale fresh-update — re-fetch.
- `max_push_drift` is too tight for your network's jitter — owner can `send_update_config(max_push_drift=...)` to raise it.

### **200 NotForgeton / 201 NotAtlas** — cross-contract inbound rejected
Sender ≠ pinned `storage.forgeton` / `storage.atlas`. Phoebe pins both at deploy; repointing requires a code upgrade. Check `await phoebe.get_ownership()`; verify `deployments.testnet.json` matches.

### **188 NoUnclaimedReward** — `ClaimReward` rejected
Operator's accrued balance is zero (never accrued, or just drained). Query `await phoebe.get_unclaimed_reward(addr)` first; only claim when it's > 0. Note: accrual is **pull-time** (when consumers pull), not push-time — if you pushed but no consumer has pulled yet, you have no balance.

## TVM-level errors (not Phoebe-specific)

- **9 CellUnderflow** — wire-struct widths off. Re-copy `RequestPrice` / `FulfillPrice` / `PriceLeaf` from the SDK's `templates/consumer.tolk` (or the Tolk snippet in `phoebe-integrate-consumer-python`).
- **13 OutOfGas** — bump `value=` on the send. For the consumer's `FulfillPrice` handler, increase value on the originating `RequestPrice`.
- **37 NotEnoughTon** — contract balance below reserve invariant. Owner-side: plain transfer to top up.
- **0xFFFF UnknownOpcode** — hit Phoebe's dispatcher `else` branch. Use `OP[...]` from `phoebe_sdk` instead of hand-encoding.

## "Pull worked but FulfillPrice never landed at my consumer"

Symptom: `RequestPrice` succeeded; consumer's `handleFulfillPrice` never fired.

1. **Consumer reverts on receive.** Phoebe sends with `bounce: false` — revert at consumer does NOT bounce back. Phoebe-side state is already finalized (pullFee retained). Walk the consumer's tx with `summarize_tx` and check the exit code.
2. **Wrong opcode.** Consumer's receiver must be `struct (0x00000072) FulfillPrice`. Wrong opcode → 0xFFFF at the consumer.
3. **Wrong workchain.** Phoebe + consumer on different workchains → action 36 at Phoebe's outbound. Pin both to workchain 0.
4. **Insufficient consumer-side gas.** Phoebe forwards what's left after `pull_fee` is retained. Heavy callback + tight value → exit 13 (OutOfGas) at consumer. Increase `value` on the originating `RequestPrice`.

## Cross-contract bounces

If the revert came from a sibling protocol (Atlas / ForgeTON), force
the lookup origin:

```python
e = explain_error(155, force_origin="forgeton")    # ForgeTON range
e = explain_error(225, force_origin="atlas")       # Atlas range
```

Returns a forwarded hint pointing at the sibling SDK's explainer.

## Wrap and re-throw

```python
from phoebe_sdk import PhoebeError, explain_error

try:
    await phoebe.send_request_price(consumer_wallet, ...)
except Exception as raw:
    code = extract_exit_code(raw)            # your helper, e.g. parsing LiteClientError
    raise PhoebeError(code, f"RequestPrice failed for query_id={query_id}") from raw
```

`PhoebeError` has `.code`, `.origin`, `.hint`, and an informative
`str(...)` that already includes the name + message.

## Useful CLI / inspection

```bash
# TS SDK CLI shortcuts (still useful even if your driver is Python):
npx phoebe info <addr> --testnet          # live state: pullFee, group key, last snapshot
npx phoebe explain 161                    # what is InvalidBlsSignature

# Python — inspect via the SDK directly:
python -c "from phoebe_sdk import explain_error, format_error_explanation; \
           print(format_error_explanation(explain_error(161)))"
```

## Watch out for

- **Don't retry blindly.** A failed push or pull burns gas. Diagnose the exit code first.
- **Don't ignore `NotForgeton` / `NotAtlas`.** These mean Phoebe was deployed pointing at the WRONG peer addresses — a deploy-time bug. Don't try to "fix" by re-syncing; you need a timelocked code upgrade to repoint.
- **Don't mix up `timestamp` and `group_epoch`.** `timestamp` is unix seconds; `group_epoch` is Atlas's monotonic rotation counter. Different concepts.
- **Don't trust an unauthenticated `FulfillPrice` body.** Sender-pin in your consumer is non-negotiable.
- **Don't catch every exception as `PhoebeError`.** Network errors from pytoniq are different — let them propagate or wrap them separately so retry logic stays sane.

## Related skills

- `/titon:phoebe-integrate-consumer-python` — if the failure was on a consumer-side `RequestPrice`.
- `/titon:phoebe-pull-fresh-price-python` — if the failure was on a mode-B fresh-update.
- `/titon:phoebe-handle-event-python` — when the event log will help trace the root cause.
- `/titon:phoebe-deploy-and-admit-python` — if `NotForgeton`/`NotAtlas` points at a mis-bound deployment.
- `/titon:phoebe-debug-revert` — TS sibling; the error ranges + diagnostics are identical.
