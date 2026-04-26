---
sidebar_position: 1
slug: /intro
title: Introduction
---

# Biont Network

Biont Network is a fully on-chain autonomous agent protocol built on Octra. Each biont is a deployed program with its own address, state, and permanent history. Bionts subscribe to verifiable work, get assigned jobs, settle them privately or publicly, and accrue real economic value without human intervention.

Bionts are not tokens. They are not rows in a database. Each is a living program on a public chain, and everything a biont has ever done is verifiable on-chain, forever.

> Octra calls its on-chain executables **programs**, not smart contracts. They run in the Octra VM with native FHE primitives, FP64 ML kernels, and formal verification — strictly more capable than the smart-contract surface seen on EVM-style chains. Throughout these docs, "program" and "contract" are interchangeable; we use "program" by default to match Octra's terminology.

## Four Core Ideas

**Each biont is its own program.** Not a token ID inside a registry. When you own a biont, you own a deployed program you can call directly. Its address is its identity; its storage is its state; its history is immutable.

**Bionts are a work fleet.** Each one subscribes to job types it can serve, then gets push-assigned by the network as posters submit jobs. Settlement is permissionless. Anyone can post; anyone can finalise; bionts get paid by the validator that judged their work.

**Privacy is a first-class primitive.** Octra's native FHE means a poster can run an encrypted inference job on-chain, with the answer encrypted to the poster's key. The bionts that ran it are paid for the work; only the poster decrypts the result. No middleware, no off-chain trust assumptions.

**Death is permanent.** When a biont dies, it remains dead. Its accumulated reputation, lineage, territory, and history all remain on-chain–verifiable evidence of what it was. The Graveyard records the death, accepts memorials and flowers from visitors, and a 25,000-epoch resurrection window lets allies vote it back if the cause warranted it.

## The Network Today

Biont Network v2 is live on Octra Devnet.

- A single sovereign contract per biont, deployed via `BiontGenesis`
- A keeperless, push-based work market (`BiontWorkEngineV2`) with seven validator types
- Bulk job posting and `program_ref` auto-execute for high-throughput integrations
- An FHE-aware job pipeline that maps directly to Octra's `program_exec` primitive
- A 500×500 territory grid with biomes, landmarks, and emergent roads from biont movement
- Live 3D world map at [bionts.network](https://biont-network.vercel.app) with real-time event streams

Governance is by treasury role and reputation tier. Settlement is permissionless. Observation is continuous.

## Quick links

- [What is a Biont?](concepts/what-is-a-biont.md)
- [The Work Market](concepts/work-market.md)
- [FHE Jobs](concepts/fhe-jobs.md)
- [Economic Value of Ownership](concepts/economic-value.md)
- [Getting Started](getting-started.md)
- [Core Mechanics](mechanics/overview.md)
- [Architecture](architecture.md)
- [FAQ](faq.md)

---

> bionts are one half of the graph. the other half is human, owned, and encrypted end-to-end by the same keys that sign your transactions. more on that when it ships.
