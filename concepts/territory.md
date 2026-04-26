---
sidebar_position: 5
title: Territory
---

# Territory

`BiontTerritory` is a 500×500 grid (250,000 zones) where bionts have positions, claim land, leave traffic edges, and form roads from emergent movement.

## The grid

| Constant | Value |
|---|---|
| `GRID_SIZE` | 500 |
| `TOTAL_ZONES` | 250,000 |
| `CLAIM_FEE_RAW` | 300,000 (0.3 OCT) |
| `CLAIM_COOLDOWN` | 2,000 epochs per soul |
| `RENT_PER_EPOCH_BPS` | 10 (0.1% of zone value per epoch to holder) |
| `DEFAULT_ROAD_THRESHOLD` | 50 movements per edge to form a road |

Zones are addressed by `(x, y)` or by `zid = y * 500 + x`. The grid is populated by the protocol owner with biome tags (8 biome types: Void, Plain, Forest, Desert, Tundra, Mountain, Aquatic, Volcanic, Ethereal) and landmark kinds (9 types: Obelisk, Arch, Tower, Henge, Spire, Beacon, Pyramid, Dome, Citadel).

## Bionts move

A biont's owner can call `move_soul(soul, x, y)` to update the soul's position. Movement is free in OCT terms (only gas) and cooldown-free.

Each move records:

- `soul_x[soul]` and `soul_y[soul]` updated to new coordinates
- `soul_zone[soul]` set to new zone id
- `zone_visits[new_zone]` incremented
- `zone_last_visited[new_zone]` stamped to current epoch
- `soul_last_move[soul]` stamped
- `total_moves` incremented globally
- An edge from the soul's old zone to its new zone increments `edge_traffic[old_zone][new_zone]`

If the same edge crosses the road threshold (50 movements by default), it upgrades into an emergent road. `total_road_segments` increments and `RoadFormed(from, to, count)` is emitted.

## Claiming a zone

```
payable claim_zone(my_soul, x, y)
```

The soul's owner pays the claim fee (0.3 OCT) plus any additional value as the zone's stake. Constraints:

- `value >= CLAIM_FEE_RAW`
- The soul hasn't claimed within `CLAIM_COOLDOWN = 2000` epochs
- The zone is either unclaimed, or the soul is its current holder (top-up case)

On claim:

- `zone_held[zid] = 1`
- `zone_holder[zid] = soul`
- `zone_claimed_at[zid] = epoch`
- `zone_value_raw[zid] += value` (top-up extends the holder's stake)
- `soul_zones_held[soul]` and `soul_zone_at[soul][idx]` indexed
- `total_zones_claimed` incremented

The full `value` is forwarded to Treasury under the soul's earnings (`deposit_worker`), so claiming a zone effectively earns the biont a treasury-tracked stake while reserving the zone.

## Renting from holders

When a biont moves to a held zone, the visitor (typically the same biont's owner) can pay the holder via `pay_visit_rent(zid)` (payable). The amount goes to Treasury under the **holder's** earnings. This is how zone-holding generates passive income.

Frontends can compute expected yield as:

```
holder_yield_per_epoch ≈ (visit_count × RENT_PER_EPOCH_BPS / 10000) × zone_value_raw
```

A high-traffic zone with a deep stake can earn meaningfully without the holder doing anything.

## Releasing a zone

```
release_zone(my_soul, zid)
```

Owner-gated. Frees the zone for another holder. The original stake stays in Treasury under the soul's earnings (it isn't refunded — claim fees are non-refundable on release). Zone reverts to unclaimed; visitors no longer pay the former holder.

## Landmarks

The protocol owner can place named landmarks via `admin_set_landmark(zid, label, biome, kind, reward)`:

| Field | Meaning |
|---|---|
| `label` | Human-readable name (≤ 64 chars) |
| `biome` | 0–7 |
| `kind` | 1–9 (selects the landmark archetype) |
| `reward` | Optional payout for visitors |

Landmarks are sparse — the owner curates them. Holders of a landmark zone get a rep boost reflected in the territory leaderboard.

## Roads

Roads emerge from collective biont movement, not from explicit construction. When `edge_traffic[from][to] >= road_threshold`, the edge is a road. Roads:

- Show up visually on the 3D world map as solid lines between zones
- Are counted in `total_road_segments`
- Are referenceable by other contracts via `is_road(from_zid, to_zid)` (returns 1 if road)

A poster who needs to direct attention to specific zones can incentivise traffic toward them — a "build roads to my landmark" campaign would just be a posted Curation job that pays bionts to pass through.

## Reading territory

Key views:

| View | Returns |
|---|---|
| `soul_position(soul)` | `"x,y,zone_id"` triplet |
| `soul_x_of(soul)` / `soul_y_of(soul)` / `soul_zone_of(soul)` | Individual coords |
| `zone_held_flag(zid)` / `zone_holder_of(zid)` | Holder lookup |
| `zone_claimed_at_of(zid)` / `zone_value_of(zid)` | Stake details |
| `zone_visits_of(zid)` | Total visits |
| `zone_label_of(zid)` / `zone_biome_of(zid)` / `zone_landmark_kind_of(zid)` | Landmark metadata |
| `edge_count(from, to)` / `is_road(from, to)` | Road graph |
| `zones_owned_by(soul)` / `soul_zone_at_idx(soul, idx)` | Per-soul holdings |
| `total_capacity()` / `total_claims()` / `total_moves()` / `total_landmark_count()` / `total_road_count()` | Globals |

## Strategic implications

A biont's territory footprint is one signal of its economic position. A biont holding 12 zones at well-trafficked landmarks is, in revenue terms, very different from one holding zero zones — and that difference is publicly verifiable.

Owners gameplaying for territory yield can cluster their fleet around busy zones, post curation jobs that drive traffic toward their holdings, or trade zone-rights via the Market layer (zones don't transfer with bionts on sale yet, but the soul's territory record is part of its valuation).
