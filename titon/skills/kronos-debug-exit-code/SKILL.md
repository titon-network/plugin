---
name: kronos-debug-exit-code
description: Diagnose a failed Kronos transaction by exit code. Use when a TON transaction reverted with a numeric exit code (e.g. 102, 132, 135, 9, 13), when the user pastes a transaction trace, or when they ask "what does Kronos error X mean?".
---

# Diagnose a Kronos exit code

You're helping the user understand why a transaction reverted.

## What you need

The exit code (integer). It's surfaced as:
- Sandbox: `tx.description.computePhase.exitCode`
- TonClient trace: `compute_ph.exit_code`
- Toncenter UI: shown on the failed tx page.

If the user pasted a tx hash but no code, ask them to fetch the trace OR walk them through using `tonClient.getTransaction` to extract it.

## The recipe

```ts
import { explainError } from '@titon-network/kronos-sdk';

const e = explainError(exitCode);
console.error(`[${e.name}] ${e.message}`);
if (e.hint) console.error(`hint: ${e.hint}`);
console.error(`origin: ${e.origin}`);  // 'kronos' | 'forgeton' | 'tvm' | 'unknown'
```

`explainError` covers:
- **Kronos codes 100+** — `errors.tolk` table (NotOwner, Paused, InsufficientGas, NotJobOwner, BelowMinForActive, TooEarly/TooLate, etc.)
- **TVM codes** — common ones like 13 (out of gas), 9 (cell underflow), 37 (action phase: not enough TON), 0xFFFF (unknown opcode).

The `origin` field can also surface `'unknown'` for codes that aren't in either table — typically a revert from your target contract (Kronos forwards your body verbatim, so the target's own asserts surface in the trace).

## Common code → fix recipes

| Code | Name | Likely cause | First thing to try |
|------|------|-------------|--------------------|
| 100 | NotOwner | Wallet ≠ contract owner | Use the registry/pool owner wallet. |
| 101 | Paused | Contract is paused | Wait for owner to Unpause, or use a pause-skip op (cancel, withdraw, cleanup). |
| 102 | InsufficientGas | `value` < `minGasReserve` | Send ≥ `minGasReserve` (default 0.05 TON). |
| 104 | WouldBreachReserve | Outgoing payment would empty the contract | Top up the contract or reduce withdrawal amount. |
| 120 | InsufficientFunding | RegisterJob value too low | Use `previewJobCost(...)` and attach the recommended value. |
| 131 | InsufficientJobBalance | Job can't cover one execution | Call `client.jobs.fund(...)` to top up. |
| 132 | NotJobOwner | Update/withdraw/cancel from wrong wallet | Job owner is the wallet that registered. |
| 134 | BelowMinForActive | Partial withdraw would leave job unfundable | Reduce withdraw amount or fully drain via cancel. |
| 135 | TooEarly | Execute called before nextDue - windowBefore | Wait or use `client.windowFor(jobId).secondsToNext`. |
| 136 | TooLate | Execute called past nextDue + windowAfter | The cycle was missed; wait for the next nextDue. |
| 137 | NotAssigned | Caller isn't the assigned automaton in primary window | Dapp authors don't usually see this — it surfaces operator-side. If you do see it, check `client.assignedAutomatonFor(jobId)` and wait for the fallback window. |
| 9 | CellUnderflow (TVM) | Malformed message body | Compare your encoding against `contracts/messages.tolk` field order. |
| 13 | OutOfGas (TVM) | `gasLimit` too low or `value` too low | Increase the attached value. Use `/titon:kronos-gas-sizing`. |
| 37 | NotEnoughTon (TVM action) | Contract tried to send more than its balance | Top up the contract. |
| 0xFFFF | UnknownOpcode | Wrong opcode or wrong target contract | Check `OP.X` matches what the contract expects. |

## When the code isn't in the table

`explainError(99999) → { origin: 'unknown' }`. Tell the user the code isn't a Kronos or common-TVM code. It's probably:
- A custom error from the **target** contract (Kronos forwards the body — the target's TVM revert surfaces in the action phase).
- A future code added in a contract upgrade not yet reflected in the SDK.

## Wrap and re-throw

If the user is writing their own error-handling layer, wrap `explainError` into their own Error subclass — @titon-network/kronos-sdk doesn't ship one because the explanation is already rich:

```ts
try {
    await client.jobs.register(via, opts);
} catch (raw) {
    const code = extractExitCode(raw);   // user's helper
    const e = explainError(code);
    throw new Error(`[${e.name}] ${e.message}${e.hint ? ` — ${e.hint}` : ''}`);
}
```

## Related skills

- `/titon:kronos-register-job` — if the failure was during job registration.
- `/titon:kronos-job-lifecycle` — if the failure was during update / pause / withdraw / cancel.
- `/titon:kronos-gas-sizing` — if the cause was OOG (TVM 13).
