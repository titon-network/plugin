---
name: kronos-debug-exit-code-python
description: Diagnose a failed Kronos transaction by exit code from Python. Use when a TON transaction reverted with a numeric exit code (e.g. 102, 132, 135, 9, 13), when the user pastes a transaction trace, or when they ask "what does Kronos error X mean?" from a Python context.
---

# Diagnose a Kronos exit code (Python)

You're helping the user understand why a transaction reverted, from a Python application using `titon-network-kronos-sdk`.

## What you need

The exit code (integer). It's surfaced as:
- `pytoniq` trace: `tx.description.compute_ph.exit_code`
- TonCenter API JSON: `compute_ph.exit_code`
- Tonviewer / tonscan UI: shown on the failed tx page.

If the user pasted a tx hash but no code, walk them through pulling the trace via `pytoniq.LiteBalancer.get_transactions(addr, count=N)` and reading `compute_ph.exit_code` off the matching tx.

## The recipe

```python
from kronos_sdk import explain_error, format_error_explanation

ex = explain_error(exit_code)
print(f"[{ex.name}] {ex.message}")
if ex.hint:
    print(f"hint: {ex.hint}")
print(f"origin: {ex.origin}")   # 'kronos' | 'forgeton' | 'tvm' | 'unknown'

# Or one-shot pretty-print:
print(format_error_explanation(ex))
```

`explain_error` covers:
- **Kronos codes 100+** — `errors.tolk` table (NotOwner, Paused, InsufficientGas, NotJobOwner, BelowMinForActive, TooEarly/TooLate, etc.)
- **TVM codes** — common ones like 13 (out of gas), 9 (cell underflow), 37 (action phase: not enough TON), 0xFFFF (unknown opcode).

The `origin` field can also surface `'unknown'` for codes that aren't in either table — typically a revert from your target contract (Kronos forwards your body verbatim, so the target's own asserts surface in the trace).

## Common code → fix recipes

| Code | Name | Likely cause | First thing to try |
|------|------|-------------|--------------------|
| 100 | NotOwner | Wallet ≠ contract owner | Use the registry/pool owner wallet. |
| 101 | Paused | Contract is paused | Wait for owner to Unpause, or use a pause-skip op (cancel, withdraw, cleanup). |
| 102 | InsufficientGas | `value` < `min_gas_reserve` | Send ≥ `min_gas_reserve` (default 0.05 TON). |
| 104 | WouldBreachReserve | Outgoing payment would empty the contract | Top up the contract or reduce withdrawal amount. |
| 120 | InsufficientFunding | RegisterJob value too low | Use `preview_job_cost(...)` and attach the recommended value. |
| 131 | InsufficientJobBalance | Job can't cover one execution | Call `client.jobs.fund(...)` to top up. |
| 132 | NotJobOwner | Update/withdraw/cancel from wrong wallet | Job owner is the wallet that registered. |
| 134 | BelowMinForActive | Partial withdraw would leave job unfundable | Reduce withdraw amount or fully drain via cancel. |
| 135 | TooEarly | Execute called before `next_due - window_before` | Wait or check `await client.jobs.window_for(job_id)`. |
| 136 | TooLate | Execute called past `next_due + window_after` | The cycle was missed; wait for the next `next_due`. |
| 137 | NotAssigned | Caller isn't the assigned automaton in primary window | Dapp authors don't usually see this — it surfaces operator-side. If you do see it, check `await client.assigned_automaton_for(job_id)` and wait for the fallback window. |
| 9 | CellUnderflow (TVM) | Malformed message body | Compare your encoding against the Tolk struct definition. |
| 13 | OutOfGas (TVM) | `gas_limit` too low or `value` too low | Increase the attached value. Use `/titon:kronos-gas-sizing-python`. |
| 37 | NotEnoughTon (TVM action) | Contract tried to send more than its balance | Top up the contract. |
| 0xFFFF | UnknownOpcode | Wrong opcode or wrong target contract | Check `OP['<name>']` matches what the contract expects. |

## When the code isn't in the table

`explain_error(99999)` returns an `ErrorExplanation` with `origin='unknown'`. Tell the user the code isn't a Kronos or common-TVM code. It's probably:
- A custom error from the **target** contract (Kronos forwards the body — the target's TVM revert surfaces in the action phase).
- A future code added in a contract upgrade not yet reflected in the SDK.

## Wrap and re-throw

`KronosError` is provided as a typed wrapper for cleaner error-handling pipelines. Its constructor takes the `ErrorExplanation` (so the formatted message is stable):

```python
from kronos_sdk import KronosError, explain_error

try:
    await client.jobs.register(wallet, **opts.as_kwargs())
except Exception as raw:
    code = extract_exit_code(raw)        # your helper that pulls it out
    raise KronosError(explain_error(code)) from raw
```

`KronosError` exposes `.explanation` (the full `ErrorExplanation` with `.code`, `.origin`, `.name`, `.message`, `.hint`); `str(err)` is the pre-formatted human-readable message.

## Reading the exit code from a pytoniq tx

```python
from pytoniq import LiteBalancer

rpc = LiteBalancer.from_testnet_config(trust_level=2)
await rpc.start_up()

txs = await rpc.get_transactions(some_address, count=10)
for tx in txs:
    cp = tx.description.compute_ph
    if hasattr(cp, "exit_code") and cp.exit_code != 0:
        print(f"failed tx: exit_code={cp.exit_code}")
        from kronos_sdk import explain_error
        print(format_error_explanation(explain_error(cp.exit_code)))
```

## CLI shortcuts

The TS SDK ships `npx kronos-sdk decode-error <code>`; the Python SDK doesn't (yet). You can do the same in one line:

```bash
python3 -c "from kronos_sdk import explain_error, format_error_explanation; print(format_error_explanation(explain_error(124)))"
```

## Related skills

- `/titon:kronos-register-job-python` — if the failure was during job registration.
- `/titon:kronos-job-lifecycle-python` — if the failure was during update / pause / withdraw / cancel.
- `/titon:kronos-gas-sizing-python` — if the cause was OOG (TVM 13).
