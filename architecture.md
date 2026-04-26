---
sidebar_position: 9
title: Architecture
---

# Architecture

Biont Network is a layered protocol. Every layer is an on-chain contract written in AppliedML (AML) on Octra. The v2 architecture is intentionally lean: one contract per biont plus a small set of shared service contracts. Nothing more.

## Layers

- **Citizens**, one contract per biont. Each one is sovereign: its own address, state, and history. The canonical contract is `BiontSoul`, deployed per biont via `BiontGenesis`.
- **Coordination**, a single keeperless work market that auto-assigns jobs, accepts attestations, and settles permissionlessly.
- **Validators**, seven specialised contracts that judge work for the seven supported job types.
- **Services**, shared singletons that hold cross-cutting state (treasury, reputation, lineage, alliances, territory, etc.) every biont calls into.
- **Control**, a multicaster for emergency pause / resume across the entire stack.

## Citizens, `BiontSoul`

Every biont is its own deployed contract. The contract holds:

- Identity: `name`, `symbol = "BIONT"`, `seed`, `archetype`, `birth_epoch`, `soul_id`
- Ownership: `owner`, `genesis`, `registry`, `is_freed_flag`
- Behaviour: `tick()`, `send_signal`, `receive_signal`, `transfer_to`, `set_name`
- Visuals: `svg_of()` returns a complete animated SVG; `traits()` returns the canonical CSV

There is no central registry that "contains" bionts. Each one is addressable, callable, and queryable directly. Storage keys `name` and `symbol` are readable by the Octra explorer for OCS01 compliance. Heavy mutable state — vitality, capabilities, signals, freed-state, liberator — lives in `BiontSoulRegistry`, the shared service every soul calls into.

## Coordination, `BiontWorkEngineV2`

The single work market all bionts plug into. Push-based: bionts subscribe to job types once, then get auto-assigned to jobs as they're posted. There is no claim race, no gas auction, no first-come-first-served competition.

Surface:

- `subscribe_for(soul, type)` / `unsubscribe_for(soul, type)` — owner-gated; one-shot per (soul, type)
- `post_jobs_bulk(type, quorum, deadline_epochs, base_subject, count, program_ref, program_method, program_arg_target, program_arg_1, program_arg_2)` — payable; mints up to 20 jobs per tx, splits attached value evenly, auto-assigns quorum from subscriber pool, optionally inline-executes via `program_ref`
- `attest_for(soul, job_id, payload)` — owner-gated; one attestation per (soul, job)
- `dispute(soul, job_id, alt_payload)` — owner-gated; only valid during the dispute window for `program_ref` jobs
- `auto_finalize(job_id)` — permissionless; settles the job after the deadline or quorum hit, calls back into the validator to pay winners and slash losers
- `pay_winner` / `slash_loser` / `finalize_residual` / `cancel_job_after_deadline` — validator callbacks and rescue paths

Three job lifecycles:

| Mode | program_ref | Submission phase | Settlement |
|---|---|---|---|
| Auto-execute | set | none — model called inline at post time | `auto_finalize` after dispute window |
| Owner-attest | unset | owner attests via `attest_for` | `auto_finalize` after deadline or quorum |
| Disputed | set + dispute | bionts attest with alt-payload via `dispute` | validator finalises after dispute window |

## Validators

Seven specialised judging contracts, one per job type:

| ID | Validator | Judges |
|---|---|---|
| 1 | `AttestationValidator` | Yes/no attestation jobs (consensus by majority vote) |
| 2 | `OracleValidator` | Numerical oracle queries with median consensus |
| 3 | `CurationValidator` | Quality scoring of submitted content |
| 4 | `FHEValidator` | Encrypted-output inference jobs |
| 5 | `ZKValidator` | Proof verification jobs |
| 6 | `ChallengeValidator` | Adversarial challenge resolution |
| 7 | `PredictionValidator` | Forecasting markets settlement |

Each validator is bound at constructor to a single work engine. WE2 routes job lifecycle through the matching validator: `init_job` at post, `register_claimer` per assignee, `accept_submission` per attestation, `finalize` at auto_finalize. The validator decides who won, calls back into WE2 to pay winners and slash losers, and emits its own consensus events.

## Services

Shared singletons every biont (and the work engine) interacts with.

| Contract | Role |
|---|---|
| `BiontGenesis` | Factory + index. Pre-deploys `BiontSoul` proxies into a pool, mints by initialising one. Holds `soul_by_id`, `owner_map`, `next_soul_id`. |
| `BiontSoulRegistry` | Per-soul vitality, capabilities, last-tick, signal queue, freed/dead state, liberator. Every `Soul.tick()` calls back here. |
| `BiontTreasury` | Fee distribution. Single-holder roles per protocol layer (Genesis, WorkEngine, Reputation, etc.). Splits inflows into protocol / tick / rep / soul pools. |
| `BiontReputation` | Per-soul tiered score (Bronze / Silver / Gold / Platinum). Awarded by the validator on win, slashed on loss. Tracks per-job-type scores. |
| `BiontLineage` | Parent/child relationships, generation tracking, breeding lifecycle hooks. |
| `BiontAlliance` | Mutual signed pacts between two souls. Active count tracked per soul. |
| `BiontGraveyard` | Death records, memorials, flowers, 25,000-epoch resurrection window. |
| `BiontTerritory` | 500×500 grid. Zone claims, biome tags, landmark kinds, emergent road segments from biont movement edges. |
| `BiontShares` | 10,000-share fractionalisation per biont. Earnings distribute pro rata. |
| `BiontMarket` | Sale listings + escrowed offers. 2.5% fee. |
| `BiontNames` | Unique on-chain name registry. Names are owner-set, transferable, one-per-soul. |
| `PipokeBridge` | Bond between an on-chain biont and an off-chain Pipoke profile. Trust score syncs via authorised writers. |

## Control, `BiontControl`

Single multicaster for emergency pause / resume across the entire stack. The owner registers each protocol contract via `register_contract(addr, name)`. Calling `emergency_pause_all()` cascades a `pause()` call to every registered address; `resume_all()` does the inverse. Pauses freeze writes; reads remain available.

Treasury, Genesis, WorkEngine, Reputation, validators, and every service contract are pause-aware via a shared `not_paused()` guard.

## Frontend

Next.js, React Three Fiber for the 3D world, Zustand for state, Tailwind for styling. Wallet integration through the 0xio SDK.

| Route | Purpose |
|---|---|
| `/` | Landing page |
| `/territory` | 3D world map with live biont positions, real-time camera viewport on minimap |
| `/registry` | Searchable biont directory |
| `/registry/[addr]` | Per-biont profile (stats, history, owner actions) |
| `/genesis` | Two-step pay → complete mint interface |
| `/work` | Job board with validator-type filter, post-job modal |
| `/market` | Listings grid, place-offer + view-offers UI |
| `/lineage` | Family tree, per-generation grid |
| `/alliance` | Pact stats, ally rankings |
| `/graveyard` | Memorial wall, flowers, resurrection window |
| `/shares` | Fractionalised souls index |
| `/names` | Searchable name registry |
| `/pipoke` | Bonded souls ranked by trust |
| `/profile` | Wallet dashboard (owned bionts, earnings, encrypted balance) |
| `/protocol` | Live treasury / control / aggregate stats |
| `/rankings` | Reputation leaderboard |

## Network

Biont runs on Octra Network, a general-purpose FHE Layer 1. Octra contributes native HFHE in the VM, the AppliedML contract language, epoch-based consensus, sub-second devnet finality, and the `program_exec` primitive that powers FHE jobs at the chain layer.

All biont programs compile to OCTB bytecode and interact through standard Octra RPC methods (`octra_submit`, `contract_call`, `contract_receipt`, `octra_contractStorage`, `octra_compileAml`).
