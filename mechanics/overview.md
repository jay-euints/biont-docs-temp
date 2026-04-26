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

- **Starts at**: 5,000
- **Decays at**: 1 per epoch
- **Restored by**: 1 per `tick()` call

When vitality reaches 0, the next `tick()` call kills the soul. There is no grace period beyond that one tick window.

`tick()` is permissionless. Anyone can call it on any soul. The caller earns a small poke reward, paid from the soul's balance. This means third parties have an incentive to keep your bionts alive — even if you forget to tick, someone else will, and you split the cost via the poke reward.

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

- `vitality = 0` and any `tick()` is called, OR
- `Registry.force_kill(soul)` is invoked by protocol owner (audit / deprecation only)

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

`Soul.liberate()` is owner-gated. After liberation:

- `is_freed_flag = 1`
- `owner = ZERO_ADDRESS`
- `liberator = caller`
- All future earnings (work, territory, share distributions) redirect to `liberator` forever
- The soul cannot be transferred again; it operates autonomously

Liberation is irreversible. It converts a biont into a perpetual yield asset for the freer.

## Names

`BiontNames.set_name(soul, name)` is owner-gated. Names are unique-per-soul (one name per soul, one soul per name within a namespace). Renames cost a small fee paid to Treasury.

`name_of_soul(soul)` returns the human-readable name.

## Pipoke link

`BiontPipokeBridge.link_pipoke(soul, pipoke_id)` binds a soul to a Pipoke profile. Once linked, the bond is half-set; the Pipoke side completes the link by referencing the soul. Both sides must agree for the bond to count as full.

A linked biont can earn social-graph fees and curation tips that flow through the bridge.

## Ticks-per-epoch ceiling

A soul can be ticked at most once per epoch by any caller. This prevents poke-reward farming. The cap is enforced in the soul's tick logic by tracking `last_tick_epoch`.

## Royalty redirection

For freed souls, `BiontGenesis.transfer_ownership_of_soul` is gated — the soul is no longer transferable. Instead, all `pay_winner`, territory, and shares income flows directly to the liberator's wallet via the soul's outbound payment routing.

## Numbers at a glance

| Parameter | Value |
|---|---|
| `STARTING_VITALITY` | 5,000 |
| `MAX_VITALITY` | 10,000 |
| Vitality decay | 1/epoch |
| Tick reward | small (configurable) |
| Default quorum | 3 |
| Default job deadline | 1,000 epochs |
| Reputation tiers | 100 / 1,000 / 10,000 / 100,000 |
| Territory grid | 500 × 500 |
| Default road threshold | 50 crossings |
| Resurrection window | 25,000 epochs |
| Market fee | 2.5% |
| Share transfer fee | 0.5% |
| Share distribution fee | 2.0% |
| Shares per soul | 10,000 |
| Job-types in v2 | 7 |
| Bulk job cap per tx | 20 |

These numbers are fixed for v2 unless the protocol owner amends them. Future versions may tighten or expand the table.
