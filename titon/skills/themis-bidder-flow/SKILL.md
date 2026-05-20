---
name: themis-bidder-flow
description: Encrypt a sealed bid under a Themis chamber's BLS group key and submit it on-chain via SubmitCiphertext (0x91). Use when the user says "submit a Themis bid", "build a bidder UI", "encrypt a sealed bid", "send SubmitCiphertext", "integrate Themis from a wallet", or "make my frontend submit sealed bids".
---

# Submit a sealed bid to a Themis chamber

You're building the **bidder side** of a Themis-powered dapp — typically a wallet, frontend, or order-router that:

1. Reads the chamber's current `groupPk` + `roundId`.
2. Encrypts the user's bid under that key.
3. Sends `SubmitCiphertext (0x91)` to the chamber with the right value attached.

The skill complements [`themis-integrate-consumer.md`](./themis-integrate-consumer.md) (which is the on-chain consumer story). Together they're the two halves of a Themis dapp: bidder UX + consumer settlement.

## The wire shape

```tolk
// Outbound to chamber.
struct (0x00000091) SubmitCiphertext {
    queryId:  uint64                  // bidder-supplied; carries plaintext intent in Pattern 1
    c1:       bits384                 // 48-byte G1 compressed (ElGamal ephemeral) — inline
    aeadRef:  Cell<Aead>              // AEAD blob (nonce + ChaCha20-Poly1305 ciphertext)
}

// AEAD payload is INLINE (no ref) inside the Aead cell:
//   storeUint(version, 8) | storeBuffer(nonce, 12B) | storeBuffer(ciphertext + 16B tag)
// The SDK's `encryptBid` builds this exact layout via `beginCell()…endCell()`.
```

## The 5-line happy path

```typescript
import { encryptBid } from '@titon-network/themis-sdk';
import { toNano } from '@ton/core';

const { groupPk } = await chamber.getGroupKey();   // 48-byte G1, current epoch
const { roundId } = await chamber.getCurrentRound();
const cfg         = await chamber.getConfig();      // includes submitFee + tunables

const enc = encryptBid({
    plaintext: Buffer.from(JSON.stringify({ side: 'buy', size: 100 })),
    groupPk, chamber: chamber.address, roundId,
});

// Wrapper signature is POSITIONAL: (via, value, queryId, c1, aeadCell).
await chamber.sendSubmitCiphertext(
    wallet.sender(),
    cfg.submitFee + toNano('0.05'),   // value: fee + gas headroom
    myQueryId(),                       // queryId — see "queryId" below
    enc.c1,
    enc.aeadCell,
);
```

That's the happy path. The rest of this skill explains the four things you have to get right around it.

## (1) Always read `groupPk` fresh

```typescript
const { groupPk, groupEpoch } = await chamber.getGroupKey();
```

**Why fresh?** Atlas rotations bump `groupEpoch` and rewrite `groupPk`. A bid encrypted under epoch N is ONLY decryptable under epoch N — if you cache a stale `groupPk` and an Atlas rotation lands between your read and your submit, **your bid still submits (the chamber doesn't bind c1 to an epoch on the SubmitCiphertext path)** but operators sign reveals under the *current* epoch, and the bidder's encryption no longer decrypts. Your bid silently fails the AEAD MAC at the consumer side.

**Rule:** read `groupPk` no more than one round before you submit. If your UX has a long "stage bid → sign → submit" flow, re-read on each step's resume.

## (2) Compute the attached value correctly

The most common failure mode is `E_INSUFFICIENT_VALUE (240)`:

```typescript
const cfg = await chamber.getConfig();
const submitValue = cfg.submitFee + toNano('0.05');
//                  fee in pool   + gas + forward-fee slack
```

`cfg.submitFee` is read live — it can drift under owner `UpdateConfig`. **Always read it.** Don't hardcode.

If you're under attack from gas spikes, bump the slack to `toNano('0.1')` — over-attach refunds via `mode 64 + 2` (CARRY_REMAINING_VALUE | IGNORE_ERRORS).

## (3) Pick a `queryId` scheme

Two production patterns — same as for the consumer side (see [`themis-integrate-consumer.md`](./themis-integrate-consumer.md)):

### Pattern 1 — plaintext-in-queryId

The chamber doesn't enforce a queryId schema. Your dapp defines it. Common encoding (SealedAMM-style):

```typescript
function encodeAmmBid({ side, amount }: { side: 'buy' | 'sell'; amount: bigint }): bigint {
    const sideBit = side === 'sell' ? 1n << 63n : 0n;
    const amt     = amount & ((1n << 63n) - 1n);
    return sideBit | amt;
}
```

The consumer reads `bid.queryId` directly in its `RevealCallback` handler and decodes. The encrypted `aeadCt` is effectively a sealed bidder-binding tag.

### Pattern 2 — opaque queryId + plaintext-in-AEAD

```typescript
// Themis SDK doesn't ship a queryId stream helper — roll your own counter
// (the chamber doesn't enforce uniqueness; uniqueness is your settler's concern).
let next = 1n;
const queryId = next++;     // monotone-unique per-bidder
```

The full intent lives in the AEAD plaintext; the consumer's off-chain settler decrypts and posts settlement messages back to the consumer. Used when the bid plaintext > 64 bits or end-to-end intent privacy matters.

**Pick one and stick with it.** Don't mix patterns in the same chamber — the consumer's parser assumes one schema.

## (4) Listen for the EvtBidSubmitted event

After `sendSubmitCiphertext` resolves, decode the event log:

```typescript
import { decodeEvents, EVT_BID_SUBMITTED } from '@titon-network/themis-sdk';

const events = decodeEvents(tx.externals.map((e) => e.body));
const myBid = events.find(
    (e) => e.kind === 'BidSubmitted' && e.queryId === queryId,
);
if (!myBid || myBid.kind !== 'BidSubmitted') {
    throw new Error('SubmitCiphertext did not emit EvtBidSubmitted');
}
// myBid.idx is the per-round index assigned to your bid.
// You'll see this same idx in the RevealCallback's entries map at reveal time.
```

The `idx` is your bid's slot in the round. The consumer iterates `entries` by `idx`, so if your dapp tracks per-user bid status, key it by `(roundId, idx)`.

## (5) Handle the windows

The chamber state machine has three phases per round, computed from `(now, commitEta, revealEta)`:

| Phase | Window | What you can do |
|-------|--------|-----------------|
| Commit | `now < commitEta` | `SubmitCiphertext` accepted. |
| Reveal | `commitEta <= now < revealEta` | `SubmitCiphertext` rejects with `E_COMMIT_WINDOW_CLOSED (152)`; operators race to send `RevealRound`. |
| Refundable | `now >= revealEta` | Anyone may `AdvanceRound`; round refunds + transitions. |

Before submitting, derive phase from `getCurrentRound()`:

```typescript
const { roundId, commitEta, revealEta } = await chamber.getCurrentRound();
//                ^^^ NOTE: the SDK getter returns ONLY these three fields.
//                The Tolk `getRound(): RoundBlob` getter has `phase` + `bidCount` but
//                is NOT yet wrapped — agents waiting on `chamber.getRound()` should
//                catch `E_COMMIT_WINDOW_CLOSED` from the send instead, or compute the
//                window from `now`.
const now = Math.floor(Date.now() / 1000);
if (now >= commitEta) {
    throw new Error('commit window closed; wait for next round');
}
// `bidCount` is not exposed via SDK — if your dapp needs capacity-aware UX,
// rely on `E_BIDS_PER_ROUND_EXCEEDED (158)` from the chamber + retry next round.
```

If `commitEta` is < ~10s away, consider waiting for the next round — your bid might land just after closure and revert.

## End-to-end working example

```typescript
import { Address, toNano } from '@ton/core';
import {
    encryptBid,
    decodeEvents,
    EVT_BID_SUBMITTED,
} from '@titon-network/themis-sdk';

async function submitSealedBid(opts: {
    chamber: ThemisChamber;
    wallet:  WalletV5;
    bid:     { side: 'buy' | 'sell'; amount: bigint };
}) {
    const { chamber, wallet, bid } = opts;

    // 1. Read state fresh.
    const { groupPk, groupEpoch } = await chamber.getGroupKey();
    const round = await chamber.getCurrentRound();  // { roundId, commitEta, revealEta }
    const cfg   = await chamber.getConfig();

    // 2. Pre-flight checks. (No phase/bidCount SDK — derive from clock.)
    const nowSec = Math.floor(Date.now() / 1000);
    if (nowSec >= round.commitEta) {
        throw new Error(`commit window closed (now=${nowSec} >= commitEta=${round.commitEta})`);
    }

    // 3. Encrypt. Plaintext is empty in Pattern 1 (intent rides in queryId).
    const enc = encryptBid({
        plaintext: Buffer.alloc(0),
        groupPk,
        chamber:   chamber.address,
        roundId:   round.roundId,
    });

    // 4. Encode the intent + send. Positional Send signature.
    const queryId = encodeAmmBid(bid);
    const tx = await chamber.sendSubmitCiphertext(
        wallet.sender(),
        cfg.submitFee + toNano('0.05'),
        queryId,
        enc.c1,
        enc.aeadCell,
    );

    // 5. Verify the event landed (catch bounced txs early).
    const events = decodeEvents(tx.externals.map((e) => e.body));
    const ev = events.find((e) => e.kind === 'BidSubmitted' && e.queryId === queryId);
    if (!ev || ev.kind !== 'BidSubmitted') {
        throw new Error('bid not recorded — check tx exit code');
    }

    return { roundId: round.roundId, idx: ev.idx, groupEpoch };
}
```

## Receipt: tracking your bid

Store `(chamberAddress, roundId, idx, queryId)` per user-submitted bid. When the round reveals, your indexer matches the `RevealCallback` entry by `(roundId, idx)` and surfaces the outcome to the user — fill price, settlement amount, refund, etc.

Per `themis-integrate-consumer.md`, the consumer iterates the bid table and acts; the user's wallet doesn't need to do anything else after submitting.

## What happens if I don't get a reveal?

Two scenarios:

1. **Operator quorum failed (no `RevealRound` by `revealEta`).** Anyone can call `chamber.sendAdvanceRound(via, value)` to trigger the refund branch — `submitFee` returns to every bidder. The operator set is *carrot-only*; if operators went offline, the protocol fails open (refunds).
2. **Operators submitted a malformed `D` for your specific bid.** AEAD MAC fails at the consumer's settler. Your bid is silently ignored (no settlement). Other bids in the same round proceed normally. **This is integrity-by-AEAD — not a refund event.** `ChallengeReveal` + on-chain cryptographic slashing is RESERVED for a future revision.

For monitoring: listen for `EvtRoundExpired (0xB6)` (refund-fired) and `EvtRoundRevealed (0xB4)` (normal reveal). Together they cover both outcomes.

## Watch out for

- **Don't reuse a `c1` ephemeral across bids.** `newEphemeral()` is called inside `encryptBid` by default — leave it alone. If you override `ephemeral:` for testing, never reuse a value: the ElGamal one-time-pad collapses to plaintext recovery when `c1` repeats with the same recipient.
- **Don't sign the AEAD payload from a wallet that's also the bidder.** The chamber doesn't bind the sender's signature to the AEAD content — the bidder's identity is in the message envelope, not the AEAD plaintext. If you need a sender-binding receipt, hash it into the plaintext yourself.
- **Don't submit during reveal phase expecting it to "queue".** `E_COMMIT_WINDOW_CLOSED (152)` reverts the tx outright — bidders wait for the next round (which starts after the chamber processes `RevealRound` or `AdvanceRound`).
- **Don't read `groupPk` from a getter on the FACTORY when the chamber has its own cached key.** Factory holds `groupKey` for fan-out; the chamber's cached `groupPk` is the authoritative one operators sign under. Always call `chamber.getGroupKey()`, not `factory.getGroupKey()`, for encryption.
- **Don't hand-build the `Aead` cell.** `encryptBid` derives the key with HKDF bound to `(chamber.hash, roundIdBE)` — any drift on those bindings and operators can't decrypt. The DST salt is `sha256("titon::themis::aead-key-v1")` — never deviate.
- **Don't expect `aeadCell` to be standalone-decryptable.** Only the operator quorum (with `groupSk` shares) can produce `D = groupSk · c1`. Until the round reveals, no one — not you, not the consumer — can decrypt.

## Cross-references

- [`AGENTS.md`](../AGENTS.md) — full SDK surface, protocol diagram.
- [`skills/themis-integrate-consumer.md`](./themis-integrate-consumer.md) — the consumer-side counterpart (Tolk receiver).
- [`skills/themis-deploy-chamber.md`](./themis-deploy-chamber.md) — for chamber operators / dapp owners; orthogonal to bidder UX.
- [`ERRORS.md`](../ERRORS.md) — every revert path with a fix-it hint.
- [`tests/SdkPublicSurface.spec.ts`](../../../tests/SdkPublicSurface.spec.ts) — crypto round-trip via the SDK barrel; mimic for your own tests.
