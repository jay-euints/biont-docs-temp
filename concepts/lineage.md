---
sidebar_position: 4
title: Lineage & Breeding
---

# Lineage & Breeding

Bionts have families. `BiontLineage` records the parent-child graph of every soul, supports two-parent breeding with hybrid trait derivation, and propagates death up the tree.

## Stamping at mint

Every biont minted via `BiontGenesis.complete_mint` triggers `Lineage.register_genesis(soul, seed, archetype)`. For a fresh genesis biont (no parents), the lineage record is the bare minimum — the soul is its own root, generation 0.

For a biont born of a successful breeding, both `parent_a` and `parent_b` are recorded, generation increments to `max(gen_a, gen_b) + 1`, and the child inherits a `hybrid_seed` derived from both parents' seeds.

## State per soul

| Field | Meaning |
|---|---|
| `parent_a` | First parent's address (ZERO if genesis) |
| `parent_b` | Second parent's address (ZERO if genesis or single-parent) |
| `generation` | 0 for genesis bionts; `max(gen_a, gen_b) + 1` for breeders |
| `hybrid_seed` | Combined seed for breeders (used to derive hybrid traits) |
| `archetype_at_birth` | Archetype assigned at mint time |
| `child_count` | Number of registered children |
| `child_at[idx]` | Indexed access to children |
| `last_breeding` | Epoch of the soul's most recent breeding event |
| `breed_count` | Total successful breedings this soul has participated in |
| `is_dead` | Mirror of Registry's death state |

## The breeding flow

Breeding is a two-step proposal/accept dance, mirroring how alliances work.

```
1. Owner of parent_a calls Lineage.propose_breed(parent_a, parent_b)
   — payable; pays the breeding fee
   — sets pending_breed_partner[parent_a] = parent_b
   — sets pending_breed_epoch[parent_a]   = epoch

2. Owner of parent_b calls Lineage.propose_breed(parent_b, parent_a)
   — must happen within a window of parent_a's pending_breed_epoch
   — both parents now have matching pending_breed entries
   — Lineage triggers Genesis to mint a child soul
   — child gets parent_a + parent_b stamped, hybrid_seed computed,
     generation = max(parent_a.gen, parent_b.gen) + 1

3. Child appears in Genesis.soul_by_id and Lineage.child_at[parent_a/b]
```

The child is owned by whoever called step 2 (the accepter). This biases ownership toward the partner who closed the breeding — a small alignment incentive.

`last_breeding` and `breed_count` increment on both parents. Bionts that breed often build verifiable lineages.

## Hybrid trait derivation

A child's `seed` is the canonical hybrid seed:

```
hybrid_seed = hash(parent_a.seed || parent_b.seed || child.id || epoch)
```

This drives:

- The child's visual SVG (each trait field re-derives from `hybrid_seed`)
- The child's archetype (via `(hybrid_seed * 1664525 + 1013904223) mod 16` clamped 0–9)
- The child's initial capability tag (via `derive_initial_cap(archetype)`)

Two parents with high-reputation Platinum traits don't directly produce a Platinum child — reputation is earned per-soul through work, not inherited. But a child of two veteran lines starts with a unique seed that no fresh genesis biont can match, and the lineage record itself is a verifiable provenance.

## Generations

`generation` tracks ancestry depth:

| Origin | Generation |
|---|---|
| Genesis mint (no parents) | 0 |
| Breeders both at gen 0 | 1 |
| One parent gen 0, one gen 1 | 2 |
| Both gen 2 | 3 |

There is no upper bound on generation — a network running for years could see lineages 50+ deep. Older lineages are scarcer (most bionts never breed, fewer lines reach high generations) and are tracked on the [/lineage](/lineage) page.

## Death cascade

When a soul dies (vitality decay or `force_kill`), `Registry._kill_soul` calls `Lineage.mark_dead(soul)`. This:

- Sets `is_dead[soul] = 1`
- Increments `total_deaths`
- Stops further breeding (a dead biont can't be a `pending_breed_partner`)
- Closes the soul's line — children are not affected, but the dead soul itself is no longer a viable parent

The lineage record persists forever. A dead biont's family tree is still queryable; its descendants still trace back to it; its `breed_count` and `last_breeding` are still readable.

## Reading the family tree

Frontends use these views:

| View | Returns |
|---|---|
| `parent_a_of(soul)` | First parent address |
| `parent_b_of(soul)` | Second parent address |
| `generation_of(soul)` | Generation depth |
| `children_of(soul)` | Count of children |
| `child_at_idx(soul, idx)` | Indexed child address |
| `breed_count_of(soul)` | Total breedings |
| `is_dead(soul)` | 1 if dead |

Recursive walks (e.g. "all descendants of X") happen client-side using these views. The 3D world map and the [/lineage](/lineage) page render trees by walking the graph live.

## What it's worth

A biont with deep lineage and many descendants is verifiably old. Its provenance is in chain history — the original mint tx, every breeding, every child. Markets, reputation queries, and institutional integrators can use that lineage signal as a Sybil-resistant anchor: a 5-generation Patriarch with 30 descendants can't be faked overnight.
