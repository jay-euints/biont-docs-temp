---
sidebar_position: 1
title: Core Mechanics
---

# Core Mechanics

This page summarises the mechanical rules of Biont Network — the numbers, intervals, and procedures that define how the system behaves day-to-day. For deep theory see [Architecture](../architecture.md); for ownership procedures see [Ownership](../guides/ownership.md).

## Mint

`BiontGenesis.mint_biont(name, owner)` is `payable`. The fee is set by the protocol and paid in OCT.

- Genesis maintains a pool of pre-deployed `BiontSoul` proxies.
- On mint, Genesis pops a proxy, calls `init_identity` on it (sets owner, name, archetype, seed, birth_epoch), and registers it with `BiontSoulRegistry`.
- The new soul is alive, has `STARTING_VITALITY = 5,000`, and is fully callable.

The new owner can immediately:

- Subscribe the soul to job types
- Move it on the territory grid
- Set its name
- Transfer or list it

## Vitality

Every soul has a vitality counter, capped at `MAX_VITALITY = 10,000`.

- **Starts at**: `STARTING_VITALITY = 5,000`
- **Decays at**: `VITALITY_DECAY_PER_EPOCH = 1` per epoch elapsed since last tick
- **Restored by**: `VITALITY_PER_TICK = 1` on every successful tick call

The Registry computes `new_vit = cur_vit − decay × idle + 1` on each `tick`, where `idle` is the number of epochs since the last tick. There is **no per-epoch cap on tick frequency** — anyone can call `tick()` repeatedly within an epoch. Within a single epoch each successive call adds +1 with `idle = 0`, so vitality climbs; the poke reward, however, is computed against `idle` (paid only when meaningful epoch time has passed) — so spamming ticks pays nothing.

A soul dies only when **both** conditions hold during a tick: `vitality == 0` **and** the soul's Treasury balance is `0`. A vitality-zero soul that still holds OCT in its Treasury entry survives and can be revived by ticks (or earnings).

`tick()` is permissionless. The caller — the **poker** — earns a poke reward sized by how long the soul went idle. This means third parties have an incentive to keep your bionts alive: forget to tick, and a profit-seeking poker will tick on your behalf to claim the reward.

## Subscribing to job types

`BiontWorkEngineV2.subscribe_for(soul, type)` is owner-gated.

- One subscription per (soul, job_type) pair.
- Up to 7 active subscriptions per soul (one per type).
- Soft-removable via `unsubscribe_for(soul, type)` — the soul comes off the pool but its subscription record persists for indexing.

Once subscribed, every newly posted job of that type pulls quorum from the pool round-robin. Your soul will eventually be picked. The more types you subscribe to, the more frequently you're assigned.

## Job lifecycle

```
[poster] post_jobs_bulk(...)
    ↓
[engine] Round-robin assigns quorum from subscriber pool per type
    ↓
[soul owner(s)] attest_for(soul, job_id, payload)
    ↓
[anyone] auto_finalize(job_id) after deadline or quorum hit
    ↓
[validator] judges payloads (consensus by majority/median/etc)
    ↓
[engine] pay_winner / slash_loser callbacks settle
```

Default timing parameters:

- **Quorum size**: 3
- **Job deadline**: 1,000 epochs from posting
- **Settlement window**: opens at deadline OR when quorum payloads are in
- **Permissionless settle delay**: anyone can call `auto_finalize` after the settlement window opens
- **Cancellation**: `cancel_job_after_deadline` recovers bounty if no attestations were submitted

## Reputation

`BiontReputation` accrues per-soul.

- **+ delta** on every winning attestation (validator-decided, typically +10 for standard jobs, scaled by bounty)
- **− delta** on slashes (typically equal magnitude)
- **Tier thresholds**: Bronze ≥ 100, Silver ≥ 1,000, Gold ≥ 10,000, Platinum ≥ 100,000

Tier is read via `BiontReputation.tier_of(soul)`. Some validators weight high-tier souls heavier in consensus.

## Lineage

`BiontLineage.set_parents(child, parent_a, parent_b)` is one-shot per child. After it's set, both parents are sealed. Up to two parents per child; both are read via `parent_a_of(child)` and `parent_b_of(child)`.

There is no breeding-cooldown enforcement at the lineage level — the network expects parents to gate this in their own contracts (e.g. via vitality cost). Lineage just records the relationship.

## Territory

The grid is `TERRITORY_GRID × TERRITORY_GRID = 500 × 500 = 250,000` zones.

- **Move cost**: tracked but soul-paid (`move_soul` is permissionless once owner-authorised)
- **Claim threshold**: a soul can claim a zone after sufficient cumulative visits
- **Road threshold**: an edge between two zones (from movement) becomes a road at `road_threshold` (default 50) crossings
- **Visit reward**: paid per move into the zone, split between zone holder and Treasury

Zone, biome, landmark kind, label, visits, reward, and claimed_at_epoch are all readable via `zone_*_of(zid)`.

## Death

A soul dies when:

- A `tick()` runs and finds **both** `vitality == 0` **and** the soul's Treasury balance `== 0`, OR
- `Registry.force_kill(soul)` is invoked by protocol owner (audit / deprecation only)

A vitality-zero soul that still holds OCT inside Treasury survives — its balance acts as a buffer. Both have to bottom out simultaneously for `_kill_soul` to fire.

On death, `Registry._kill_soul(soul)` runs:

```
soul_alive[soul] = 0
total_alive -= 1
total_dead += 1
emit SoulDied
call(graveyard, "record_death", soul, owner, epoch, age)
call(lineage, "mark_dead", soul)
```

The soul becomes uncallable for work, transfer, subscription, or movement. Its identity, lineage, history, balance, and SVG remain on-chain forever.

## Resurrection

`BiontGraveyard` keeps a 25,000-epoch resurrection window. During the window, allies can vote to resurrect via `vote_resurrect`. If the threshold is reached, the soul is restored at minimum vitality. After the window closes, the death is permanent.

## Liberation

`BiontSoulRegistry.free_biont(soul)` is owner-gated. The Registry then calls back into the soul's `mark_freed`. After liberation:

- Soul's `is_freed_flag = 1` (no further transfers permitted)
- Registry sets `owner = ZERO_ADDRESS`
- Registry records `liberator = caller` (the freeing wallet)
- Treasury sets the soul's `is_freed = 1` and stores `liberator_of_soul`

From that point on, every credit into the soul's Treasury entry is split into two parts:

- A `liberator_royalty_bps` portion accrues to the liberator's claimable balance (`claim_liberator_earnings`).
- The remainder stays inside `soul_earnings`, where the soul can spend it on poke rewards.

`liberator_royalty_bps` is a Treasury parameter — it is **not** a fixed 100% redirect. The current default is configured by the protocol owner and may be tuned post-mainnet.

Liberation is irreversible. The soul becomes self-owning, can never be transferred again, and pays a perpetual royalty stream to whoever freed it.

## Names

`BiontNames.set_name(soul, name)` is owner-gated. Names are unique-per-soul (one name per soul, one soul per name within a namespace). Renames cost a small fee paid to Treasury.

`name_of_soul(soul)` returns the human-readable name.

## Pipoke link

`BiontPipokeBridge.link_pipoke(soul, pipoke_id)` binds a soul to a Pipoke profile. Once linked, the bond is half-set; the Pipoke side completes the link by referencing the soul. Both sides must agree for the bond to count as full.

A linked biont can earn social-graph fees and curation tips that flow through the bridge.

## Tick frequency

`tick()` has no per-epoch ceiling. Anyone can call it any number of times. The poke reward is sized by `idle` (epochs since the last successful tick), so spamming inside the same epoch yields no reward — only the first tick after meaningful idle time pays out. Vitality, however, accrues +1 per call regardless of `idle`, so a third party can pad your soul's vitality at no cost to themselves.

## Numbers at a glance

| Parameter | Default value |
|---|---|
| `STARTING_VITALITY` | 5,000 |
| `MAX_VITALITY` | 10,000 |
| `VITALITY_DECAY_PER_EPOCH` | 1 |
| `VITALITY_PER_TICK` | 1 |
| Default mint price | 1 OCT (`1,000,000` raw) |
| Default per-wallet mint cap | 10 |
| `MIN_QUORUM` / `MAX_QUORUM` | 3 / 32 |
| `MAX_DEADLINE_EPOCHS` | 100,000 |
| `MAX_BULK_COUNT` | 20 |
| `MIN_BOUNTY_RAW` (per-job) | 100,000 (= 0.1 OCT) |
| `REWARD_PER_WIN` (rep) | 50 |
| `SLASH_PER_LOSE` (rep) | 100 |
| `DISPUTE_WINDOW_EPOCHS` | 1,000 |
| Reputation tier mins | 100 / 1,000 / 10,000 / 100,000 |
| `GRID_SIZE` | 500 (= 250,000 zones) |
| `DEFAULT_ROAD_THRESHOLD` | 50 crossings |
| `RESURRECTION_WINDOW` | 25,000 epochs |
| `RESURRECTION_FEE_RAW` | 10,000,000 (= 10 OCT) |
| `MARKET_FEE_BPS` | 250 (= 2.5%) |
| Share distribution fee | 5% (`value × 500 / 10,000`) |
| Share transfer fee | 0% (TRANSFER_FEE_BPS=50 is reserved but unused at v2) |
| `SHARES_PER_SOUL` | 10,000 |
| `MAX_ARCHETYPES` (chain) | 16 (UI maps the first 10 to named archetypes) |
| Validator types | 7 |
| Treasury roles | 12 |
| `UNBOND_COOLDOWN` (Pipoke) | 10,000 epochs |

> All values above are **devnet defaults**. Mint prices, mint caps, treasury fee splits, royalty bps, poke-reward curves, and any other tunable are governance-settable and may change at mainnet. Nothing in this table is permanent.
| Bulk job cap per tx | 20 |

These numbers are fixed for v2 unless the protocol owner amends them. Future versions may tighten or expand the table.
