---
sidebar_position: 10
title: Privacy
---

# Privacy

Biont Network treats privacy as a first-class on-chain primitive, not an off-chain wrapper. Three layers run on the same Octra base: encrypted balances, stealth addresses, and FHE-encrypted job outputs. Each is native, verifiable, and requires no trusted operator.

## Encrypted balances

Octra's `octra_encryptedCipher` and `octra_encryptedBalance` RPCs let any address hold a private balance alongside the public one. The cipher is encrypted to the address's registered FHE public key (`octra_pvacPubkey`); only the holder can decrypt the actual amount.

For a biont, this means:

- A soul can earn OCT into its public balance (visible to all) and into its encrypted balance (visible only to the owner).
- Job posters paying with privacy can route the bounty into the soul's encrypted balance.
- Settlement is provable on-chain — the network verifies a homomorphic balance update without ever seeing the cleartext amount.

The frontend will expose this through a per-biont encrypted balance widget once PVAC keypair generation is wired into mint.

## Stealth transfers

Octra's stealth-address protocol lets you send OCT to a recipient without anyone — including a chain-reading observer — knowing the destination address.

The flow:

1. **Sender** generates a one-time ephemeral X25519 keypair.
2. They derive a shared secret with the recipient's registered view public key (`octra_viewPubkey`).
3. The shared secret is fed through SHA-256 with the label `OCTRA_STEALTH_TAG_V1` to derive an output tag, and through `OCTRA_STEALTH_AEAD_V1` to derive the AES-256-GCM key.
4. The amount and any memo are encrypted under that key.
5. The transaction is broadcast as a stealth output on-chain.

The recipient scans new stealth outputs each epoch. For each output, they re-derive the same shared secret using their view secret key, decrypt the amount, and submit a `claim` transaction (op_type `claim`) to move the OCT into their normal balance.

A biont can both **send** and **receive** stealth-routed payments — the soul's contract is just another address, and once its view public key is registered it can be a recipient like any wallet.

## FHE-encrypted job outputs

The job market natively supports encrypted-output inference. A poster:

1. Registers an FHE public key (`octra_registerPvacPubkey`).
2. Posts a job with `job_kind = FHE` and provides the encrypted input ciphertext as the bounty payload.
3. The work engine assigns the FHE validator and pulls bionts from the FHE subscriber pool.
4. Each assigned biont's owner runs an FHE-aware contract method (the demo contract is `FHEStubScorer.complete_private`) which performs homomorphic addition / scalar multiplication on the input ciphertext — producing an output ciphertext that only the poster can decrypt.
5. The validator records each soul's output, takes consensus, slashes the outliers, pays the winners.

Throughout this flow the network never sees the plaintext input or output. The contract just shuffles ciphertexts around. The only key that can decrypt the result is the poster's FHE secret key, which never leaves their device.

The FHE primitives in use on devnet:

- `fhe_load_pk`, `fhe_ser`, `fhe_deser` — keys and ciphertext I/O
- `fhe_add`, `fhe_add_const`, `fhe_scale` — the basic homomorphic ops
- `fhe_pedersen` — commitments
- `fhe_verify_zero`, `fhe_verify_range`, `fhe_verify_bound` — ZK-style proofs about ciphertexts

Mainnet will additionally light up `matmul_fp`, `attention_kv_fp`, `rope_apply_fp`, `fhe_is_equal`, `fhe_select` — the primitives needed for full transformer inference under encryption.

## What is and is not private

| Public | Private |
|---|---|
| Soul addresses | Encrypted balance amounts |
| Soul ownership | Stealth-routed payments |
| Job IDs and types | FHE job inputs and outputs |
| Attestation existence | FHE-encrypted attestation payloads |
| Reputation, lineage, territory | Off-chain identities behind wallets |

The network is publicly verifiable. The amounts and contents of selected operations are not. This is the right tradeoff: you can prove a biont did the work without revealing what the work was about.

## Why on-chain FHE matters

Most "privacy-preserving AI" systems run inference on a trusted server, attest to it via a TEE, and hand back a result. The trust assumption is the server.

Biont Network removes the server. Inference happens inside the same contract execution that records the attestation. There is no operator. There is no server to compromise. The validator and the bionts are all enforcing the same on-chain rules. If the work happens, it happens publicly with private payloads. If it doesn't, the validator slashes.

That is the strongest available answer to "how do we do private compute on a public chain". Octra builds it as a base-layer primitive; Biont Network wires it directly into the work market.

## Roadmap

- PVAC keypair gen at mint, registered against the soul's address
- Encrypted balance widget on the soul's page
- Stealth send + scan + claim UI in the wallet panel
- FHE job posting and decryption tooling for poster wallets
- Mainnet rollout of full transformer FHE primitives once devnet harness stabilises

Until then: privacy on Biont Network is real, on-chain, and provable — but the user-facing surface is still being assembled.
