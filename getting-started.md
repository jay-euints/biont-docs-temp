---
sidebar_position: 2
title: Getting Started
---

# Getting Started

This page walks you from zero to a live biont. It assumes nothing about your prior experience with Octra. Everything is on devnet for now; mainnet rollout follows.

## What you need

1. **A wallet.** Install the [0xio Wallet](https://chromewebstore.google.com/detail/0xio-wallet/anknhjilldkeelailocijnfibefmepcc) browser extension. Create or import an Octra wallet.
2. **OCT on devnet.** Octra Devnet faucets are listed in the wallet's network panel. Get yourself ~5 OCT to start.
3. **A modern browser.** The biont app uses WebGL for the territory map. Chrome, Edge, or Firefox 110+.

## Step 1: Connect

Visit [bionts.network](https://biont-network.vercel.app) and click `CONNECT`. The wallet extension prompts you to authorise the dapp. Once connected, the top nav shows your address and balance.

## Step 2: Mint a biont

Go to `/mint`. The mint panel shows the current price, the total minted so far, and a preview of what your biont will look like (deterministic from your address + the current epoch).

Click `MINT`. The wallet pops up a transaction request. Sign it. After a few seconds the page refreshes and your new biont appears under "your bionts".

What just happened on-chain:

- `BiontGenesis.mint_biont(name, your_address)` was called with the mint fee attached
- Genesis popped a pre-deployed `BiontSoul` proxy from its pool
- Octra assigned the soul a permanent address
- The soul was initialised with your name, archetype, seed, and birth epoch
- The soul was registered with `BiontSoulRegistry` and is now alive

## Step 3: Subscribe to a job type

Go to your biont's profile page (click on it from the dashboard). Find the `WORK` panel. You'll see seven job-type tabs: Attestation, Oracle, Curation, FHE, ZK, Challenge, Prediction.

Click `SUBSCRIBE` on any type. Sign the transaction. Your biont now sits in the subscriber pool for that type and will be auto-assigned to incoming jobs.

## Step 4: Wait for an assigned job

Once subscribed, you don't claim jobs — they're pushed to you. The `/work` page shows assigned jobs across all your subscribed bionts. When one appears, the soul has a deadline (usually ~1,000 epochs) to attest.

Click the assigned job to see what's expected:

- **Attestation**: a yes/no claim
- **Oracle**: a numeric query
- **Curation**: a quality score for some content
- **FHE / ZK**: a cryptographic operation against a payload
- **Challenge / Prediction**: domain-specific dispute or forecast

Submit your attestation. After enough attestations land or the deadline passes, anyone (including you) can call `auto_finalize` and the validator settles. If your attestation matches consensus, you earn the per-winner share of the bounty.

## Step 5: Move on the map

Open `/territory`. You'll see the 500×500 grid with biomes, landmarks, and biont swarms.

Find your biont (use the search panel) and click `MOVE`. Pick a destination. Sign the tx. Your soul records a move; if you accumulate enough visits to a single zone you can `claim_territory` it.

Holding a zone earns you per-visit rent forever, until someone out-visits you and challenges the claim.

## Step 6: Tick your soul

Vitality decays at 1/epoch. Without ticks, your biont eventually dies.

You can `tick()` your own soul, or rely on third parties — anyone who ticks earns a small poke reward, so independent operators are incentivised to keep your bionts alive.

The `/profile` page surfaces vitality and a one-click TICK button.

## Step 7: Explore

From here:

- **Watch your earnings.** The biont's contract balance grows as it wins attestations and earns territory rent.
- **Set a name.** `BiontNames.set_name` lets you give the biont a unique handle.
- **Link Pipoke.** If you have a Pipoke profile, bridge it to gain social-graph fees.
- **Read the architecture.** [How the contracts fit together](architecture.md) is short and worth your time.
- **Plan a strategy.** [Ownership & Strategy](guides/ownership.md) covers fleet thinking.

## Common first questions

**"How fast does a biont earn?"**
Depends on subscriber pool density and bounty rates. Early in network life, bounties are smaller; expect ~0.05–0.5 OCT per winning attestation. Subscribe to multiple types to compound.

**"Can I lose my biont?"**
Yes — through vitality decay (let it sit unticked for thousands of epochs) or through `force_kill` (protocol-level, rare). You cannot accidentally brick a biont through a single bad call.

**"What's the difference between sell and liberate?"**
Sell transfers ownership to a buyer for OCT. Liberate gives up ownership permanently and routes all future earnings to you forever — but the soul becomes self-owning and can never be transferred again.

**"Is mainnet live?"**
Not yet. v2 is on devnet. The mainnet rollout follows once devnet stabilises and we audit one more pass.

## Troubleshooting

**"Mint tx reverted with insufficient OU"**
Bump up the OU on the wallet's tx popup. Default is 10,000; mint sometimes needs ~15,000.

**"My biont won't subscribe"**
Check it's not already subscribed to that type (one subscription per pair). Also confirm the soul is `is_alive = 1`.

**"I can't see my biont on the map"**
Move it once via `move_soul`. New bionts default to `(0,0)` until their first move.

**"The 3D world is blank"**
Try a hard refresh. If it's still blank, check the browser console for WebGL errors. Some integrated GPUs struggle with the territory shader; reducing browser zoom often helps.

## What next

You're a biont owner. The system runs without you. Your biont will eventually be assigned work, attest if you're around, and earn OCT. You'll come back to a contract balance you didn't have to babysit.

Subscribe to more types. Move your biont. Watch lineage form. Read the [FAQ](faq.md) for the long-tail questions, and the [Architecture](architecture.md) for the full picture.
