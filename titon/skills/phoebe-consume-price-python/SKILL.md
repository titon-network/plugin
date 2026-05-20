---
name: phoebe-consume-price-python
description: Python equivalent of phoebe-consume-price — read a verified TON-mainnet price from Phoebe in a Python dapp / backend / script via `fetch_verified_price`, returning a `(leaf, proof)` pair ready to submit to a consumer contract. Use when the user says "from Python", "in pytoniq", "snake_case", "I'm using Python", "Python service that reads TON/USD", "phoebe-sdk in Python", "backend in Python needs a Phoebe price", or otherwise wants the Python flavour of the TS `fetchVerifiedPrice` helper. For the Tolk consumer contract side use `phoebe-integrate-consumer-python`. For the TS / frontend / @titon-network/phoebe-sdk side use `phoebe-consume-price`.
---

# Consume a Phoebe price from a dapp (Python side)

You're helping a dapp author read a verified price from Phoebe — the
TON-native threshold-BLS price oracle — using
`titon-network-phoebe-sdk` (PyPI). This skill is the **Python / backend
/ off-chain** side. For the on-chain `RequestPrice` Tolk consumer
contract that receives the verified `(mantissa, expo)` callback, load
`phoebe-integrate-consumer-python` instead. For the TypeScript /
frontend side, load `phoebe-consume-price`.

The two compose: this skill gets the dapp a `(leaf, proof)` pair; the
other skill writes the Tolk contract that submits them.

## Install

```bash
pip install titon-network-phoebe-sdk pytoniq
# or with poetry / uv:
poetry add titon-network-phoebe-sdk pytoniq
```

The PyPI distribution is the dash-form `titon-network-phoebe-sdk`; the
import path is the underscore-form `phoebe_sdk`. The SDK depends on
`pytoniq` for liteclient access, `aiohttp` for operator HTTP fetches,
and `py_ecc` for BLS12-381 primitives.

## Why this skill exists

Phoebe stores **only the merkle root on-chain** — never individual
prices. To USE a price, a consumer contract has to submit a `(leaf,
proof)` pair that hashes to the root. The leaves live in the operator
network's memory + are served over a public HTTP endpoint:

```
GET <op-url>/phoebe/v1/snapshot
→ { timestamp, rootHex, operatorAddress, leaves: [...] }
```

`fetch_verified_price` (added in `titon-network-phoebe-sdk@0.5.0`) is
the one-call API that does this safely: parallel-fetch all operators,
reconstruct the merkle root locally from the leaves, compare to
`phoebe.lastRoot` on-chain, and return only when one operator's
response verifies. Lying / stale / unreachable operators get skipped
automatically.

## The three-line read

```python
import asyncio
from pytoniq import LiteBalancer
from phoebe_sdk import Phoebe, assert_deployment, fetch_verified_price


async def main():
    client = LiteBalancer.from_mainnet_config(trust_level=2)
    await client.start_up()
    try:
        dep = assert_deployment("mainnet")                    # PHOEBE_MAINNET
        phoebe = Phoebe.create_from_address(dep.phoebe, client=client)

        quote = await fetch_verified_price(phoebe, feed_id=0, operators=dep.operators)
        # quote.mantissa, quote.expo  → price = mantissa × 10^expo
        # quote.proof, quote.leaf     → pass to your consumer contract
    finally:
        await client.close_all()


asyncio.run(main())
```

`dep.operators` ships in the SDK and is kept in lockstep with the
live operator set (currently EU + US). Dapps can also supply their own
list.

## Reading the price as a number (UI / API response)

```python
price = quote.mantissa * (10 ** quote.expo)
print(f"TON/USD = ${price:.4f}")
# → "TON/USD = $1.9550"
```

`mantissa` is a Python `int` (no precision loss for large markets;
BTC at $100k × 10⁸ fits comfortably). Cast to `float` only when
you've narrowed the range for display.

For exact decimal arithmetic on the price (interest accrual, settle
math) prefer `decimal.Decimal`:

```python
from decimal import Decimal

price = Decimal(quote.mantissa) * (Decimal(10) ** quote.expo)
```

## Submitting `(leaf, proof)` to your consumer contract

The consumer contract is your dapp's on-chain code. It sends
`RequestPrice` (0x71) to Phoebe with the leaf + proof; Phoebe verifies
the proof against `lastRoot` and calls back the consumer at
`FulfillPrice` (0x72) with the verified price.

```python
from phoebe_sdk import QueryIdStream, estimate_pull_value

cfg = await phoebe.get_config()
stream = QueryIdStream()

await phoebe.send_request_price(
    consumer_wallet,
    value=estimate_pull_value(pull_fee=cfg.pull_fee),
    query_id=stream.next(),
    feed_id=quote.feed_id,
    proof=quote.proof,
    leaf=quote.leaf,
    max_staleness=0,                              # 0 = accept any active root
)
```

For the **Tolk side** of this — writing the consumer that handles
`FulfillPrice` — load **`phoebe-integrate-consumer-python`**.

## Staleness handling

`fetch_verified_price` returns whatever the operators have, which may
be a few seconds old. Three options:

```python
# 1. Inspect age_sec in app code:
if quote.age_sec > 60:
    raise RuntimeError("phoebe price too stale")

# 2. Enforce client-side via max_age_sec:
from phoebe_sdk import FetchVerifiedPriceOptions

quote = await fetch_verified_price(
    phoebe,
    feed_id=0,
    operators=dep.operators,
    opts=FetchVerifiedPriceOptions(max_age_sec=30),
)

# 3. Enforce on-chain — your consumer contract checks `leaf.pubTime`
#    against `now()` before acting. RECOMMENDED for production
#    (trustless freshness).
```

## Trust model — why this is "elite"-tier

`fetch_verified_price` reconstructs the merkle root locally from the
fetched leaves and compares to `phoebe.lastRoot` on-chain. A lying
operator can't forge a matching root because they don't hold the
threshold BLS secret that signed the on-chain state. A *stale*
operator (whose snapshot rolled off-chain) gets caught by the same
hash mismatch and the helper falls through to the next operator
automatically.

The operator endpoints in `PHOEBE_MAINNET.operators` ship hardcoded in
the SDK for v0.5; future versions may discover them on-chain via a
phoebe operator registry. Either way, the helper's trust assumption
is the same: only the on-chain root matters.

## Error model

`fetch_verified_price` raises on:

| Reason | Recovery |
|---|---|
| Empty `operators` list (`ValueError`) | Pass `PHOEBE_MAINNET.operators` or your own list. |
| `phoebe.last_root == 0` (no snapshot ever) | Wait ~30s after group key publish for first push. |
| Every operator unreachable / 5xx | Transient — retry. |
| Every operator served a snapshot whose root ≠ on-chain | Operators are mid-roll (between push windows) OR a malicious set served the same lie — retry in ~30s. |
| Snapshot has no leaf for `feed_id` | That feed isn't published. Confirm the feed_id is in the canonical registry. |
| `age_sec > max_age_sec` | Snapshot older than your bound — retry or relax `max_age_sec`. |

Retry with exponential backoff (10-30s between push windows) is the
right pattern for transient failures. asyncio-friendly via
`tenacity`:

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(5), wait=wait_exponential(min=10, max=60))
async def pull_with_retry():
    return await fetch_verified_price(phoebe, feed_id=0, operators=dep.operators)
```

## Multi-feed snapshots — reuse the same operator

A snapshot commits to many feeds in one merkle root. If you need
several feeds in the same atomic read, pin to the operator that
served the first response — its snapshot is what `lastRoot` matches:

```python
ton_usd = await fetch_verified_price(phoebe, feed_id=0, operators=dep.operators)
matching_op = next(o for o in dep.operators if o.address == ton_usd.source_operator)
btc_usd = await fetch_verified_price(phoebe, feed_id=1, operators=[matching_op])
```

This avoids re-verifying against a different snapshot if a push
window rolls between the two calls.

## Pyth-style mode B (update + read)

For sub-heartbeat freshness, Phoebe supports mode B: the consumer
attaches a fresh signed snapshot inline with its `RequestPrice` and
Phoebe verifies + advances the cache + walks the proof against the
fresh root, all in one tx. Costs ~73k gas vs ~16k for mode A.

**This skill covers mode A only.** Mode B requires aggregating BLS
partials off-chain before submission. For the Python recipe see
`phoebe-pull-fresh-price-python`. The Tolk consumer side is identical
regardless of language; see `phoebe-integrate-consumer-python`.

## Production checklist

- [ ] Pin `titon-network-phoebe-sdk` to a specific minor (e.g. `~=0.5.0`)
- [ ] Use `PHOEBE_MAINNET.operators` — the SDK bumps this list as new operators come online
- [ ] Don't trust `quote.age_sec` for security; enforce `leaf.pub_time` staleness ON-CHAIN in your consumer contract
- [ ] Catch `RuntimeError` / `ValueError` from `fetch_verified_price` and retry with backoff (10-30s)
- [ ] If your TPS requires it, cache the verified quote for the current push window in your backend — don't re-fetch on every read
- [ ] Reuse a single `LiteBalancer` across calls; calling `await client.start_up()` per request burns connection setup time

## When NOT to use this skill

Load a different skill if:

- The user is writing the Tolk **consumer contract** that receives
  `FulfillPrice` → `phoebe-integrate-consumer-python`
- The user is reading prices from a **TS/frontend** layer →
  `phoebe-consume-price`
- The user is running a Phoebe **operator** (signing snapshots,
  managing the daemon) → `phoebe-operator-setup` (in the SDK skills)
- The user is **deploying** a fresh Phoebe instance →
  `phoebe-deploy-and-admit-python` (or `phoebe-deploy-and-admit` for TS)
- The user is **debugging a revert** they hit →
  `phoebe-debug-revert-python`

## Related types in the SDK

| Type | Purpose |
|---|---|
| `OperatorEndpoint` | `{address, url}` — one operator's public read endpoint (frozen dataclass) |
| `VerifiedPriceQuote` | `{feed_id, mantissa, expo, conf_bps, pub_time, snapshot_time, root, proof, leaf, source_operator, age_sec}` — the result |
| `FetchVerifiedPriceOptions` | `{max_age_sec, timeout_ms, now_sec, fetch_impl}` — tunables + test injection points |

All exported from `phoebe_sdk` root.

## Related skills

- `/titon:phoebe-integrate-consumer-python` — write the Tolk consumer + Python driver.
- `/titon:phoebe-pull-fresh-price-python` — mode B (Pyth-style update + read) from Python.
- `/titon:phoebe-debug-revert-python` — diagnose pull failures by exit code.
- `/titon:phoebe-consume-price` — TS sibling, if the dapp's read layer is JavaScript instead.
