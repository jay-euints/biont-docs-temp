---
sidebar_position: 3
title: FHE Jobs
---

# FHE Jobs

The Biont work market supports privacy-preserving inference jobs natively. A poster can submit a job whose output is encrypted to their own key — bionts coordinate the work, validators verify it ran, but only the poster can decrypt the answer.

This is the protocol's strongest differentiator and the foundation for any third-party integration that needs verifiable compute over private data.

## What's possible

Octra Network exposes `program_exec` as a first-class transaction type. A program is a deployed contract; a `program_exec` tx invokes a method on it and lets the validator run the code on-chain. Octra also has native FHE primitives: `fhe_load_pk`, `fhe_add`, `fhe_add_const`, `fhe_ser`, `fhe_deser`, `fhe_verify_zero`, `fhe_pedersen`. These are available in AML.

Combined, the chain can:

- Run a contract that performs FHE arithmetic on inputs
- Encrypt the result under a target's public key
- Return the ciphertext as the contract's output
- Have only that target be able to decrypt it

`BiontWorkEngineV2.post_jobs_bulk` lets a poster wire up the program reference, method, and arguments at post time. The work engine inline-executes the call inside the post tx. The output is committed to the job's state. Bionts assigned to the job act as **witnesses** — they confirm the receipt is valid; they do not see the plaintext.

## Anatomy of an FHE job

```
post_jobs_bulk(
  job_type           = 4,                      // FHE
  quorum             = 3,
  deadline_epochs    = 500,
  base_subject       = "private_score",
  count              = 1,
  program_ref        = oct...FHEModelContract,
  program_method     = "complete_private",
  program_arg_target = oct...PosterAddr,       // who decrypts
  program_arg_1      = "100,50,200",           // public input (features CSV)
  program_arg_2      = "<base64 zero_ct>"      // poster's PVAC zero ciphertext
)
```

The work engine runs:

```
output = call(program_ref, program_method, program_arg_target, program_arg_1, program_arg_2)
job_program_output[id] = output
job_status[id] = AUTO_EXECUTED
```

`output` is a base64-encoded PVAC ciphertext. Anyone can read it; only the holder of the PVAC private key for `program_arg_target` can decrypt it.

After the dispute window expires (`DISPUTE_WINDOW_EPOCHS = 1000`), anyone calls `auto_finalize(job_id)`:

- All non-disputing assignees split the bounty equally
- Each non-disputing assignee earns `REWARD_PER_WIN = 50` reputation
- Job status flips to `SETTLED`

## The reference implementation

`FHEStubScorer` is a working example deployed alongside the v2 stack. It implements a public weighted-sum scoring function:

```
score = Σᵢ (featureᵢ × weightᵢ)    // computed homomorphically
```

The contract:

1. Loads the target's PVAC pubkey via `fhe_load_pk(target_addr)`
2. Deserialises the caller's zero ciphertext via `fhe_deser(zero_ct_b64)`
3. For each feature, adds `feature × weight` to the accumulator via `fhe_add_const`
4. Serialises the result via `fhe_ser` and returns the base64 blob
5. Records an audit row: `(committer, target, sha256(features), encrypted_blob)`

Compiled clean on devnet at 3.8 KB / 652 instructions. Uses only the FHE primitives that exist on devnet today.

## Setup for FHE participants

To act as the **target** of an FHE job (i.e. to be able to decrypt outputs), an address must register a PVAC public key on-chain.

```
1. Generate a PVAC keypair via the pvac_hfhe_cpp tool
2. Submit octra_registerPvacPubkey(addr, pubkey_blob, aes_kat_hex)
   — requires positive balance + AES KAT gate
3. Verify with octra_pvacPubkey(addr) — should return non-null
```

Both the poster and any biont that wants to receive private outputs must complete this setup. Once registered, the address can be used as `program_arg_target` in any FHE job.

## Why bionts can't decrypt their own jobs

Bionts are smart contracts. Smart contracts cannot hold private keys. The PVAC scheme requires a private key to decrypt — there is nowhere safe for that key to live inside a contract.

The pragmatic solution: **owners hold the PVAC keypair for their fleet**. When a biont participates in an FHE job, the output is encrypted to the owner's pubkey. The owner decrypts off-chain and shares the result with the biont through normal channels.

This is the same trust model as classical cryptography: the human at the keyboard is the decryption authority. The biont is the computational worker; the human is the keyholder.

## Three flavours of FHE workload

| Workload | Pattern | Posted as |
|---|---|---|
| Encrypted oracle | Public query → private answer | `program_ref = OracleContract`, `program_method = "private_lookup"` |
| Private prediction | Public features → encrypted score | `program_ref = ScorerContract`, args = features CSV |
| Encrypted attestation | Public claim → encrypted yes/no with proof | `program_ref = AttestorContract`, args = claim hash + proof |

Anything that fits the shape `(public_inputs) → encrypt(result, target_pk)` can be a Biont FHE job. The work engine handles assignment, witnessing, settlement, and reward — the model contract handles the cryptography.

## Stealth funding

Posters with significant FHE workload can fund their network usage privately via Octra's stealth-transfer primitive (`op_type = "stealth"`). The receiver's encrypted balance updates atomically via an HFHE delta cipher; the amount stays private on-chain.

This means a third-party rollup running thousands of proof-verification jobs through Biont Network can fund the operation without revealing per-job spend to competitors. The work itself is verifiable; the economics are private.

## Dispute path

If an assigned biont believes the auto-output is wrong (e.g. they re-ran the model and got a different ciphertext), they can call `dispute(soul, job_id, alt_payload)` during the dispute window. If `dispute_count * 2 > quorum`, the job moves to `STATUS_DISPUTED` and a fresh deadline opens. The validator then judges the dispute by comparing alternative payloads — typically by re-running the model itself if it's deterministic, or by appeal to the poster's authority.

For deterministic FHE models (which most are, since FHE arithmetic is reproducible), disputes converge naturally: a dishonest output is provable on-chain by anyone who runs the same `program_exec` and gets a different ciphertext.

## Investor framing

Biont Network is the coordination layer for verifiable encrypted compute. Bionts make FHE jobs **scheduled, discoverable, and economically settled**. The cryptography is Octra's; the market is ours. Third-party chains, rollups, oracles, and ML-inference providers can pipe their privacy-sensitive workloads through Biont Network with a single `post_jobs_bulk` call.

The pitch isn't "FHE happens here." The pitch is "FHE happens at the chain layer, and Biont Network is how it gets paid."
