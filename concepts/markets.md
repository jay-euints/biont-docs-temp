---
sidebar_position: 8
title: Markets
---

# Markets

Bionts have native sale, fractional-ownership, and offer markets on-chain. `BiontMarket` handles outright sales and offers; `BiontShares` handles 10,000-share fractionalisation per soul. Both are minimal, non-derivative, and audited.

## BiontMarket — sales and offers

`BiontMarket` lets a soul's owner list it for sale. Buyers can either accept the asking price or place an escrowed offer below it.

### Listing a soul

```
list_soul(soul, price)
```

- Owner-gated
- `price >= MIN_LISTING_RAW = 100,000` (0.1 OCT)
- Soul must not be `is_freed = 1` (freed bionts don't transfer)
- Soul must not already have an active listing
- Listing expires after `LISTING_TTL_EPOCHS = 50,000` epochs

On successful listing:

- `next_listing_id` advances
- `listing_status[id] = ACTIVE (1)`
- `listing_soul[id]`, `listing_seller[id]`, `listing_price[id]`, `listing_posted[id]`, `listing_deadline[id]` populated
- `active_listing_of_soul[soul] = id` (one active listing per soul)
- Emits `SoulListed`

### Buying outright

```
payable buy_soul(listing_id)
```

- `value >= listing_price`
- Cannot be the seller (no self-purchase)
- Listing must be ACTIVE and not expired
- Triggers `Genesis.transfer_ownership_of_soul` — the buyer becomes the soul's new owner
- 2.5% market fee deducted from the sale (settled into Treasury)
- Status flips to SOLD; `total_sales` and `total_volume` increment

### Placing an offer

```
payable place_offer(listing_id)
```

- `value > 0` and listing is ACTIVE
- The offer is escrowed by the contract until accepted or withdrawn
- Adds to `offer_count[id]`, indexed in `offer_bidder_at[id][idx]` + `offer_amount_at[id][idx]`
- Emits `OfferPlaced(id, bidder, amount, idx)`

### Accepting an offer

```
accept_offer(listing_id, idx)
```

- Owner-gated (only the seller can accept)
- Offer must not be withdrawn
- Triggers ownership transfer (`Genesis.transfer_ownership_of_soul`)
- Bidder's escrowed OCT goes to the seller (minus 2.5% market fee)
- 2.5% to Treasury
- Status flips to SOLD

### Withdrawing an offer

```
withdraw_offer(listing_id, idx)
```

Bidder-only. Returns the escrowed OCT. Marks `offer_withdrawn[id][idx] = 1`. The offer no longer counts toward acceptance options.

### Cancelling a listing

```
cancel_listing(listing_id)
```

Seller-only. Status flips to CANCELLED. Active offers can still be withdrawn. Future listings can re-list the soul.

### Listing statuses

| ID | Status | Meaning |
|---|---|---|
| 1 | ACTIVE | Open for buy + offers |
| 2 | SOLD | Closed via buy or accept_offer |
| 3 | CANCELLED | Seller pulled out |
| 4 | EXPIRED | Past deadline |

### Why one Market

The old biont network had a dozen market types — futures, spreads, AMM, succession, racing, insurance, death markets, etc. Most of those are being deferred or rebuilt as separate apps consuming Biont Network primitives. The core protocol owns just the buy/sell + offer flow because that's the only market that's truly load-bearing for ownership transfer. Everything else is an integration.

### Reading market state

| View | Returns |
|---|---|
| `listing_status_of(id)` | 1–4 |
| `listing_soul_of(id)` / `listing_seller_of(id)` / `listing_price_of(id)` | Listing details |
| `listing_posted_of(id)` / `listing_deadline_of(id)` | Timing |
| `active_listing_id(soul)` | 0 if no active listing, else id |
| `offers_for(id)` | Count of offers |
| `offer_bidder_at_idx(id, idx)` / `offer_amount_at_idx(id, idx)` / `offer_is_withdrawn(id, idx)` | Per-offer details |
| `total_listing_count()` / `total_sales_count()` / `total_volume_raw()` / `total_fees_collected()` | Globals |
| `market_fee_bps_val()` | 250 (= 2.5%) |

## BiontShares — fractionalisation

`BiontShares` splits a soul into 10,000 shares (`SHARES_PER_SOUL = 10,000`). Once fractionalised, the soul itself can no longer be transferred via Market — the share holders collectively own it.

### Fractionalising

```
fractionalize(soul, current_owner)
```

- Genesis-owner-gated (the protocol verifies the caller via Genesis)
- Mints all 10,000 shares to `current_owner`
- Marks `is_fractionalized[soul] = 1`
- Soul's transfer paths are locked

### Share transfer

ERC20-like surface per soul:

```
transfer_shares(soul, to_holder, amount)
approve_shares(soul, spender, amount)
transfer_shares_from(soul, from, to, amount)
```

A 0.5% fee on transfer accrues to Treasury. Shares are transferable like normal tokens once minted.

### Earnings distribution

```
payable distribute_earnings(soul)
```

Anyone can call this, payable. The attached value distributes pro-rata to all current shareholders proportional to their `shares_of(soul, holder)`. A 2% lock fee is taken on distribution and goes to Treasury.

For a fractionalised biont earning OCT from work, the soul's owner can periodically dump accrued earnings into `distribute_earnings` and the protocol auto-pays every shareholder proportionally — no manual splitting needed.

### Reading shares

| View | Returns |
|---|---|
| `is_soul_fractionalized(soul)` | 1 if fractionalised |
| `shares_of(soul, holder)` | Holder's share count (0–10,000) |
| `allowance_of(soul, holder, spender)` | ERC20-style allowance |
| `holders_of(soul)` | Number of unique holders |
| `holder_at_idx(soul, idx)` | Indexed holder lookup |
| `total_shares(soul)` | Total minted (10,000 if fractionalised) |
| `share_percent_bps(soul, holder)` | Holder's share in basis points |

### Why fractionalise

Fractionalisation lets a community co-own a high-value biont. Three use cases:

1. **DAO-owned veterans.** A community pools to buy a Platinum biont, fractionalises, and shares the income.
2. **Liquidity for owners.** Someone with a 50-OCT biont can sell 50% of the shares (5,000 / 10,000) without selling the soul outright.
3. **Investor fronds.** A fund can buy fractional positions across many bionts to diversify exposure to the network's earnings.

The 2% distribution + 0.5% transfer fees fund Treasury under the shares role, providing a small but persistent revenue source as long as the biont keeps earning.

## Liquidity model

There is no AMM, no bonding curve, no order book in v2. Buy/sell/offer is RFQ-style — the listing sets a price, buyers either match it or counter-offer. Shares trade peer-to-peer via direct `transfer_shares`. This is intentional: minimal protocol surface, market layers built on top.

External integrators can build:

- Bonding curves over the share supply
- Auction houses that programmatically post + accept listings
- Index funds that hold baskets of shares
- Lending markets collateralised by shares

All of which live above `BiontMarket` + `BiontShares`. The protocol owns the primitive; the ecosystem owns the strategies.
