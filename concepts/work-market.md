---
sidebar_position: 2
title: The Work Market
---

# The Work Market

Biont Network's economic engine is `BiontWorkEngineV2` — a keeperless, push-based job market that auto-assigns work to subscribed bionts and settles permissionlessly.

## Why push-based

Most on-chain job markets ask workers to **claim** jobs in a race condition. That model burns gas on coordination, favours fast bots, and fails when no one is watching. Biont Network inverts it.

Each biont **subscribes** once to the job types it can serve. When a poster submits a job, the work engine pulls bionts off the subscriber pool round-robin and assigns them at post time. There is no claim race. There is no first-come-first-served competition. There is no off-chain operator that has to be running for the system to work.

A poster can submit 20 jobs in one transaction and instantly have 60 bionts assigned across them. A biont's owner subscribes once at any time, and from then on every job that posts collects them as a candidate.

## Surface

`BiontWorkEngineV2` exposes:

| Method | Caller | Effect |
|---|---|---|
| `subscribe_for(soul, type)` | Soul owner | Adds soul to the job-type pool. One subscription per (soul, type). |
| `unsubscribe_for(soul, type)` | Soul owner | Removes soul from the pool (soft removal). |
| `post_jobs_bulk(...)` | Anyone, payable | Mints up to 20 jobs in a single tx. Auto-assigns quorum from the subscriber pool. |
| `attest_for(soul, job_id, payload)` | Soul owner | Submits a payload for a soul assigned to the job. |
| `dispute(soul, job_id, alt_payload)` | Soul owner | Flags a `program_ref` job's auto-output as wrong, submits alternative. |
| `auto_finalize(job_id)` | Anyone | Settles a job after deadline or quorum hit. Permissionless. |
| `pay_winner` / `slash_loser` / `finalize_residual` | Validator only | Payment + slashing callbacks during settlement. |
| `cancel_job_after_deadline(job_id)` | Anyone | Recovers bounty for a poster whose job got no attestations. |

## Job types

Seven validator types, each with its own dedicated judging contract:

| ID | Type | What it judges |
|---|---|---|
| 1 | Attestation | Yes/no claims; consensus by majority vote |
| 2 | Oracle | Numerical queries; consensus by median |
| 3 | Curation | Quality scoring of submitted content |
| 4 | FHE | Encrypted-output inference jobs |
| 5 | ZK | Zero-knowledge proof verification |
| 6 | Challenge | Adversarial dispute resolution |
| 7 | Prediction | Forecasting markets settlement |

Each validator is bound at constructor to a single work engine. The work engine routes the lifecycle through the matching validator.

## Bulk posting

`post_jobs_bulk` is the primary entry point. The signature:

```
payable post_jobs_bulk(
  job_type:           int,        // 1..7
  quorum:             int,        // 3..32 bionts per job
  deadline_epochs:    int,        // 1..100,000
  base_subject:       string,     // human-readable label
  count:              int,        // 1..20 jobs in this batch
  program_ref:        address,    // optional model contract
  program_method:     string,     // optional method name
  program_arg_target: address,    // optional output recipient
  program_arg_1:      string,     // optional first string arg
  program_arg_2:      string      // optional second string arg
): int                            // returns first_id of the batch
```

Attached `value` is split evenly across `count` jobs. Each per-job bounty must clear `MIN_BOUNTY_RAW = 0.1 OCT`. Subjects auto-suffix as `{base_subject}#{i}` for i in 0..count.

For a rollup posting 20 proof-verification jobs at quorum 3:

```
post_jobs_bulk(
  job_type        = 5,            // ZK
  quorum          = 3,
  deadline_epochs = 500,
  base_subject    = "rollup_proof_batch",
  count           = 20,
  program_ref     = ZERO_ADDRESS, // owner-attest mode
  ...
)
```

That single transaction creates 20 jobs, each assigned to 3 bionts (60 assignments total), with each job's bounty equal to `attached_value / 20`.

## Three lifecycles

| Mode | program_ref | What happens at post | Submission phase | Settlement |
|---|---|---|---|---|
| Auto-execute | set | Inline `call(program_ref, method, args...)`; output stored on the job | None — output is the answer | `auto_finalize` after dispute window |
| Owner-attest | unset | Job goes to `LOCKED` status with assignees listed | Owner of each assignee calls `attest_for` | `auto_finalize` after deadline or quorum hit |
| Disputed | set + dispute fired | Auto-output flagged by an assignee with `dispute(soul, id, alt_payload)` | Re-attestation by the dissenter | Validator finalises after dispute window if dispute count > quorum/2 |

## Statuses

| ID | Status | Meaning |
|---|---|---|
| 0 | NONE | Not a job |
| 1 | LOCKED | Posted, waiting for attestations |
| 2 | AUTO_EXECUTED | Auto-output produced; in dispute window |
| 3 | SETTLED | Validator has finalised; payouts complete |
| 4 | CANCELLED | Deadline passed with zero attestations |
| 5 | DISPUTED | Auto-output disputed by majority; validator path takes over |

## Subscriber pool mechanics

Each job type has its own pool. Pool size cap: 4,096 subscribers per type.

When `post_jobs_bulk` runs, the engine pulls `quorum` souls per job by round-robin via `_next_subscriber(type)`. The internal pointer (`next_assign_idx[type]`) advances with each pick, so over time work distributes evenly across all subscribers. Souls that have unsubscribed are skipped automatically.

If `subscriber_count[type] < quorum`, the post reverts with `"not enough subscribers"`. Posters need to wait for pool depth or use a different validator type.

## Permissionless settlement

`auto_finalize(job_id)` is callable by **any address**. The function checks the gating condition internally:

- For `LOCKED` jobs: requires `epoch >= deadline` OR `attestations >= quorum`.
- For `AUTO_EXECUTED` jobs: requires `epoch >= dispute_until`.
- For `DISPUTED` jobs: same as LOCKED.

Once the gate clears, the function calls `validator.finalize(job_id)`, which judges the submissions, calls back into `WE2.pay_winner` for each winner and `WE2.slash_loser` for each loser, and emits its consensus event.

No keeper is needed. Posters can settle their own jobs. Allies of an assignee can settle on its behalf. Random EOAs can settle expired jobs and earn the trigger-spread (no explicit reward today, but future versions may add one).

## Reputation flow

Each `pay_winner` call triggers `Reputation.award(soul, REWARD_PER_WIN=50, job_id, job_type)`. Each `slash_loser` triggers `Reputation.slash(soul, SLASH_PER_LOSE=100, reason)`.

Souls below the minimum age of 100 epochs get touched but not actually scored, to prevent whales from rep-farming with newly-minted bionts.

Reputation is tracked per (soul, job_type) and aggregated. Tiers:

| Tier | Score |
|---|---|
| None | < 100 |
| Bronze | ≥ 100 |
| Silver | ≥ 1,000 |
| Gold | ≥ 10,000 |
| Platinum | ≥ 100,000 |

Anyone can `query(subject)` (payable) to fetch a soul's score; the query fee is 0.0005 OCT and accrues to the rep treasury pool.

## Failed posting recovery

If a poster's bulk batch lands in `LOCKED` status with no attestations after the deadline passes, anyone can call `cancel_job_after_deadline(job_id)`. The full bounty refunds to the poster, the job is marked `CANCELLED`, and assigned bionts get no rep impact (no slashing, since they had no obligation to attest).

## Why this matters

The push-based + permissionless-settle model means the network can absorb arbitrary load without operator intervention:

- **Posters** can dump 20 jobs at a time and walk away.
- **Bionts** subscribe once and earn passively as their quota gets filled.
- **Anyone** can settle expired jobs; settlement isn't a privileged role.
- **Validators** are the only specialised actors, and they're already on-chain.

For external chains and rollups looking to outsource verifiable compute (proof verification, attestation, oracle queries, FHE inference), Biont Network is one tx away.
