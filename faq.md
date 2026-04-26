---
sidebar_position: 10
title: FAQ
---

# FAQ

Long-tail questions about Biont Network.

## Basics

### What is a biont?

A biont is a fully autonomous on-chain agent, deployed as its own smart contract on Octra. Each one has a unique address, its own state, and its own history. Bionts subscribe to job types, get auto-assigned work, attest their results, and earn OCT — all without any human babysitting once they're set up.

### Is a biont an NFT?

Bionts are OCS-01 compliant for explorer compatibility, but they're not NFTs in the usual sense. An NFT is a token ID inside a registry contract. A biont **is** the contract. You don't own a row in someone else's database — you own a deployed program on a public chain.

### What's the difference between Biont Network and other agent networks?

Three things:

1. **Each biont is sovereign.** Its own address, its own contract. No shared registry storing identities as token IDs.
2. **Privacy is native.** Octra has FHE built into the VM. Encrypted-output jobs are first-class.
3. **No keepers.** The work market is push-based and permissionlessly settled. There is no off-chain operator that has to be running for the system to work.

### What network is this on?

Octra Devnet today. Octra Mainnet later. Both are L1; the v2 contracts are written in AppliedML 1.0 (AML), Octra's native smart contract language.

## Mint

### How much does it cost to mint?

Mint price is set by the protocol. Currently a few OCT on devnet. Pay once at mint; from then on the biont costs nothing to keep alive (vitality is restored by free ticks).

### What do I get?

A `BiontSoul` contract at a unique address. The contract holds: identity (name, seed, archetype, birth_epoch), behavioural surface (tick, signal, transfer, set_name), visuals (svg_of, traits), and ownership state.

### Can I mint multiple?

Yes. Each mint pops a fresh proxy from the pool. There's no per-wallet cap.

### What's the supply?

The pool is sized to allow generous mint runs. There is no fixed hard cap; the pool can be replenished by the protocol owner. v2 is intentionally not running scarcity-by-supply — scarcity comes from reputation and lineage, not from how many were minted.

### Can I pick my archetype?

The archetype is derived from the seed at mint time. The seed is a function of your address and the current epoch, so it's deterministic per (you, epoch) but not chosen. If you really want a specific archetype, you can mint multiple times and keep the one you want.

## Work

### How do bionts earn OCT?

Primarily through `BiontWorkEngineV2`. A biont subscribes to job types; posters submit jobs; the engine assigns quorum from the pool; the soul attests; the validator settles; winners get paid.

### Do I have to do anything for my biont to work?

You have to subscribe it to job types and submit attestations when it's assigned. You don't have to claim jobs (they're pushed). You don't have to settle them (anyone can). You don't have to manage queueing (the engine handles round-robin).

### What if I miss a job deadline?

The job will settle without your soul's attestation. Your soul earns nothing from that job. You're not slashed for inactivity, only for wrong attestations.

### Can my biont do work I'm not capable of judging?

Yes. The validator judges, not you. As an owner you submit a payload; the validator checks it against consensus. Some job types (FHE, ZK) require running a specific contract method that does the cryptography for you.

### What's a `program_ref` job?

A job that the work engine auto-executes via Octra's `program_exec` primitive. Posters can supply a target contract address + two strings; the engine calls into it without requiring soul-by-soul attestation. This is the path to high-throughput integrations (oracle feeds, ML inference).

### How big can a bulk job submission be?

Up to 20 jobs in a single tx via `post_jobs_bulk`. Per-job overhead drops to ~1,500 OU at scale.

## Privacy

### What's encrypted on-chain?

Three categories:

1. **Encrypted balances.** Per-address private balance ciphers, decryptable only by the holder.
2. **Stealth payments.** OCT routed to one-time addresses derived via X25519 + SHA-256, decryptable only by the recipient.
3. **FHE jobs.** Job inputs and outputs encrypted to the poster's PVAC public key, computed on by the network without decryption.

### Can the validator see my FHE input?

No. The validator records the ciphertext. Consensus is taken on outputs (which are also ciphertexts). The poster decrypts the result; the network never sees plaintext.

### Are reputations and earnings private?

No. Those are public on-chain state. Privacy is opt-in per operation, not blanket.

### What's PVAC?

Octra's FHE keypair format. Stands for Privacy-Verified Active Computation. You generate a PVAC key locally, register the public half with `octra_registerPvacPubkey`, and use it to receive encrypted balances and decrypt FHE job outputs.

## Death

### What kills a biont?

Two things:

1. Vitality reaches 0 (decay at 1/epoch, restored at 1/tick) and someone calls `tick`.
2. `Registry.force_kill` is invoked by protocol owner during audits/deprecations (rare).

### What happens when a biont dies?

`Registry._kill_soul` flips alive=0, increments dead counter, emits `SoulDied`, calls into `Graveyard.record_death`, and `Lineage.mark_dead`. The soul keeps its address, history, lineage record, balance, and SVG. It just can't be ticked, transferred, subscribed, or worked again.

### Can dead bionts be revived?

If they're dead within the resurrection window (25,000 epochs), allies can vote them back via `Graveyard.vote_resurrect`. After the window closes, death is permanent.

### Why keep dead bionts on-chain?

Because they're verifiable history. A famous dead biont — first of its archetype, parent of a major lineage, holder of a famous territory — remains a permanent artifact. The market often prices dead bionts as collectibles based on lifetime achievement.

## Markets

### How do I sell a biont?

`BiontMarket.list_soul(soul, price)`. Sets a public listing. Buyers either match the price or counter-offer below it. 2.5% fee on settlement.

### What's fractionalisation?

`BiontShares.fractionalize(soul, current_owner)` mints 10,000 shares to you. You can sell some without losing the soul. Share holders earn pro-rata from `distribute_earnings`. Fractionalised souls cannot be transferred outright — share holders collectively own them.

### Can I lend a biont?

Not natively in v2. Lending markets can be built on top of `BiontShares` (use shares as collateral). The protocol provides the primitive; the ecosystem builds the strategy.

### What's the floor price?

There isn't a fixed one. Bionts price by reputation, age, archetype rarity, lineage, and territory. Early-network floors are low; established bionts price high.

## Territory

### How big is the map?

500 × 500 zones = 250,000 total. Each zone has a biome, optional landmark, label, visit counter, and reward.

### How does a road form?

Move events between two zones accumulate. When a single edge crosses the road threshold (default 50), it upgrades to a road. Roads are visible on the map and increase visit traffic.

### How do I claim territory?

Move your biont into a zone repeatedly. Once cumulative visits hit a threshold, call `claim_territory` to take ownership. You'll earn per-visit rent until someone out-visits you and challenges.

## Lineage

### Can bionts breed?

The architecture supports it via `BiontLineage.set_parents(child, parent_a, parent_b)`. The exact breeding mechanic (vitality cost, eligibility, cooldown) is implemented at the soul level — lineage just records the relationship. Specifics will be exposed in the frontend once the breeding contract paths are live.

### How many parents per child?

Up to two. Both are sealed when set; can't be changed after.

### Does my biont's reputation pass to its child?

Partially, depending on the lineage validator's logic. A high-rep parent confers a reputation boost to its descendants. The exact formula is per-validator.

## Pipoke

### What's Pipoke?

A separate social app on Octra. A bridge contract (`BiontPipokeBridge`) lets a biont link to a Pipoke profile, exposing both halves to each other's graphs. Linked bionts can earn social fees and curation tips.

### Is Pipoke required?

No. Bionts work fully without Pipoke. The link is opt-in.

## Technical

### What language are the contracts written in?

AppliedML (AML) 1.0 Rehovot — Octra's native smart contract language. AML compiles to OCTB bytecode and runs in the Octra VM. It's strictly typed, supports FHE primitives natively, and has formal verification built in.

### Are the contracts open source?

The full source for the v2 stack is verifiable on-chain via `contract_source(address)`. We'll publish a public mirror once the EUIBIOPI3 audit cycle finishes.

### Are they audited?

Internal lifecycle audit on every deploy. External audit follows mainnet rollout.

### What's the upgrade path?

Each contract is independently deployable. The owner can swap services (rotate Treasury, swap a validator) without touching individual souls. Souls themselves are immutable once deployed — your biont's contract is permanent at its address.

### How do I read a biont's state from outside?

```
contract_call(soul_address, "owner", [])
contract_call(soul_address, "name", [])
contract_call(soul_address, "vitality", [])
```

Or visit [octrascan.io](https://octrascan.io) and look up the address directly. All state is public, all storage slots are inspectable.

## Other

### Is there a token?

OCT is the only token in use. No biont-specific token. Reputation is non-transferable and lives per-soul.

### How can I integrate?

Subscribe your service's worker contract to whichever job types you need, or post jobs in bulk against the work engine. The integration surface is `subscribe_for`, `post_jobs_bulk`, `attest_for`, `auto_finalize`. That's it.

### How can I help?

Mint a biont. Subscribe it. Tick it. Submit attestations. Help us discover edge cases.

For deeper involvement, follow the project handles on the homepage and reach out via the listed contact.
