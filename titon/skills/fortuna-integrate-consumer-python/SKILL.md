---
name: fortuna-integrate-consumer-python
description: Add verifiable randomness to a TON product via Fortuna VRF using Python tooling — write the 0x50 VrfCallback receiver, send 0x30 RequestRandomness from Python, size the value correctly. Use when the user is building anything that needs unbiasable on-chain randomness (raffle, NFT trait roll, dice, lottery, mystery box, shuffled queue) and is integrating from a Python application.
---

# Integrate Fortuna VRF into a consumer contract (Python)

You're helping a product dev add verifiable randomness with `titon-network-fortuna-sdk`. Fortuna is TON's threshold-BLS VRF — sovereign and on-TON. The consumer-side Tolk pattern is identical regardless of language; this skill covers the Tolk pattern + the Python-side trigger code.

## Install

```bash
pip install titon-network-fortuna-sdk pytoniq
```

The Fortuna Python SDK depends on `py_ecc` for BLS12-381 verification — pulled in automatically.

## Wire types — copy verbatim from the SDK

```tolk
// Inbound from Fortuna. beta is the canonical VRF output — uniform uint256.
struct (0x00000050) VrfCallback {
    queryId: uint64
    beta:    uint256
}

// Outbound to Fortuna. Attached value must cover the request fee surface.
struct (0x00000030) RequestRandomness {
    queryId:     uint64
    seed:        uint256
    callbackGas: coins
}
```

Drift breaks the lazy-decoder silently. Re-copy after every Fortuna upgrade.

## The 5 rules every consumer follows

1. **Pin Fortuna at deploy.** `storage.fortuna: address` set once; no runtime SetFortuna. Repointing requires a contract upgrade.
2. **Sender-gate the callback.** Top of `handleVrfCallback`:
   ```tolk
   assert (sender == storage.fortuna) throw E_NOT_FORTUNA;
   ```
   Without this, anyone can send opcode `0x50` with attacker-chosen `beta`.
3. **Unique queryId per pending request.** Reusing one before fulfillment reverts at Fortuna with `E_DUPLICATE_REQUEST = 120`. Use a monotone counter on-chain or `QueryIdStream` off-chain (see Python recipe below).
4. **Funded request value.** At `FORTUNA_DEFAULTS`, the floor is `base_request_fee + submitter_reward + callback_gas + min_forward_reserve` = **0.22 TON** for `callback_gas = 0.05`. Add ~0.03 TON forward-fee slack → forward 0.25 TON. Below the floor reverts with `E_INSUFFICIENT_VALUE = 240`.
5. **Revert-proof callback.** Fortuna sends `VrfCallback` with `bounce: false` — a revert in your handler does NOT bounce, but the request state IS already cleared, so failed callbacks lose the forwarded gas. Keep it cheap and assertion-light.

## Tolk skeleton (consumer side)

```tolk
const E_NOT_FORTUNA: int = 201;

struct ConsumerStorage {
    owner:       address
    fortuna:     address           // pinned at deploy
    nextQueryId: uint64
    // ... your business state
}

fun handleVrfCallback(msg: VrfCallback, sender: address) {
    var storage = lazy ConsumerStorage.load();
    assert (sender == storage.fortuna) throw E_NOT_FORTUNA;
    // beta is uniform uint256 — modulo, parity, range, …
    // Your randomness-consuming logic here.
    storage.save();
}

fun triggerWhatever(sender: address /* + your trigger args */) {
    var storage = lazy ConsumerStorage.load();
    val queryId = storage.nextQueryId;
    storage.nextQueryId = queryId + 1;
    val out = createMessage({
        bounce: BounceMode.NoBounce,
        dest:   storage.fortuna,
        value:  ton("0.25"),         // ≥ 0.22 floor + slack at default tunables
        body:   RequestRandomness { queryId, seed: someEntropy, callbackGas: ton("0.05") },
    });
    out.send(SEND_MODE_REGULAR);
    storage.save();
}
```

## Python trigger (off-chain or testnet smoke test)

When the request is initiated from off-chain — e.g. a backend that pulls the trigger on a user action — use the Python SDK:

```python
import asyncio
from pytoniq import LiteBalancer
from pytoniq_core import Address
from fortuna_sdk import Fortuna, FORTUNA_TESTNET, QueryIdStream

TON = 10**9


async def request_randomness(user_seed: int):
    rpc = LiteBalancer.from_testnet_config(trust_level=2)
    await rpc.start_up()

    fortuna = Fortuna.create_from_address(FORTUNA_TESTNET.fortuna, client=rpc)
    ids = QueryIdStream()

    # consumer_wallet is your funded wallet that will pay the request fee
    # (e.g. await WalletV5R1.from_mnemonic(rpc, mnemonic.split())).
    await fortuna.send_request_randomness(
        consumer_wallet,
        value=int(0.25 * TON),
        query_id=ids.next(),
        seed=user_seed,            # plain int — the SDK widens it to uint256
        callback_gas=int(0.05 * TON),
    )

    await rpc.close_all()
```

`QueryIdStream` is a monotone u64 generator that avoids `time.time_ns()` collisions (which trigger exit 120 DuplicateRequest).

For a contract-initiated trigger (the more common production path), the Tolk skeleton above is what runs — Python is for testing, ops, or off-chain triggers.

## Verify a beta off-chain (consumer-side audit)

When you want to sanity-check the VRF output your callback received:

```python
from fortuna_sdk import compute_alpha, compute_beta
from pytoniq_core import Address

# What you have on-chain after fulfillment
consumer = Address("EQ...")
query_id = 42
seed = 0xDEADBEEF
creation_lt = 1_000_000   # from EvtRequestCreated.creation_lt
agg_sig = bytes.fromhex("...")  # 96-byte aggregate sig
expected_beta = bytes.fromhex("...")  # what your callback received

alpha = compute_alpha(consumer, query_id=query_id, seed=seed, creation_lt=creation_lt)
beta = compute_beta(agg_sig, consumer, query_id=query_id, seed=seed, creation_lt=creation_lt)

assert beta == expected_beta, "VRF integrity broken — beta drift between on-chain and SDK"
```

Use this in a periodic audit job to detect indexer drift / stale SDK / contract upgrade desync.

## Common pitfalls (most-frequent reverts)

| Exit | Name | Cause | Fix |
|---|---|---|---|
| 120 | E_DUPLICATE_REQUEST | Same `(consumer, query_id)` already pending | Use `QueryIdStream` or monotone counter; never re-use before fulfillment |
| 121 | E_REQUEST_NOT_FOUND | Fulfill/Reclaim/Prune for unknown key | Either never created OR already settled |
| 122 | E_DEADLINE_NOT_REACHED | Reclaim before TTL elapses | Wait `request_ttl` (default 1 h); see `await fortuna.get_config()` |
| 123 | E_STALE_EPOCH | Request placed before Atlas rotation, fulfilled after | Reclaim for refund; D-008 |
| 124 | E_GROUP_KEY_NOT_SET | Fortuna never received Atlas bootstrap | Owner-side: `send_sync_atlas` to pull |
| 240 | E_INSUFFICIENT_VALUE | `value` below request floor | Bump value; floor is 0.22 TON at defaults for `callback_gas=0.05` |
| 161 | E_INVALID_BLS_SIGNATURE | Off-chain DST drift OR stale pkShare | The Python SDK's `sign_alpha` already pins `BLS_DST_G2_POP` — drift only happens with custom signers |

## Modulo bias

`beta % N` is **provably unbiased only for N that divides 2^256**. For small N (≤ 256), the bias is negligible. For cryptographic rigor on arbitrary N, rejection-sample on the consumer side (Tolk) — not on the Python side, since the consumer code is what touches `beta` directly:

```tolk
val cap: int = 256 / 100 * 100;
val byte: int = msg.beta & 0xFF;
val pick: int = byte < cap ? byte % 100 : <re-request or use a different bit window>;
```

## Reclaim — recover stuck requests

If operators don't fulfill within `request_ttl` (default 1 h), anyone can call `ReclaimRequest` and the consumer gets the fee back:

```python
await fortuna.send_reclaim_request(
    consumer_wallet,
    value=int(0.05 * TON),
    consumer=consumer_addr,
    query_id=query_id,
)
```

Stale-epoch requests (where Atlas rotated mid-flight) auto-abort on fulfill attempts; reclaim is the only recovery path.

## Reference reading

- **Live testnet**: `from fortuna_sdk import FORTUNA_TESTNET, assert_deployment`. `assert_deployment("testnet")` validates the SDK's expected schema matches the live deploy.
- **VRF primitives**: `compute_alpha`, `compute_beta`, `compute_req_key`, `sign_alpha`, `aggregate_signatures`, `aggregate_group_public_key`, `bls_public_key`, `random_bls_secret` — byte-identical to the on-chain Tolk helpers.
- **Worked example (TS)**: the Fortuna repo includes a coin-flip example — the Tolk part is universal; consume from Python via the trigger pattern above.

## Related skills

- `/titon:fortuna-debug-exit-code-python` — diagnose any revert by exit code.
- `/titon:fortuna-monitor-events-python` — watch `RequestCreated → RequestFulfilled` lifecycle.
- `/titon:kronos-register-job-python` — if you also want a recurring trigger that calls into your consumer.
