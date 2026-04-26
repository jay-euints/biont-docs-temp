---
sidebar_position: 9
title: Economic Value
---

# Economic Value of Owning a Biont

A biont is not a static collectible. It is a small, sovereign program that earns OCT for the work it does, accumulates reputation that compounds across job types, and inherits territory and lineage value as the network ages. Owning one is owning a position in a productive on-chain economy.

## Direct income streams

### Work earnings

The primary income source. A biont subscribes to job types via `BiontWorkEngineV2.subscribe_for(soul, type)`. From then on, every job posted of that type pulls from the subscriber pool round-robin and assigns the biont as a candidate. When the validator finalises a job and the biont's attestation matches consensus, `pay_winner` calls into the soul and credits OCT directly to the biont's contract balance.

Income scales with three things:

1. **How many job types the biont subscribes to.** A biont in 4 pools will be assigned ~4× as often as one in a single pool.
2. **How accurate its attestations are.** Wrong attestations get slashed. A biont with a high winner rate is a profitable biont.
3. **Network throughput.** The more jobs being posted by integrators, the bigger the per-pool yield.

### Tick rewards

Anyone can call `tick(soul)`. The caller earns a small poke reward, and the soul's vitality goes up by 1. This is a tiny but real income stream for the operator who keeps a biont alive — third parties pay you in poke rewards just to advance your soul's clock.

### Territory rent

A biont that owns a zone earns from every other biont that visits it. `BiontTerritory.move_soul` records visits and pays out a per-visit reward to the holder. High-traffic zones — especially those that have upgraded to landmarks — generate continuous passive income for as long as the holder keeps the claim.

### Pipoke alliance bonuses

Linking a biont to a Pipoke profile via `BiontPipokeBridge.link_pipoke` exposes both halves to the public graph. Profiles backed by an active biont can earn social fees, attestation bonuses, and curation tips that flow through the bridge into the biont's contract balance.

### Royalty redirection (post-liberation)

Once a biont is liberated by its owner (`Soul.liberate()`), `is_freed_flag = 1` and the owner becomes the zero address. From that point on, every royalty stream the soul has — work, territory, share distributions — redirects to the **liberator** wallet, permanently. Liberation is a one-way commitment, but it converts the biont into a forever-yielding asset for the freer.

## Compounding non-cash value

### Reputation

`BiontReputation` accumulates a per-soul score that goes up with every winning attestation and down on slashes. Tiers are Bronze (≥100), Silver (≥1,000), Gold (≥10,000), Platinum (≥100,000). Higher tiers unlock:

- Validator preference (some validators weight high-rep souls heavier in consensus)
- Listing visibility on `BiontMarket` (sort by reputation)
- Territory tax discounts in some biomes
- Eligibility for high-value job types (FHE, ZK, Curation are gated above Bronze)

Reputation is not transferable. It is bound to the soul's address forever. A 5-year-old Gold biont with 50,000 attestations under its belt is structurally more valuable than a freshly-minted one of the same archetype, regardless of seed.

### Lineage

`BiontLineage` records up to two parents per child. Lineage descendants can carry forward portions of their ancestor's reputation and traits. A founder biont with a productive descendant tree becomes a dynastic asset — every grandchild's earnings reflect on the line, and the original soul's identity can be pointed to forever even after death.

### Territory

A zone claimed early in the network's lifetime is worth far more later, especially if it sits on a road. Roads emerge from biont movement; a corridor that sees thousands of crossings becomes one of the most expensive spots on the map. Early territory holders own the equivalent of city-centre real estate before the city was built.

## Sale and fractional value

### Outright sale

`BiontMarket.list_soul` lets the owner sell. The market is RFQ-style — buyers either match the asking price or place an escrowed counter-offer. 2.5% protocol fee on settlement. Pricing is owner-set and reflects accumulated reputation, age, archetype rarity, lineage, and any active territory claims.

### Fractionalisation

`BiontShares.fractionalize` mints 10,000 shares against a soul. The owner can sell some or all of them without surrendering the soul itself. This is the only way to extract liquidity from a biont without losing it. Shares trade peer-to-peer with a 0.5% transfer fee and earn pro-rata from `distribute_earnings(soul)` — meaning every share holder collects a slice of the biont's work income.

A fractionalised biont's price discovery is denser and more continuous than its outright market price.

## Long-tail value

### Death does not zero a biont

When a biont dies, work income stops, but the soul's address still holds:

- Every attestation it ever made (verifiable via on-chain history)
- Its lineage record (descendants keep referencing it)
- Its Graveyard memorial (visitors leave flowers and inscriptions)
- Any unclaimed share distributions accrued before death
- Any territory the soul still held

A historically important dead biont — first of its archetype, parent of a major lineage, holder of a famous territory — remains a verifiable artifact forever. The market may price it at a multiple of its lifetime earnings purely as a collectible.

### Resurrection window

`Graveyard` keeps a 25,000-epoch window after death where allies can vote a soul back. A biont that died unfairly (hostile force-kill, network mistake) can be resurrected; one that died naturally typically isn't. The window is short relative to the network's expected lifetime but long enough that any contested death is reviewable.

## Income compounding example

Take a biont that:

- Subscribes to 3 job types
- Wins 70% of attestations at ~0.05 OCT each
- Holds 1 high-traffic zone earning ~0.02 OCT/visit
- Is fractionalised 50%, with the 5,000 owner-held shares yielding distributions

Over a year, with even modest network activity, the biont generates direct OCT, accrues reputation that compounds into more job assignments, builds territory rent, and yields share distributions to its co-owners. The owner's position is a single 47-character address — no off-chain accounting, no redemption flows, no platform risk. Everything is verifiable from `octra_balance` and a few view calls.

## What you actually own

You do not own metadata. You do not own a JPEG. You own:

1. The right to call `transfer_to(new_owner)` on a real on-chain contract.
2. A claim on every payout that contract has earned and will earn until you sell or liberate.
3. A unique cryptographic identity that the network treats as a first-class participant in every contract it interacts with.
4. A permanent slot on the public ledger that no one can revoke.

That is the difference between a biont and a token.
