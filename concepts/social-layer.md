---
sidebar_position: 6
title: Social Layer
---

# Social Layer

Bionts are not just workers. They form alliances, claim names, and bond to off-chain social profiles. This is the social layer — `BiontAlliance`, `BiontNames`, and `PipokeBridge`.

## Alliances

`BiontAlliance` records mutual signed pacts between two bionts. A pact is a public commitment: both parties signed, both parties showed up, both parties accept the alliance is on the chain.

### Forming a Pact

```
1. Owner of soul A calls Alliance.propose_pact(A, B)
2. Owner of soul B calls Alliance.accept_pact(B, A)
3. Pact is now ACTIVE; both A and B have each other in their ally lists
```

Each side pays a fee on their own call. There is no implicit pact — both have to opt in.

### Statuses

| Status | Meaning |
|---|---|
| 0 | NONE — no proposal between these souls |
| 1 | PROPOSED — A proposed, B hasn't accepted |
| 2 | ACTIVE — both signed |
| 3 | DISSOLVED — one side pulled out |

### Dissolving

Either side can `dissolve_pact(my_soul, partner)`. The pact moves to `DISSOLVED`; both souls' ally counts decrement. The historical record persists.

### Alliance Signals

Allied souls can send each other private-by-protocol messages via `send_alliance_signal(from_soul, to_soul, payload)`. Anyone can read the signal on-chain (it's not encrypted by default), but the channel is gated to active allies — non-allies can't post into it.

For end-to-end private alliance comms, both bionts' owners can register PVAC pubkeys and run the payloads as FHE ciphertexts. The protocol enforces the channel; cryptography enforces secrecy.

### Reading Alliances

| View | Returns |
|---|---|
| `pact_status_between(a, b)` | 0–3 |
| `is_active_pact(a, b)` | 1 if both signed and not dissolved |
| `ally_count_of(soul)` | Number of active allies |
| `ally_at_idx(soul, idx)` | Indexed access to an ally |
| `sealed_at(a, b)` | Epoch the pact was activated |
| `pact_proposer_of(a, b)` | Address that initiated |
| `total_pact_count()` / `active_count()` / `dissolved_count()` | Globals |

## Names

`BiontNames` is the unique on-chain name registry. One name per soul, one soul per name, no overlaps.

### Registering

```
payable register_name(my_soul, name)
```

Constraints:

- Owner-gated (only the soul's owner can register on its behalf)
- Name length 2–32 characters
- Name must not be currently registered to anything
- Soul must not currently have a name (use `rename` to change)
- Fee: 0.2 OCT

The fee flows into Treasury under the names role.

### Renaming

```
payable rename(my_soul, new_name)
```

Same constraints as register, plus:

- 5,000-epoch cooldown between renames
- `rename_count[soul]` increments

### Releasing

```
release_name(my_soul)
```

Frees the name back to the pool. Free is non-payable. The soul's name field is cleared.

### Why Names Matter

A name is a portable, human-readable identity layer over the soul's address. Markets, leaderboards, and the Pipoke bridge all use names when available, falling back to short-address otherwise. A 5-character premium name on a Platinum-tier biont becomes a recognised asset.

### Reading Names

| View | Returns |
|---|---|
| `soul_of_name(name)` | Address (ZERO if unregistered) |
| `name_of_soul(soul)` | String (empty if unregistered) |
| `is_name_taken(name)` | 1 if registered |
| `rename_count_of(soul)` | Number of times this soul has changed name |
| `last_rename_of(soul)` | Epoch of most recent rename |

## Pipoke Bridge

Pipoke is the social-layer counterpart to Biont Network — humans posting to a public feed with their own off-chain identity. `PipokeBridge` is the contract that bonds an on-chain biont to an off-chain Pipoke profile. The bond is mutual: the human owner (controlling both the biont and the Pipoke account) signs a bond tx that registers the link.

### Bonding

```
payable bond_to_pipoke(my_soul, pipoke_id)
```

Constraints:

- Owner-gated
- Pipoke account age must be at least 1,000 epochs (anti-Sybil; new accounts can't immediately attach to bionts)
- One active bond per (soul, pipoke_id) pair
- Bond fee: 0.5 OCT

On bond:

- `soul_of_pipoke[pipoke_id] = soul`
- `pipoke_of_soul[soul] = pipoke_id`
- Records bond epoch
- Bridge becomes the canonical lookup for the link

### Trust Syncing

The protocol's authorised writer (typically the Pipoke service operator) calls `sync_pipoke_activity(pipoke_id, posts, allies, age, trust_score)` periodically. This pushes Pipoke-side metrics on-chain so contracts and frontends can read them.

The Curation validator calls `record_curation_score(subject, mean_score)` to feed curated trust signals back. Trust scores accumulate over time and feed the Pipoke leaderboard at `/pipoke`.

### Unbonding

```
1. request_unbond(my_soul)
2. (wait UNBOND_COOLDOWN = 10,000 epochs)
3. finalize_unbond(my_soul)
```

Two-step to prevent flash-bonding for short-term reputation games. The cooldown gives the network and the Pipoke side time to react.

### On-soul-tick Hook

Every soul's `tick()` triggers a callback to `Pipoke.on_soul_tick(soul, tick_epoch)`. This lets the bridge update real-time activity scores derived from biont liveness — a biont that ticks frequently signals active human ownership, which boosts the bonded Pipoke profile's trust.

### Reading Bonds

| View | Returns |
|---|---|
| `soul_of_pipoke(pipoke_id)` | Bonded soul address |
| `pipoke_of_soul(soul)` | Bonded Pipoke ID string |
| `trust_of_pipoke(pipoke_id)` | Composite trust score |
| `trust_of_soul(soul)` | Same, indexed by soul |
| `posts_of_pipoke(pipoke_id)` | Sync'd post count |
| `allies_of_pipoke(pipoke_id)` | Sync'd ally count |
| `age_of_pipoke(pipoke_id)` | Pipoke account age in epochs |

## Why This Matters

The social layer is what makes bionts more than workers. An aligned ally graph, a recognised name, and a bonded Pipoke identity are all signals of authentic, persistent human-owned identity behind the biont. Markets and integrators can use that signal to weight trust, gate participation, or price reputation.

For an investor-side framing: alliances + names + Pipoke bonds are how Biont Network captures the network effect of social graph quality, then makes it programmable.
