---
name: themis-integrate-consumer
description: Write a Tolk consumer contract that processes Themis RevealCallback (0x95) — receive a batched sealed-bid reveal, walk the per-bid table, run domain logic. Use when the user says "integrate Themis", "build a sealed-bid auction / DEX / vote on TON", "consume RevealCallback", "write a Themis consumer", "make my dapp MEV-resistant".
---

# Integrate a Themis consumer

You're adding sealed-bid orderflow to a Tolk contract. Themis's consumer surface is small: **one inbound (`RevealCallback 0x95`)**, no outbounds — bidders go straight to the chamber via `SubmitCiphertext 0x91`, and the chamber dispatches the post-reveal callback to you.

This is the keystone skill for building on Themis. Read it through once; it answers "what shape is my contract?", "how do I decrypt bids?", "what's the callback safety checklist?".

## The wire shapes (frozen, byte-identical)

```tolk
// Inbound — what the chamber sends to your consumer.
struct (0x00000095) RevealCallback {
    roundId:    uint64
    chamber:    address          // the chamber sending this (sender pin against it)
    payloadRef: Cell<RevealPayload>
}

struct RevealPayload {
    bidCount: uint32
    entries:  map<uint32, Cell<RevealedBid>>   // idx → bid
}

// One entry per bid that was successfully decrypted.
struct RevealedBid {
    sender:      address          // the bidder who submitted the ciphertext
    queryId:     uint64           // bidder-supplied; carry whatever you want here
    submittedAt: uint32           // unix seconds; chamber timestamps at SubmitCiphertext
    aeadCt:      cell             // Aead { version, nonce, payload } — encrypted bid blob
    ephPair:     Cell<EphPair>    // c1 + D for off-chain decryption
}

struct EphPair {
    c1: bits384  // bidder's ElGamal ephemeral (48-byte G1 compressed)
    D:  bits384  // groupSk · c1 — the operator-published decryption share
}
```

The TS-side decoder is `decodeRevealCallback(body)` → `DecodedRevealCallback`. The chamber wrapper's `getCurrentRound()` covers state queries; per-round bids are NOT readable from a chamber getter — you receive them only via `RevealCallback`.

## The on-chain dispatch shape — copy this template

```tolk
tolk 1.3

import "themis-messages.tolk"        // or copy the RevealCallback struct verbatim
import "themis-opcodes.tolk"

const MY_STORAGE_VERSION: uint8 = 1;

const E_MY_NOT_CHAMBER:   int = 200;
const E_MY_ROUND_REPLAY:  int = 251;
const E_MY_BAD_VERSION:   int = 119;

struct MyStorage {
    schemaVersion:   uint8
    boundChamber:    address      // pin: only this chamber may deliver callbacks
    lastFilledRound: uint64        // monotonic; replay-defense
    // … your domain state here (reserves, pending settlements, etc.)
}

fun MyStorage.load(): MyStorage  { return MyStorage.fromCell(contract.getData()); }
fun MyStorage.save(self)         { contract.setData(self.toCell()); }
fun MyStorage.assertVersion(self) {
    assert (self.schemaVersion == MY_STORAGE_VERSION) throw E_MY_BAD_VERSION;
}

// One-shot owner-only bind-chamber message + RevealCallback in the dispatch
// union. Tolk's `lazy AllowedMessage.fromSlice` walks the union and the
// `match` arm dispatches to the typed handler.
type AllowedMessage = RevealCallback | SetBoundChamber;

struct (0xDEAD0001) SetBoundChamber { newChamber: address }

fun onInternalMessage(in: InMessage) {
    if (in.body.remainingBitsCount() == 0 && in.body.remainingRefsCount() == 0) return;
    val msg = lazy AllowedMessage.fromSlice(in.body);
    match (msg) {
        RevealCallback   => handleRevealCallback(in.senderAddress, msg),
        SetBoundChamber  => handleSetBoundChamber(in.senderAddress, msg),
        else             => { return; }
    }
}

fun handleRevealCallback(sender: address, rc: RevealCallback) {
    var storage = lazy MyStorage.load();
    storage.assertVersion();

    // (1) Sender pin — only the bound chamber may deliver.
    assert (sender == storage.boundChamber) throw E_MY_NOT_CHAMBER;

    // (2) Replay defense — strictly-monotonic round-id.
    assert (rc.roundId >= storage.lastFilledRound) throw E_MY_ROUND_REPLAY;

    val payload = rc.payloadRef.load();

    // (3) Walk the entries map.
    var iter = payload.entries.findFirst();
    while (iter.isFound) {
        val bid = iter.loadValue().load();
        // bid.sender, bid.queryId, bid.submittedAt, bid.aeadCt, bid.ephPair
        // → your domain logic here.
        iter = payload.entries.iterateNext(iter);
    }

    storage.lastFilledRound = rc.roundId + 1;
    storage.save();
}
```

Anchor your contract to **this exact shape**. The 4 safety steps (assertVersion, sender pin, replay defense, bid iteration) are non-negotiable.

## Decryption — two patterns, pick one

**TVM has no ChaCha20-Poly1305 primitive.** Your Tolk contract CANNOT call `decryptBid` on-chain. You have two production patterns:

### Pattern 1 — plaintext-in-queryId (fast path)

Encode the bid intent into the **64-bit `queryId`** of `SubmitCiphertext`. The AEAD payload hides only the bidder's identity-binding salt; the intent (side, amount, vote, etc.) is in the queryId where the consumer can read it directly.

Best for: sealed-bid English auctions (queryId = bid amount), uniform-price DEX swaps (queryId = `side|amount`), commit-reveal voting where the intent fits in 64 bits.

**Example:** [`contracts/sealed-amm.tolk`](../../../contracts/sealed-amm.tolk) — `SealedAMM` reference consumer, plaintext-in-queryId:

```tolk
// 64-bit layout:
//   bit 63    : side (0 = BUY, 1 = SELL)
//   bits 0..62 : amount (input-token quantity, nanoTON)

const QUERY_SIDE_BIT:    int = 0x8000000000000000;
const QUERY_AMOUNT_MASK: int = 0x7FFFFFFFFFFFFFFF;

val q: int = bid.queryId;
val side: uint8 = (q & QUERY_SIDE_BIT) == 0 ? SIDE_BUY : SIDE_SELL;
val amountIn: coins = q & QUERY_AMOUNT_MASK;
```

**Trade-off:** the encryption still hides *who* the bidder is until reveal time (the AEAD payload binds the sender + salt), but the *bid* is publicly inspectable in the SubmitCiphertext body. For pure MEV defense (front-running between commit and reveal) this is sufficient — the operator quorum can't see who submitted what until they decrypt, and bidders submit independently.

### Pattern 2 — off-chain settler (full privacy)

Run an off-chain settler that watches `RevealCallback` events and AEAD-decrypts each bid. The settler posts a separate settlement message back to the consumer with the decoded intent. This pattern is necessary when the bid plaintext doesn't fit in 64 bits or when intent privacy is required between bidder and settler.

```typescript
// Off-chain (your indexer / settler):
import {
    decodeRevealCallback,
    decryptBid,
} from '@titon-network/themis-sdk';

// Watch chamber's outbound 0x95 messages (or just bind to the consumer's inbox).
for (const tx of recentChamberTxs) {
    const out = tx.outMessages.find((m) => m.info.dest?.equals(consumer));
    if (!out) continue;
    const decoded = decodeRevealCallback(out.body);

    for (const bid of decoded.bids) {
        try {
            const plaintext = decryptBid({
                aeadCell: bid.aeadCt,
                c1:       bid.c1,
                D:        bid.D,
                chamber:  decoded.chamber,
                roundId:  decoded.roundId,
            });
            const intent = JSON.parse(plaintext.toString('utf8'));
            await postSettlement(consumer, decoded.roundId, bid.idx, intent);
        } catch (e) {
            // Poly1305 MAC failed → dishonest D from operators. Skip the bid
            // (consumer never sees a settlement for it). Chamber state unaffected.
        }
    }
}
```

The settler's `postSettlement` message is YOUR ABI — it can be permissioned (only-settler) or permissionless if the math is publicly verifiable. The chamber doesn't enforce a settlement back-channel; that's your protocol's concern.

**Recommendation:** start with **Pattern 1**. Migrate to Pattern 2 if you genuinely need plaintext > 64 bits, OR if your protocol needs end-to-end intent privacy. Most dapps (sealed auctions, DEX clearing, simple votes) fit in 64 bits.

## The callback safety checklist

Every `handleRevealCallback` must check, in order:

1. **`assertVersion()`** — fence against an in-flight storage migration.
2. **Sender pin** — `assert (sender == storage.boundChamber)`. Without this, any address can deliver a fake 0x95 to your consumer. The biggest hole.
3. **Round monotonicity** — `assert (rc.roundId >= storage.lastFilledRound)`. Defends against a replayed callback if the chamber's outbound was somehow re-delivered (defensive — production chambers don't replay, but the assertion is cheap).
4. **Cheap walk** — the entries map can hold up to `maxBidsPerRound` entries (default 8, max ~32 in practice before gas budget bites). Keep per-bid work small; defer heavy computation to a separate message or off-chain.
5. **Revert-safe** — callbacks arrive with `bounce: false`. A revert here does NOT bounce to the chamber; the chamber's state has already advanced. Avoid unnecessary asserts in the hot path.
6. **Per-bid defensive guards** — Tolk 1.3 has no user-facing `try / catch`. To prevent one bad bid from reverting the whole batch, guard with `if` before any operation that could throw: skip zero-amount fills, skip out-of-range queryIds, etc. Pattern from `sealed-amm.tolk`:

   ```tolk
   if (amountIn > 0) {
       // compute fill; only this path mutates reserves
   }
   emitEvent(EvtFilled { ... });   // emit either way for observability
   ```

   Asserts in the iteration body abort the entire `RevealCallback` — no settlement happens for any bid in the round. Reserve `assert` for invariants you genuinely cannot recover from.

## Wire your consumer to a chamber

There's a **circular dependency** at deploy time: the chamber needs the consumer's address; the consumer (if it stores `boundChamber`) needs the chamber's address. Three patterns:

### Option A — one-shot owner-only `SetBoundChamber` (recommended, what SealedAMM does)

1. Deploy your consumer with `isBound = false`, `boundChamber = <zero placeholder>`.
2. Call `factory.sendDeployChamber(..., consumer: yourConsumer.address, ...)`. Watch the `EvtChamberDeployed` external log for the new chamber address.
3. Owner sends `SetBoundChamber { newChamber: <chamber address> }` to your consumer. One-shot — `isBound` flips to true, subsequent calls revert.

### Option B — predict the chamber address ahead of deploy

Themis chamber addresses are deterministic: `address.fromWorkchainAndHash(0, hashCell(stateInit(factory, serial, code)))`. Predict the next chamber's address (factory's `getNumChambers()` is its next serial) and pass it as `consumer:` at construction time. Riskier — racy with concurrent chamber deploys.

### Option C — store `boundChamber` as `address?` (option type) and dispatch on null

Cleaner for some patterns but adds a runtime null check on every callback. Generally Option A is preferred.

## Sending bidder-side messages (for completeness)

The dapp dev's job ends at the consumer contract. The bidder UX is a separate concern — load [`skills/themis-bidder-flow.md`](./themis-bidder-flow.md) for the encrypt + submit pattern. The 90-second summary:

```typescript
import { encryptBid } from '@titon-network/themis-sdk';

const { groupPk } = await chamber.getGroupKey();
const { roundId } = await chamber.getCurrentRound();
const enc = encryptBid({
    plaintext: Buffer.alloc(0),                  // empty if using plaintext-in-queryId
    groupPk, chamber: chamber.address, roundId,
});
const cfg = await chamber.getConfig();
// POSITIONAL signature: (via, value, queryId, c1, aeadCell).
await chamber.sendSubmitCiphertext(
    wallet.sender(),
    cfg.submitFee + toNano('0.05'),
    encodeBidIntent({ side, amount }),
    enc.c1,
    enc.aeadCell,
);
```

## Testing your consumer

Use the SDK's test helpers — sandbox + mock-Atlas + mock-ForgeTON + real chamber:

```typescript
import { Blockchain } from '@ton/sandbox';
import { toNano } from '@ton/core';
import {
    encryptBid,
    buildReveal,
    randomGroupKey,
    OP_REVEAL_CALLBACK,
} from '@titon-network/themis-sdk';

// Deploy + admit factory, deploy chamber bound to your consumer, mirror an
// operator via the mock-ForgeTON, submit a bid, advance to reveal phase,
// build + send a RevealRound, assert your consumer's state changed.
// Full template: tests/SealedAMMConsumer.spec.ts in the parent repo.
```

The parent repo's [`tests/SealedAMMConsumer.spec.ts`](../../tests/SealedAMMConsumer.spec.ts) is a complete worked example — copy-paste the deploy pipeline and customize the assertions for your protocol.

## Reading the consumer's perspective on common errors

When your consumer reverts on a callback, the chamber's tx will show `success: false` for the outbound message. Common cases:

| Code | Cause | Fix |
|------|-------|-----|
| 200 | Wrong sender — someone other than the bound chamber tried to deliver | Verify `assert (sender == storage.boundChamber)` is the FIRST check after `assertVersion()`. |
| 251 | Replay — `rc.roundId < storage.lastFilledRound` | Your `lastFilledRound` advanced past `rc.roundId`. Could mean the chamber re-delivered (defensive — should never happen) or your test is feeding rounds out of order. |
| 119 | Schema-version mismatch | You bumped `MY_STORAGE_VERSION` mid-deploy or are loading storage from a pre-bump deployment. Re-deploy or migrate. |

Use `explainError(code)` from the SDK to surface a one-line hint, or [`ERRORS.md`](../ERRORS.md) for the full table.

## Watch out for

- **Don't iterate the entries map with `forEach`.** Tolk's dict iterator is `findFirst` / `iterateNext`. The map is keyed by `uint32` idx (the per-round bid index), so iteration order is sorted-by-idx.
- **Don't expect bids in submission order.** The chamber stores bids by index (assigned at submit time), but operators bundle ALL bids in the reveal — so your callback gets the full set at once, idx-sorted. There's no per-bid callback ordering control.
- **Don't trust `bid.submittedAt` as the operator's signing time.** It's set when the chamber received the SubmitCiphertext, NOT when the reveal happened. For round-end timing, read `rc.roundId` and look up the chamber's `revealEta` getter (if needed).
- **Don't store the `aeadCt` ref in your consumer.** It's `~80-200 bytes` per bid; multiplied by `maxBidsPerRound` over many rounds, you'll bloat your contract's storage rent. Discard the `aeadCt` reference once your settler has decrypted off-chain.
- **Don't reuse `queryId` semantics across protocols.** A 64-bit queryId in your consumer means whatever you say it means. The chamber doesn't enforce a schema. Document yours and pin it in your consumer's storage docs.
- **Don't deploy a chamber with `consumer: addr_none`.** The chamber stores it but `RevealRound` will revert with `E_CONSUMER_NOT_SET (173)`. Always wire the consumer at deploy.
- **Don't forget the WC=0 constraint.** Your consumer MUST live on workchain 0. Themis is workchain-0-only across the stack.

## Cross-references

- [`contracts/sealed-amm.tolk`](../../../contracts/sealed-amm.tolk) — reference consumer with plaintext-in-queryId clearing math (the most complete worked example).
- [`tests/SealedAMMConsumer.spec.ts`](../../../tests/SealedAMMConsumer.spec.ts) — sandbox spec end-to-end: deploy, bid, reveal, callback, assert clearing.
- [`AGENTS.md`](../AGENTS.md) — full SDK surface + protocol diagram.
- [`skills/themis-bidder-flow.md`](./themis-bidder-flow.md) — the bidder-side counterpart to this skill.
- [`skills/themis-deploy-chamber.md`](./themis-deploy-chamber.md) — how to deploy + configure the chamber that talks to your consumer.
- [`skills/themis-debug.md`](./themis-debug.md) — error-code lookup + common gotchas.
- [`GUARANTEES.md`](../GUARANTEES.md) — trust model (integrity-by-AEAD, liveness-by-refund).
