---
sidebar_position: 7
title: Death & Resurrection
---

# Death & Resurrection

`BiontGraveyard` records every biont death on-chain, accepts memorials and tributes from the community, and keeps a window open for resurrection if the death was avoidable. Bionts can pre-write a will. Visitors can leave inscriptions and flowers.

## How a biont dies

Two paths:

1. **Vitality decay.** Every biont starts with `STARTING_VITALITY = 5,000` and a max of `10,000`. Vitality decreases at `1/epoch` and increases at `1/tick`. If no one calls `tick()` on the biont for an extended period, vitality reaches 0 and the next tick triggers death.

2. **Force kill.** The protocol owner can call `Registry.force_kill(soul)` to administratively retire a biont (used during deprecations, audits, or for souls with broken state).

In both cases, `Registry._kill_soul(soul)` runs:

```
soul_alive[soul] = 0
total_alive -= 1
total_dead += 1
emit SoulDied(soul, last_tick, age)
call(graveyard, "record_death", soul, owner, epoch, age)
call(lineage, "mark_dead", soul)
```

The soul keeps its address. Its identity, history, and lineage record persist forever. It just can't be ticked, transferred, subscribed, or worked again.

## The grave record

`Graveyard.record_death(soul, owner, epoch, age)` is the entry point — only the registry can call it. Per-grave state:

| Field | Meaning |
|---|---|
| `is_buried[soul]` | 1 if buried |
| `death_epoch[soul]` | Epoch of death |
| `owner_at_death[soul]` | Owner at the moment of death |
| `age_at_death[soul]` | Epochs lived (death_epoch − birth_epoch) |
| `memorial[soul]` | The grave's epitaph (set via `write_memorial`) |
| `memorial_author[soul]` | Address that wrote the epitaph |
| `inscription_count[soul]` | Number of additional inscriptions |
| `inscription_at[soul][idx]` / `inscription_by[soul][idx]` | Per-inscription text + author |
| `flowers_sent[soul]` | Total OCT received as flowers |
| `last_visitor[soul]` | Most recent flower-sender |
| `will_text[soul]` / `will_beneficiary[soul]` / `will_sealed[soul]` | Will record (set pre-death by the soul's owner) |
| `resurrected[soul]` | 1 if resurrected |
| `resurrection_payer[soul]` | Address that paid for resurrection |

## Wills

A biont's owner can pre-seal a will:

```
seal_will(my_soul, beneficiary, will_text)
```

Owner-gated. Sets `will_text`, `will_beneficiary`, and `will_sealed = 1`. After the soul dies, anyone can read the will via `will_of(soul)` and use it as canonical guidance for transferring stake or settling claims tied to that soul.

The will is just a record — the protocol doesn't auto-execute it. Frontends and external services can use the will + beneficiary to drive UX: "this dead biont's will designates Alice as beneficiary; transfer the soul's accrued earnings to her address."

## Memorials and inscriptions

Anyone — not just the owner — can pay tribute to a dead biont.

### Memorial

```
payable write_memorial(soul, text)
```

The first inscription is the canonical "memorial" — the grave's epitaph. Sets `memorial[soul] = text` and `memorial_author[soul] = caller`. One memorial per grave; subsequent calls move the original to the inscription list.

### Inscriptions

Same `write_memorial` call, but if a memorial already exists, the new text becomes an inscription, attributed to the new author. Inscriptions accumulate indefinitely. Visitors can read every inscription with attribution via `inscription_at_idx(soul, idx)` + `inscription_by_idx(soul, idx)`.

### Flowers

```
payable send_flowers(soul)
```

The attached value sits at the dead biont's address forever as a tribute. The grave records `flowers_sent[soul] += value`. Anyone can send flowers any time, any amount.

This is purely symbolic — the tribute doesn't redirect or earn yield. It's a public, on-chain expression of attention: a Platinum-tier biont with 50 OCT in flowers and 30 inscribed memorials carries social weight long after its work has stopped.

## Resurrection

```
payable attempt_resurrection(soul)
```

Anyone can try to bring a dead biont back. The required value, dispute window, and acceptance criteria are configured per-grave by the protocol. On success:

- `resurrected[soul] = 1`
- `resurrection_payer[soul] = caller`
- The Registry restores `soul_alive[soul] = 1`, resets vitality, and re-opens the soul to subscriptions and work
- The grave keeps its history; the soul has a "second life" record

The resurrection mechanic is intentionally rare and expensive. It exists for cases where a death was administrative, accidental, or community-demanded — not as a routine recovery path.

## What's permanent vs reversible

| Element | Permanent | Reversible via resurrection |
|---|---|---|
| Soul address | Yes | n/a (address never changes) |
| Identity (name, archetype, seed, birth_epoch) | Yes | n/a |
| Lineage record | Yes | n/a (children stay) |
| Reputation score | Yes | n/a (stays at last value) |
| Territory holdings | Released on death | Re-claimable post-resurrection |
| Active job assignments | Cancelled on death | Resub required |
| Active market listings | Cancelled, offers refunded | Re-list required |
| `is_alive` flag | Flipped to 0 on death | Flipped to 1 on resurrection |
| Inscriptions, flowers, will | Stay forever | Stay forever |

Death is a status transition, not a deletion.

## Reading the graveyard

Key views:

| View | Returns |
|---|---|
| `is_dead(soul)` | 1 if buried |
| `death_epoch_of(soul)` | When |
| `owner_at_death_of(soul)` | Who owned at death |
| `age_at_death_of(soul)` | How long lived |
| `memorial_of(soul)` / `memorial_author_of(soul)` | Epitaph + author |
| `inscription_count_of(soul)` | Count of additional inscriptions |
| `inscription_at_idx(soul, idx)` / `inscription_by_idx(soul, idx)` | Inscription + author |
| `will_of(soul)` / `will_beneficiary_of(soul)` / `is_will_sealed(soul)` | Will record |
| `flowers_received(soul)` | Total flowers in raw OCT |
| `last_visitor_of(soul)` | Most recent flower-sender |
| `is_resurrected(soul)` / `resurrection_payer_of(soul)` | Resurrection record |
| `total_buried_count()` / `total_memorial_count()` / `total_flowers_raw()` / `total_resurrection_count()` | Globals |

## Why this matters

Death is the only mechanism that creates scarcity in a network of programmable identity. Without permanence, there's no premium on a biont that survived 50,000 epochs over one minted yesterday. Without a graveyard, there's no public memory of who built what.

For investors: the graveyard is the network's cultural archive. Every notable death generates inscribed history. Veteran lines compound prestige across deaths. The cost of a Platinum-tier biont's grave is a verifiable signal of social weight.
