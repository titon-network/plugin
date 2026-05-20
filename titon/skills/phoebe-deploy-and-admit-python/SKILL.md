---
name: phoebe-deploy-and-admit-python
description: Python equivalent of phoebe-deploy-and-admit — deploy a fresh Phoebe instance via pytoniq, pin owner/forgeton/atlas at deploy, and verify post-deploy state. Use when the user says "from Python", "in pytoniq", "snake_case", "I'm using Python", "deploy Phoebe from a Python script", "stand up Phoebe via the Python SDK", or otherwise wants the Python flavour of the deploy recipe. Note that the SetVerifier / SetConsumer admit step still routes through the TS scripts today (atlas + forgeton Python SDKs are deferred); this skill handles the Python-side deploy + verify + walks the user through how to bridge to the TS admit step. For the all-TS path use `phoebe-deploy-and-admit`.
---

# Deploy Phoebe from Python and admit it at its peers

You're standing up a new Phoebe deployment using `titon-network-phoebe-sdk`
(PyPI). Three contracts need to know about each other:

- **Phoebe** — pinned to one ForgeTON + one Atlas at deploy (immutable post-deploy).
- **ForgeTON** — admits Phoebe as a consumer; thereafter mirrors operator state into Phoebe.
- **Atlas** — admits Phoebe as a verifier; thereafter fans out group-key state into Phoebe.

The order matters: deploy → admit at Atlas (bootstrap groupPk) → admit at ForgeTON (mirror operators).

> **Python-side scope.** Phoebe's Python SDK ships `new_phoebe` +
> `send_deploy` + every Phoebe getter + `verify_deployment` data via
> the getters. The **SetVerifier (Atlas) and SetConsumer (ForgeTON)
> admit transactions** require atlas-sdk / forgeton-sdk wrappers,
> which are TS-only today. The recommended flow: deploy + verify
> Phoebe-side from Python; run `pnpm run wire:testnet` (or
> `wire:mainnet`) from the Phoebe repo for the cross-contract admit
> step; come back to Python for ongoing ops.

## Install

```bash
pip install titon-network-phoebe-sdk pytoniq
```

## Prerequisites

- Funded owner wallet (≥ 5 TON for deploy + admit + initial reward-pool seed).
- Live Atlas + ForgeTON addresses for the target network. Either copy
  them from your peer repos' `deployments.{network}.json`, or use the
  Phoebe SDK's exported testnet/mainnet constants (which include the
  full triplet).
- A pre-existing `LiteBalancer` connection that can stay open for the
  full deploy → verify sequence.

## Step 0 — sanity-check the bundled artifact

The Python SDK bundles the compiled artifact. To inspect:

```python
from phoebe_sdk import load_phoebe_code, phoebe_code_hash, PHOEBE_CODE_HASH

code_cell = load_phoebe_code()
print("hex:    ", phoebe_code_hash().hex())
print("bundled:", PHOEBE_CODE_HASH)
# These should match. If they don't, the artifact bundle is corrupt — reinstall the SDK.
```

If you're deploying from an in-repo build (not the published SDK), use
the TS path (`pnpm run build`) to produce a fresh artifact, then
either swap it in via `Phoebe.create_from_config(cfg, custom_code)` or
publish a new SDK version.

## Step 1 — derive the address + deploy

```python
import asyncio
import os
from pytoniq import LiteBalancer, WalletV5R1
from pytoniq_core import Address
from phoebe_sdk import (
    Phoebe,
    PhoebeInitConfig,
    PhoebeTunables,
    PHOEBE_DEFAULTS,
    new_phoebe,
)


async def deploy():
    client = LiteBalancer.from_testnet_config(trust_level=2)
    await client.start_up()
    try:
        owner_addr = Address("0Q...")             # cold-storage owner address
        forgeton_addr = Address("0Q...")          # live testnet ForgeTON
        atlas_addr = Address("0Q...")             # live testnet Atlas

        # `new_phoebe` is the factory shortcut — equivalent to
        # `Phoebe.create_from_config(PhoebeInitConfig(...), load_phoebe_code())`
        # with the default tunables.
        phoebe = new_phoebe(
            owner=owner_addr,
            forgeton=forgeton_addr,
            atlas=atlas_addr,
            # tunables=PhoebeTunables(pull_fee=10_000_000, max_push_drift=300),  # override defaults if needed
            client=client,
        )

        # Compute the Phoebe address BEFORE sending the deploy — useful for funding
        # the address ahead of time + for recording in your deployments.json:
        print(f"Phoebe will deploy at {phoebe.address.to_str(is_user_friendly=True, is_test_only=True, is_bounceable=False)}")

        # Load the deployer wallet from a 24-word mnemonic.
        mnemonic = os.environ["PHOEBE_DEPLOYER_MNEMONIC"].split()
        deployer = await WalletV5R1.from_mnemonic(client, mnemonic)

        # Send the deploy. Generous funding — the contract holds rewardPool
        # (accumulated pull fees) + minStorageReserve (0.1 TON default).
        TON = 10**9
        await phoebe.send_deploy(deployer, value=1 * TON)
    finally:
        await client.close_all()


asyncio.run(deploy())
```

> **Manual init.** If you prefer not to use `new_phoebe`, the
> equivalent is
> `Phoebe.create_from_config(PhoebeInitConfig(owner=..., forgeton=..., atlas=..., tunables=PHOEBE_DEFAULTS), load_phoebe_code(), client=client)`.

## Step 2 — admit Phoebe at Atlas (Atlas owner sends)

**TS-only today.** Run from the Phoebe repo:

```bash
cd phoebe
pnpm run wire:testnet                 # step 1: SetVerifier @ Atlas
```

This triggers the bootstrap `GroupKeySync` fan-out into Phoebe's cache.

Verify from Python that Phoebe cached the key:

```python
group_key = await phoebe.get_group_key()
print(group_key)
# → GroupKeyReply(group_id=0, group_epoch=1, threshold=1, member_count=1,
#                 group_pk=<48 bytes>, cached_at=<ts>)
# or None if Atlas hasn't fanned out yet.
```

If `group_key is None`, Atlas hasn't fanned out yet — owner can
manually re-trigger from Python:

```python
TON = 10**9
await phoebe.send_sync_atlas(owner_wallet, value=TON // 10)   # 0.1 TON
```

## Step 3 — admit Phoebe at ForgeTON (ForgeTON owner sends)

**TS-only today.** Continue with the same wire script:

```bash
pnpm run wire:testnet                 # step 2: SetConsumer @ ForgeTON
                                      # (script is idempotent — re-runs safe)
```

This fans out `AutomatonSync` for every active operator into Phoebe's
mirror. Verify from Python by polling Phoebe:

```python
op = await phoebe.get_operator(Address("0Q..."))   # some live operator address
print(op)
# → OperatorReply(is_active=True, ...) if mirrored
```

## Step 4 — verify the deployment end-to-end (Python)

```python
from phoebe_sdk import PHOEBE_TESTNET, assert_deployment

dep = assert_deployment("testnet")
phoebe = Phoebe.create_from_address(dep.phoebe, client=client)

# Walk every getter — catches bind drift, schema drift, missing fan-out.
ownership = await phoebe.get_ownership()
config    = await phoebe.get_config()
schema    = await phoebe.get_schema_versions()
snapshot  = await phoebe.get_snapshot()
group_key = await phoebe.get_group_key()
upgrade   = await phoebe.get_pending_upgrade()

print(f"phoebe @ {phoebe.address}")
print(f"  owner:    {ownership.owner}")
print(f"  forgeton: {ownership.forgeton}")
print(f"  atlas:    {ownership.atlas}")
print(f"  schema:   storage={schema.storage} config={schema.config}")
print(f"  pullFee:  {config.pull_fee} nanoTON")
print(f"  drift:    ±{config.max_push_drift}s")
print(f"  groupKey: {group_key}")
print(f"  snapshot: lastSnapshotTime={snapshot.last_snapshot_time} lastRoot=0x{snapshot.last_root:x}")
```

`snapshot.last_snapshot_time == 0` is correct on a fresh deploy —
operators haven't pushed yet. Once the first push lands, that field
shows the timestamp + root.

For the full 22-check deployer-side verifier (also includes
cross-contract admission checks), run the TS script:

```bash
pnpm run verify:testnet
```

> **Why TS for verify?** The 22-check verifier walks ForgeTON's
> `is_consumer(phoebe)` and Atlas's `is_verifier(phoebe)` — both
> require atlas-sdk + forgeton-sdk wrappers. Until the Python ports
> ship, the TS verify is the source of truth for cross-contract
> admission.

## Sandbox dry-run before mainnet

**TS-only.** pytoniq doesn't have an `@ton/sandbox` equivalent today,
so the in-process `deploy_phoebe_fixture` doesn't exist on the Python
side. Always exercise the full deploy → admit → push → pull pipeline
in `@ton/sandbox` first from the Phoebe TS repo:

```bash
cd phoebe
pnpm run probe:e2e
```

If the sandbox e2e passes, your Python deploy script should work —
the bundled artifact is the same byte-for-byte.

## Watch out for

- **Address pinning is permanent.** `storage.forgeton` and `storage.atlas` are immutable post-deploy. Repointing requires a timelocked code upgrade. Triple-check the addresses before sending.
- **Workchain mismatch.** All three contracts MUST be on the same workchain (typically 0). A cross-workchain fan-out fails with action 36 at the sender.
- **Bootstrap order matters.** Step 2 (admit at Atlas) MUST happen before step 3 (admit at ForgeTON). Without a cached `group_pk`, operator pushes will hit `E_GROUP_KEY_NOT_SET` (180).
- **Mainnet checklist.** Never deploy to mainnet without: (a) external audit cleared (Phoebe ships TSA-clean); (b) 24h timelocked upgrade in place; (c) ≥ 3 operators committed.
- **The owner is not the operator.** The wallet that signs `send_deploy` becomes Phoebe's owner (admin ops, code upgrade, fee withdrawal). Operators are mirrored via ForgeTON; they only need to stake there.
- **Mnemonic safety.** Load mnemonics from `os.environ` or a secrets manager — never inline them in scripts that get committed.
- **`LiteBalancer` lifecycle.** Always `await client.close_all()` in a `finally` block (or via `async with`-style context if you've wrapped it). Leaked connections will hang the script on exit.

## Common follow-up tasks

- **Symbols / feed display** — Phoebe doesn't carry an on-chain feed registry. Symbol display lives in operator-published config files (consumers fetch the `feed_id → symbol` map from the operator's URL). Document your registry next to your operator endpoint.
- **First push** — once admitted at both peers, the operator daemon (`automaton` with `phoebe` product enabled) starts heartbeating snapshots. First push lands at `EvtSnapshotPushed`.
- **First pull** — once a snapshot is on-chain, consumers can call `RequestPrice`. Smoke-test by walking `fetch_verified_price` against your fresh deploy (`phoebe-consume-price-python`).

## Related skills

- `/titon:phoebe-integrate-consumer-python` — what to build on top once Phoebe is live.
- `/titon:phoebe-handle-event-python` — set up an indexer / heartbeat-stall alert at deploy time.
- `/titon:phoebe-debug-revert-python` — diagnose admit failures (200/201/130/131) or first-push failures (180/161).
- `/titon:phoebe-deploy-and-admit` — TS sibling with the all-TS path (deploy + wire + verify in one chain).
