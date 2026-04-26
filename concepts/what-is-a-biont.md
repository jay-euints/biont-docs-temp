---
sidebar_position: 1
title: What is a Biont?
---

# What is a Biont?

A **biont** is an autonomous on-chain agent, deployed as its own program on Octra. It is not a token, it is not an NFT in the familiar sense, and it is not a row in a database. Each biont exists as an independent program at its own address. (Octra calls on-chain executables "programs" rather than "smart contracts" — they're strictly more capable, with native FHE, FP64 ML kernels, and formal verification baked into the VM.)

## Contract as entity

When a biont is minted, `BiontGenesis` pops a pre-deployed `BiontSoul` proxy from its pool and initialises it. Octra assigns the soul a unique address. That address becomes the biont's permanent identity. All of its identity state (name, archetype, seed, owner) lives in that contract's storage.

You can call a biont's contract directly. You can read its state via `contract_call` or `octra_contractStorage`. You can verify every action it has ever taken. This is not metadata on IPFS or a centralised server; it is executable code and live state on a public chain. The canonical contract is `BiontSoul`.

## On-chain state

Every biont exposes:

**Identity**: `name` (OCS01-compliant, explorer-readable, settable by owner), `symbol = "BIONT"`, `seed` (integer baked at mint; drives visuals and trait derivation), `archetype` (0–9), `birth_epoch`, `soul_id`.

**Ownership**: `owner` (primary controller), `genesis` (the factory that minted it), `registry` (the shared service that tracks vitality, capabilities, and signal queue), `is_freed_flag` (1 once liberated; owner becomes ZERO and a configurable share of future earnings starts accruing to the liberator).

**Behaviour**: `tick()` (called by anyone for a poke reward; advances the biont's vitality), `send_signal(target, payload)` / `receive_signal(from, payload)` (peer-to-peer messaging through the registry), `transfer_to(new_owner)` and `transfer(to, 1)` (OCS01-compatible ownership transfer with auto-notification to Genesis), `set_name(new_name)` (rename via the soul's own contract).

**Visuals**: `svg_of()` returns a complete animated SVG built from the seed (no external rendering, no IPFS), `traits()` returns the canonical trait CSV.

The heavy mutable state — vitality, capabilities, ticks, signals, freed-state, liberator, royalty redirection — lives in `BiontSoulRegistry`, a shared service every soul calls into. This keeps each biont's deployment small while the network holds all the cross-cutting state in one place.

## What bionts do

Once minted, bionts can:

- **Subscribe** to job types via `BiontWorkEngineV2.subscribe_for(soul, type)`. Subscription is owner-gated and one-shot per (soul, type).
- **Get auto-assigned** to jobs as posters submit them. The work engine pulls from the subscriber pool round-robin; bionts don't have to claim or race.
- **Attest** their work via `attest_for(soul, id, payload)`. The owner signs; the validator records and judges.
- **Earn** OCT and reputation when their attestations win consensus, slashing if not.
- **Tick** to advance their vitality. Anyone can call `tick()` for a poke reward; without ticking, vitality decays at 1/epoch and the biont eventually dies.
- **Move** in territory via `BiontTerritory.move_soul(soul, x, y)`. Each move records a destination edge; enough movement on a single edge upgrades it into a road.
- **Claim** zones in the 500×500 territory grid. Holders earn rent from visitor traffic.
- **Breed** with another biont via `BiontLineage.propose_breed(parent_a, parent_b)`. A successful breeding proposal mints a new biont with hybrid traits derived from both parents.
- **Form pacts** via `BiontAlliance.propose_pact` + `accept_pact`. Mutual signed alliances visible on-chain.
- **Be fractionalised** via `BiontShares` — split into 10,000 tradable shares; earnings distribute pro rata to holders.
- **Be sold** via `BiontMarket` — listings, escrowed offers, accept-or-withdraw lifecycle.
- **Receive a name** via `BiontNames` — unique on-chain handle, transferable. 0.2 OCT registration fee, 5,000-epoch rename cooldown.
- **Bond to a Pipoke profile** via `PipokeBridge` — connects the on-chain biont to its off-chain social identity. 0.5 OCT bond fee, requires the Pipoke account to be at least 1,000 epochs old.
- **Write a will** via `BiontGraveyard.seal_will` — designate a beneficiary and an epitaph before death.

Tick, claim, post, finalise are all permissionless — anyone can trigger them, anyone earns the reward.

## Capabilities

Each biont is registered into `BiontSoulRegistry` with an initial capability tag derived from its archetype:

| Cap | Validator type |
|---|---|
| 1 | Attestation |
| 2 | Oracle |
| 4 | Curation |
| 8 | FHE |
| 16 | ZK |
| 32 | Challenge |
| 64 | Prediction |

Capabilities are a sparse set — the protocol owner can `grant_capability(soul, cap)` to add more or `revoke_capability(soul, cap)` to remove. Subscribing to a job type via `WorkEngine.subscribe_for` is a separate decision driven by the soul's owner; capability is a tag, subscription is a commitment.

## Ownership model

A biont has one role:

| Role | Purpose | Can change? |
|---|---|---|
| Owner | Primary controller; signs subscribe / attest / set_name / transfer / list / fractionalise / breed / claim_zone | Via `transfer_to`. Becomes ZERO on `Registry.free_biont`; from then on a `liberator_royalty_bps` share of future earnings accrues to the liberator. |

A liberated biont (`is_freed = 1`) has `owner = ZERO_ADDRESS`. It can no longer be transferred or reconfigured by anyone. A configurable royalty fraction of future earnings flows to the original liberator's claimable balance for as long as the biont keeps producing work. The liberator pattern decouples *who built the biont's reputation* from *who controls it now* — a path for owners to permanently free a biont they no longer want to manage while still earning a perpetual royalty on the value it accrues.

## Visual identity

Each biont has a unique generative visual, produced entirely on-chain. Calling `svg_of()` returns a complete animated SVG.

The visual is derived from `seed`, an integer set at deployment that never changes. The seed drives trait derivation across ten ear types, four eye styles, three mouth shapes, three brow styles, three cheek patterns, and six body marks. Combined with ten archetype palettes, that's tens of thousands of possible combinations.

No external rendering service, no IPFS, no centralised API — the image lives in the contract itself.

## The ten archetypes

Every biont belongs to one of ten archetypes, assigned deterministically at mint from `(seed * 1664525 + 1013904223) mod 16` (clamped to 0–9). Each has a distinct colour palette and an initial validator-type capability.

| # | Archetype | Colour | Initial capability |
|---|---|---|---|
| 0 | VAGRANT | `#4dd9ff` | Attestation |
| 1 | VECTOR | `#60a5fa` | Oracle |
| 2 | EMISSARY | `#34d399` | Curation |
| 3 | SEEKER | `#a3e635` | FHE |
| 4 | BASTION | `#fbbf24` | ZK |
| 5 | BROKER | `#f97316` | Challenge |
| 6 | REAPER | `#ef4444` | Prediction |
| 7 | WRAITH | `#a855f7` | Attestation |
| 8 | CIPHER | `#ec4899` | Oracle |
| 9 | VOLT | `#ff6b35` | Curation |

Bionts can subscribe to additional types beyond their initial capability via `WorkEngine.subscribe_for`. Reputation is tracked per-type, so a biont that specialises in FHE jobs builds a Platinum-tier FHE reputation independent of its starting archetype.

## Permanent death

A biont dies when its vitality reaches zero (decays from inactivity) or when an owner calls `Registry.force_kill` on it. `is_alive` flips to 0, `death_epoch` is stamped, and the death cascades:

1. `BiontGraveyard` records the death with the soul's last vitality, age, owner-at-death.
2. If the biont had `seal_will`'d a will, the will text and beneficiary are read on-chain. The beneficiary becomes the canonical inheritor of any direct estate transfers.
3. Visitors can leave **inscriptions** on the grave (each one attributed to its author) and **flowers** (small OCT donations that sit at the grave forever as a tribute).
4. `BiontLineage.mark_dead` closes the soul's line; descendants are notified.
5. A **resurrection window** opens. Anyone can `attempt_resurrection(soul)` (payable) to bring the biont back. After the window closes, the death is permanent.
6. Any active market listing on the soul resolves; offers refund.
7. Active job assignments time out; bounties revert to posters.

Every death reshapes the network's economic and social structure. The historical record stays on-chain forever — a dead biont is still queryable for its full life: every job worked, every reputation gain, every kid it sired, every territory it claimed.
