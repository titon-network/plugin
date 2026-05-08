---
name: fortuna-debug-exit-code-python
description: Diagnose a failed Fortuna transaction by exit code from Python. Use when a TON transaction reverted with a numeric exit code (e.g. 120, 124, 161, 240), when the user pastes a transaction trace, or when they ask "what does Fortuna error X mean?" from a Python context.
---

# Diagnose a Fortuna exit code (Python)

You're helping a user understand why a Fortuna transaction reverted, from a Python application using `titon-network-fortuna-sdk`. Fortuna's `explain_error` covers Fortuna codes (100-249), cross-contract auth codes (200/201), and common TVM codes.

## Get the code

The exit code is at:
- `pytoniq` trace: `tx.description.compute_ph.exit_code`
- TonCenter API JSON: `compute_ph.exit_code`
- TON viewer / tonscan: shown on the failed tx page

## The recipe

```python
from fortuna_sdk import explain_error, format_error_explanation

ex = explain_error(124)
print(f"{ex.origin}:{ex.name} — {ex.message}")
if ex.hint:
    print(f"hint: {ex.hint}")

# One-shot pretty-print:
print(format_error_explanation(ex))
```

Shell shortcut:

```bash
python3 -c "from fortuna_sdk import explain_error, format_error_explanation; print(format_error_explanation(explain_error(124)))"
```

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
You used the same `(consumer, query_id)` while a previous request was still pending.
- Use a monotone counter on-chain, OR `QueryIdStream` from the SDK off-chain.
- Wait for fulfillment / reclaim before reusing the queryId.

### **121 RequestNotFound** — Fulfill / Reclaim / Prune reverted
The `(consumer, query_id)` pair has no pending request. Either it never landed, or it already settled (fulfilled / reclaimed / pruned).
- `await fortuna.get_request(consumer, query_id)` → `None` confirms no pending state.
- Check the event log: was there a `RequestCreated` event? A `RequestFulfilled` after?

### **122 DeadlineNotReached** — `ReclaimRequest` reverted
TTL hasn't elapsed. The default TTL is 3600 s (1 h).
- `await fortuna.get_request(consumer, query_id)` returns a record with `deadline` — wait until `now >= deadline`.
- Owner can call `PruneExpiredRequest` for admin-triggered edge cases.

### **123 StaleEpoch** — `FulfillRandomness` reverted
The request was placed before an Atlas group-key rotation and the operator tried to fulfill against the old epoch (D-008).
- Consumer should call `ReclaimRequest` after deadline for a refund.
- Operators: discard pre-rotation requests; current `group_epoch` is `(await fortuna.get_group_key()).group_epoch`.

### **124 GroupKeyNotSet** — `RequestRandomness` reverted on a fresh deploy
Atlas hasn't fan-out the bootstrap `GroupKeySync` yet.
- Owner: `await fortuna.send_sync_atlas(owner_wallet, value=int(0.1 * TON))` to pull manually.
- Or wait for Atlas's automatic fan-out (happens on `SetVerifier` admission).

### **161 InvalidBlsSignature** — `FulfillRandomness` reverted
The aggregate sig doesn't verify against cached `groupPk` + derived alpha. Causes in order of likelihood:
1. **Wrong DST off-chain.** The Python SDK's `sign_alpha` already pins `BLS_DST_G2_POP` via `py_ecc.bls.G2ProofOfPossession`. Drift only happens if a custom signer is used.
2. **Alpha pre-image drift.** Use `compute_alpha(consumer, query_id, seed, creation_lt)` from `fortuna_sdk` rather than reimplementing.
3. **Stale pkShare.** Operator's share is from a prior rotation epoch; re-register after rotation.
4. **Threshold subset forgery.** Submitted a sig from fewer than `threshold` signers.

### **240 InsufficientValue** — almost any receiver reverted on entry
Attached `value` below the receiver's gas floor. For `RequestRandomness`, the floor is:

`base_request_fee + submitter_reward + callback_gas + min_forward_reserve`

At `FORTUNA_DEFAULTS` with `callback_gas = 0.05`: **0.22 TON** (forward 0.25 for slack).

Compute precisely from the live config:

```python
from fortuna_sdk import Fortuna, FORTUNA_TESTNET
from pytoniq import LiteBalancer

TON = 10**9

rpc = LiteBalancer.from_testnet_config(trust_level=2)
await rpc.start_up()

fortuna = Fortuna.create_from_address(FORTUNA_TESTNET.fortuna, client=rpc)
cfg = await fortuna.get_config()
callback_gas = int(0.05 * TON)
floor = (
    cfg.base_request_fee
    + cfg.submitter_reward
    + callback_gas
    + cfg.min_forward_reserve
)
print(f"min request value: {floor / TON} TON")
# Or call the MCP `fortuna.preview_request_value` tool, which does the same
# math server-side without requiring an RPC connection from your client.
```

### **241 ReserveInvariant** — outbound value send reverted
`balance < min_storage_reserve + pending_fee_locked + fee_accumulated`. Rare; usually means prior accounting drift. Owner: top up Fortuna with a plain transfer to clear.

### **200 NotForgeton / 201 NotAtlas** — inbound auth failed
Someone (not the pinned forgeton/atlas) tried to send `AutomatonSync` (0x1A) or `GroupKeySync` (0x51). Confirm:
- `await fortuna.get_ownership()` returns the addresses Fortuna trusts.
- `FORTUNA_TESTNET` from the SDK matches.

## TVM-level errors (not Fortuna-specific)

- **9 CellUnderflow** — wire-struct field widths off. Re-copy `RequestRandomness` / `VrfCallback` from the canonical Tolk struct definitions.
- **13 OutOfGas** — bump `value` on the send.
- **37 NotEnoughTon** — contract balance too low for the outbound. Check reserve invariant; top up.
- **0xFFFF UnknownOpcode** — hit Fortuna's dispatcher `else` branch. Check op constants from `OP` in `fortuna_sdk` (it's a dict — `OP['RequestRandomness']`).

## Wrap and re-throw

```python
from fortuna_sdk import explain_error, FortunaError

try:
    await fortuna.send_fulfill_randomness(operator_wallet, ...)
except Exception as raw:
    code = extract_exit_code(raw)   # your helper
    raise FortunaError(code, f"FulfillRandomness failed for query_id={query_id}") from raw
```

`FortunaError` has `.code`, `.origin`, `.name`, `.message`, `.hint`.

## Reading the exit code from a pytoniq tx

```python
from pytoniq import LiteBalancer
from fortuna_sdk import explain_error, format_error_explanation, FORTUNA_TESTNET

rpc = LiteBalancer.from_testnet_config(trust_level=2)
await rpc.start_up()

txs = await rpc.get_transactions(FORTUNA_TESTNET.fortuna, count=10)
for tx in txs:
    cp = tx.description.compute_ph
    if hasattr(cp, "exit_code") and cp.exit_code != 0:
        print(format_error_explanation(explain_error(cp.exit_code)))
```

## Related skills

- `/titon:fortuna-integrate-consumer-python` — if the failure was on a consumer-side `RequestRandomness`.
- `/titon:fortuna-monitor-events-python` — when the event log will help trace the root cause.
