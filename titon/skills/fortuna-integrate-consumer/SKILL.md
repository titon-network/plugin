---
name: fortuna-integrate-consumer
description: Add verifiable randomness to a TON product via Fortuna VRF — write the 0x50 VrfCallback receiver, send 0x30 RequestRandomness, size the value correctly. Use when the user is building anything that needs unbiasable on-chain randomness (raffle, NFT trait roll, dice, lottery, mystery box, shuffled queue) and wants to consume Fortuna.
---

# Integrate Fortuna VRF into a consumer contract

You're helping a product dev add verifiable randomness. Fortuna is TON's threshold-BLS VRF — sovereign and on-TON. The consumer side is a 5-rule pattern: pin Fortuna, sender-gate the callback, generate unique queryIds, fund the request, keep the callback revert-proof.

## The fastest path: clone the worked example

The Fortuna SDK ships a complete coin-flip dapp that demonstrates the full lifecycle. **Recommend this first** — it's faster than writing from scratch:

```bash
npm install @titon-network/fortuna-sdk @ton/core
# In your Blueprint project root:
cp node_modules/@titon-network/fortuna-sdk/examples/coin-flip/coin-flip.tolk     contracts/
cp node_modules/@titon-network/fortuna-sdk/examples/coin-flip/CoinFlip.compile.ts wrappers/
cp node_modules/@titon-network/fortuna-sdk/examples/coin-flip/CoinFlip.ts         wrappers/
cp node_modules/@titon-network/fortuna-sdk/examples/coin-flip/CoinFlip.spec.ts    tests/
npx blueprint test tests/CoinFlip.spec.ts
```

The test passes against `deployFortunaFixture` (no testnet wallet, no funding). Adapt the storage + business logic to your domain.

## Or scaffold a fresh starter

```bash
npx fortuna init                  # writes FORTUNA.md AI-context to repo root
npx fortuna scaffold consumer     # drops templates/consumer.tolk into your cwd
npx fortuna estimate request --callback-gas 0.05   # exact request-value floor
```

`FORTUNA.md` is the dense AI-friendly brief — Claude reads it and generates correct integration code on the first try.

## Wire types — copy verbatim from `@titon-network/fortuna-sdk` source

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
3. **Unique queryId per pending request.** Reusing one before fulfillment reverts at Fortuna with `E_DUPLICATE_REQUEST = 120`. Use a monotone counter on-chain or `QueryIdStream` off-chain.
4. **Funded request value.** At FORTUNA_DEFAULTS, the floor is `baseRequestFee + submitterReward + callbackGas + minForwardReserve` = **0.22 TON** for `callbackGas = 0.05`. Add ~0.03 TON forward-fee slack → forward 0.25 TON. Below the floor reverts with `E_INSUFFICIENT_VALUE = 240`.
5. **Revert-proof callback.** Fortuna sends `VrfCallback` with `bounce: false` — a revert in your handler does NOT bounce, but the request state IS already cleared, so failed callbacks lose the forwarded gas. Keep it cheap and assertion-light.

## Tolk skeleton

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

## TypeScript trigger (off-chain or test)

```ts
import { Fortuna, FORTUNA_TESTNET, QueryIdStream } from '@titon-network/fortuna-sdk';
import { toNano } from '@ton/core';

const fortuna = tonClient.open(Fortuna.createFromAddress(FORTUNA_TESTNET!.fortuna));
const ids = new QueryIdStream();

await fortuna.sendRequestRandomness(consumerSender, {
    value: toNano('0.25'),
    queryId: ids.next(),
    seed: 0xc0ffeen,
    callbackGas: toNano('0.05'),
});
```

## Common pitfalls (most-frequent reverts)

| Exit | Name | Cause | Fix |
|---|---|---|---|
| 120 | E_DUPLICATE_REQUEST | Same `(consumer, queryId)` already pending | Use a monotone counter; never re-use before fulfillment |
| 121 | E_REQUEST_NOT_FOUND | Fulfill/Reclaim/Prune for unknown key | Either never created OR already settled |
| 122 | E_DEADLINE_NOT_REACHED | Reclaim before TTL elapses | Wait `requestTtl` (default 1 h); see `getConfig()` |
| 123 | E_STALE_EPOCH | Request placed before Atlas rotation, fulfilled after | Reclaim for refund; D-008 |
| 124 | E_GROUP_KEY_NOT_SET | Fortuna never received Atlas bootstrap | Owner-side: `sendSyncAtlas` to pull |
| 240 | E_INSUFFICIENT_VALUE | `value` below request floor | Bump value; floor is 0.22 TON at defaults for callbackGas=0.05 |
| 161 | E_INVALID_BLS_SIGNATURE | Off-chain DST drift OR stale pkShare | Pin `BLS_DST_G2_POP`; rotate the share |

## Modulo bias

`beta % N` is **provably unbiased only for N that divides 2^256**. For small N (≤ 256), the bias is negligible. For cryptographic rigor on arbitrary N, rejection-sample:

```tolk
// Fair pick from [0, 100) without modulo bias:
val cap: int = 256 / 100 * 100;       // largest multiple of 100 ≤ 256
val byte: int = msg.beta & 0xFF;
val pick: int = byte < cap ? byte % 100 : <re-request or use a different bit window>;
```

## Reclaim — recover stuck requests

If operators don't fulfill within `requestTtl` (default 1 h), anyone can call `ReclaimRequest` and the consumer gets the fee back:

```ts
await fortuna.sendReclaimRequest(consumerSender, {
    value: toNano('0.05'),
    consumer: consumerAddr,
    queryId,
});
```

Stale-epoch requests (where Atlas rotated mid-flight) auto-abort on fulfill attempts; reclaim is the only recovery path.

## Reference reading

- **Worked example:** `node_modules/@titon-network/fortuna-sdk/examples/coin-flip/` — full Tolk + TS + sandbox spec.
- **Bare template:** `node_modules/@titon-network/fortuna-sdk/templates/consumer.tolk` — comment-rich starter.
- **Deep AI brief:** `node_modules/@titon-network/fortuna-sdk/llms-full.txt` — load this into your assistant's context for one-shot integration.
- **Live testnet:** `import { FORTUNA_TESTNET, assertDeployment } from '@titon-network/fortuna-sdk'`.

## Related skills

- `/titon:fortuna-debug-exit-code` — diagnose any revert by exit code.
- `/titon:fortuna-monitor-events` — watch `RequestCreated → RequestFulfilled` lifecycle.
- `/titon:kronos-register-job` — if you also want a recurring trigger that calls into your consumer.
