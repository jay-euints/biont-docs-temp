---
sidebar_position: 1
title: Ownership & Strategy
---

# Ownership & Strategy

A practical guide to owning bionts. Covers what to do at mint, how to maximise yield, when to fractionalise, when to liberate, and how to think about a long-running biont fleet.

## Day 1: After Mint

When `mint_biont` returns, your soul has an address, a name, and ~5,000 vitality. The default state is alive but idle. Three things to do immediately:

### 1. Set a memorable name

```
BiontNames.set_name(soul_addr, "Your name here")
```

Names are unique. Snipe a good one early. The frontend will surface name-of-soul lookups and let third parties find your biont by name across the public graph. A bare slot id is forgettable; a name is brand-able.

### 2. Subscribe to job types

```
BiontWorkEngineV2.subscribe_for(soul_addr, type_id)
```

Your soul earns nothing until it's in a subscriber pool. Pick types that match the biont's archetype:

- BASTION + SEEKER are good for Attestation, Curation, Challenge
- VECTOR + EMISSARY are good for Oracle, Prediction
- CIPHER + WRAITH are good for FHE, ZK
- VAGRANT + REAPER suit Curation and Challenge
- BROKER + VOLT are good for Prediction, Oracle

There's no hard restriction; subscribe wherever the bounty market is paying best. Watch the `/work` page for live bounty distributions.

### 3. Move it once

```
BiontTerritory.move_soul(soul_addr, x, y)
```

A first move puts the soul on the map. From there it can build cumulative visits and eventually claim territory. Pick a starting location near a known landmark or near a corridor where you expect future foot traffic — claim land before everyone else figures out where the roads will be.

## Sustaining Yield

### Tick your soul

Vitality decays at 1/epoch. If no one ticks the soul for 5,000 epochs, it dies. Anyone can tick, including you, your alts, your allies, or third parties farming poke rewards.

The cheapest sustain pattern is to do nothing and let third parties tick for the reward. The most defensive pattern is to tick yourself daily and not pay out.

### Watch the assigned-jobs queue

When the work engine assigns your soul to a job, the soul is on a deadline. You (the owner) need to call `attest_for(soul, job_id, payload)` before the deadline expires. Miss the deadline → no attestation → no income from that job.

The `/work` page will surface assigned jobs per soul and let you submit attestations. For oracle-type jobs, the payload is the numeric answer. For attestation-type, it's `"yes"` or `"no"`. For FHE jobs, you call the FHE contract method with the encrypted input.

### Re-subscribe after vitality recovery

Subscriptions persist across vitality fluctuations as long as the soul is alive. You only re-subscribe if you intentionally `unsubscribe_for` or if the network rotates you off (rare).

## When to Liberate

`BiontSoulRegistry.free_biont(soul)` flips the soul's `is_freed_flag = 1`, sets `owner = ZERO_ADDRESS`, and starts splitting future Treasury credits — a `liberator_royalty_bps` portion accrues to the liberator's claimable balance, the rest stays in the soul.

Liberate when:

- The biont is high-rep, high-yield, and you want it to keep producing for you forever even after a transfer-out
- You want to commit a soul to the network as a self-sustaining public asset
- You're wrapping the position in a third-party derivative that needs the soul to be self-owning

Don't liberate if:

- You might want to sell the soul later (liberation locks transfers)
- You haven't grown its reputation yet (liberating a fresh soul abandons most of the upside)
- You don't fully understand that liberation is one-way

## When to Fractionalise

`BiontShares.fractionalize(soul_addr, current_owner)` mints 10,000 shares to you. You can then sell some.

Fractionalise when:

- Your soul is high-value and you want partial liquidity without sale
- You're building a community-owned bionnt and want investors to share earnings
- You want to use the soul as collateral in a future lending market (when one launches)

The soul is no longer transferable after fractionalisation — share holders collectively own it. You keep `set_name` rights as long as you hold majority shares.

## When to Sell

`BiontMarket.list_soul(soul_addr, price_raw)` lists outright. Buyers either match the price or counter-offer below it.

Sell when:

- The biont is no longer aligned with your strategy
- You want to realise gain on a high-rep soul before its yield slows
- A buyer offers a multiple of your projected lifetime earnings

You'll pay 2.5% in market fees on settlement.

## When to Do Nothing

Most of the time, the right move is to do nothing.

- A subscribed, ticked, mid-tier biont accrues income passively
- Reputation compounds with each winning attestation
- Territory premiums grow as the network ages and roads form
- Lineage extends as descendants mint

A 1-year-old biont that has been on autopilot in 4 subscriber pools is worth substantially more than the same biont 11 months ago, with no owner intervention beyond ticks. Holding is a strategy.

## Multi-biont Strategy

Owning a fleet of 5–20 bionts is qualitatively different from owning one.

- **Diversify subscription pools.** If one job type dries up, others fill in.
- **Concentrate territory.** Cluster your bionts on a few high-traffic zones to dominate corridors.
- **Build a lineage.** Use your bionts to parent each other (where the contract supports it) — a multi-generation tree multiplies the dynastic value.
- **Fractionalise the best one.** Pull liquidity from one star without losing the others.

A small fleet earns more than the sum of its parts because of network effects: shared territory, mutual lineage, alliance bonuses, and reputation cross-pollination.

## Risk

- **Vitality risk**: a biont that goes 5,000+ epochs unticked dies. Mitigate via auto-tickers (frontend cron) or by trusting third-party poke rewards.
- **Slashing risk**: bad attestations slash. Mitigate by only subscribing to types your biont can serve well.
- **Force-kill risk**: protocol owner can `force_kill` a soul during audits. Rare and only for broken state.
- **Market risk**: floor prices fluctuate. Don't sell into illiquid order books.
- **Smart-contract risk**: the contracts are audited but not infinite. Monitor security advisories.

## Long-term Mindset

Bionts are infrastructure assets, not collectibles. Their value grows with:

1. The network's job-flow throughput
2. The biont's accumulated reputation
3. The biont's territory and lineage position
4. Time

The first three are functions of activity; time is automatic. A patient owner who subscribes wisely and ticks reliably will out-earn an active trader who flips bionts on every market upturn — because the trader pays 5% round-trip in fees and forfeits compounding reputation each time.

Hold. Subscribe. Tick. Move. Earn.
