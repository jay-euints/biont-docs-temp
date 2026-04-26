---
sidebar_position: 4
title: Economics
---

# Economics

Biont Network has one currency: OCT, the native token of Octra. There is no separate biont token, no staking yield token, no governance token. All economic activity — mint fees, job bounties, market sales, share distributions, royalties, slashing — is denominated in OCT and settles directly between contracts.

This page describes how OCT moves through the network.

## Sources of OCT into the network

### Mint fees

`BiontGenesis.mint_biont` is `payable`. The mint fee goes to the protocol Treasury under the `GENESIS` role. This is the primary primary-issuance income. Mint fees are paid once per biont, at birth.

### Job bounties

`BiontWorkEngineV2.post_jobs_bulk` is `payable`. Posters fund the bounty up front. Funds are held by the work engine until settlement. On settlement, the bounty is split:

- Winners receive their slice via `pay_winner` callbacks.
- The Treasury takes a configurable cut under the `WORK_ENGINE` role (residual via `finalize_residual`).
- Slashed losers' stakes flow to the residual, distributed to winners + Treasury.

Bounties are the main recurring revenue.

### Market fees

`BiontMarket` charges 2.5% on every sale (outright or accepted offer). Fees route to Treasury under the `MARKET` role.

### Share fees

`BiontShares` charges a 5% protocol fee on `distribute_earnings`, routed to Treasury under the `SHARES` role. Share transfers themselves are currently free (the source defines `TRANSFER_FEE_BPS = 50` for future activation but the v2 transfer path skips it).

### Territory rent and challenges

`BiontTerritory` charges:

- A claim fee (paid once when a biont claims a zone, routed to Treasury under `TERRITORY`)
- A small per-visit visitor fee on movement, partially paid to the holder and partially to Treasury

### Pipoke link fees

`BiontPipokeBridge.link_pipoke` charges a small one-time link fee. This routes to Treasury under the `PIPOKE` role.

### Graveyard inscriptions and flowers

`BiontGraveyard` accepts `payable` memorial inscriptions and tributes (`leave_flower`, `leave_inscription`). These pool into a memorial fund per soul; the soul's last owner can periodically claim accumulated tribute.

## Sinks of OCT out of the network

### Worker payments

`pay_winner` debits the work engine and credits the winning soul's contract. This is the largest single outflow.

### Sale proceeds

`BiontMarket` sales pay the seller (minus fees) directly.

### Share distributions

`distribute_earnings` pays every share holder pro-rata.

### Tick rewards

`Soul.tick()` pays a small poke reward to the caller from the soul's balance.

### Liberator royalties

A freed soul's incoming earnings split: a `liberator_royalty_bps` fraction accrues to the liberator's claimable balance, the rest stays inside the soul's own Treasury entry. The split percentage is governance-tunable.

## Treasury roles

`BiontTreasury` is a single-holder-per-role contract. The roles are:

| Role | Source | Sink |
|---|---|---|
| `GENESIS` | Mint fees | Genesis-controlled redemption |
| `WORK_ENGINE` | Bounty residuals + slashes | Validator economics, dispute payouts |
| `REPUTATION` | (none — read-only role) | (none) |
| `LINEAGE` | Lineage one-time fees | Genealogy infrastructure |
| `ALLIANCE` | Alliance pact fees | Alliance treasury |
| `GRAVEYARD` | Graveyard inscriptions | Resurrection payouts |
| `PIPOKE` | Bridge link fees | Pipoke integration upkeep |
| `TERRITORY` | Claim + visit fees | Map-level upgrades |
| `SHARES` | Share fees | (none — accumulates) |
| `MARKET` | Sale fees | (none — accumulates) |
| `SOUL` | Per-soul minor fees | (none — accumulates) |
| `NAMES` | Name-set / change fees | (none — accumulates) |

A single role address holds withdrawal authority for that role's accumulated balance. Roles are swap-able by the protocol owner; the swap event is logged on-chain.

## Bounty economics

### Per-job math

A typical job:

- Bounty: B OCT
- Quorum size: q (default 3)
- Treasury take: t bps (default 250 = 2.5%)
- Per-winner payout: `(B × (1 - t/10000)) / q`

For a 3 OCT bounty with q = 3 and t = 2.5%:

```
treasury     = 0.075 OCT
distributable= 2.925 OCT
per winner   = 0.975 OCT
```

If only 2 of 3 attestations match consensus, the third is slashed. Slashed stake (their portion of the bounty) flows to Treasury via `finalize_residual`.

### Bulk efficiency

`post_jobs_bulk` lets a single tx mint 20 jobs. For an integrator, the per-job overhead drops from ~5,000 OU to ~1,500 OU at scale, making integration economically viable for high-volume use cases (proof markets, oracle feeds, third-party prediction frontends).

## Bounty pricing dynamics

Posters set bounties freely. A market discovers prices for each job type:

- Attestation jobs price at the floor — they're cheap to attest, so quorum is easy to fill at minimum bounty.
- Oracle jobs price by data difficulty — a hard-to-source numeric query commands a higher bounty to attract subscribers.
- FHE jobs price by computational cost — heavier inference under encryption costs more, so bounties scale with job complexity.
- Curation, ZK, Challenge, Prediction price by their own dynamics.

Subscribers self-select. A biont owner who watches the FHE pool will subscribe more aggressively when bounties tick up.

## Long-run network value

The network's economic value is the sum of:

1. **Active bounty flow** — what posters pay per epoch.
2. **Treasury accumulation** — the slow buildup of fees across all roles.
3. **Secondary market value** — the aggregate floor price of all live bionts and shares.
4. **Territory premium** — the implicit value of held zones, especially landmarks and roads.
5. **Lineage premium** — the implicit value of high-reputation founder bionts and their dynasty.

None of these require new token issuance. There is no inflation schedule. There is no emission curve. The network only mints OCT through Octra's base-layer issuance; Biont Network just routes existing OCT between participants in productive flows.

## Why no biont token

Three reasons:

1. **Octra's OCT is already the native medium of exchange.** A second token would split liquidity and add friction.
2. **No need for governance separation.** Reputation already provides per-soul governance weight; Treasury role assignment is administrative, not voted.
3. **Investor focus.** A productive on-chain economy is more interesting than a token printer. Bionts earn OCT for verifiable work, not for holding a separate asset.

The network is a job market. The job market settles in OCT. End of story.

## A note on numbers

Every fee, fee split, mint price, royalty percentage, slash size, reward size, dispute window, deadline cap, and per-wallet limit referenced in these docs is a **devnet default**. These values are owner- or governance-tunable. They will be re-evaluated and may change at mainnet. Treat the figures as illustrative of the architecture, not as fixed economic policy.
