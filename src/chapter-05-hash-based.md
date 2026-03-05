# Chapter 5: Hash-Based Signatures

Hash-based signatures derive their security from a single assumption: the collision resistance and preimage resistance of a cryptographic hash function. No algebraic structure, no lattice problems, no integer factorisation. If SHA-256 or SHAKE-256 is secure, hash-based signatures are secure. This makes their security argument unusually clean.

The tradeoff is that they are signature-only — hash functions alone do not readily yield key encapsulation mechanisms. The other tradeoff, historically, was statefulness: many early hash-based schemes required the signer to track which one-time keys had been used. SLH-DSA (SPHINCS+) eliminates that requirement.

## One-time signatures: Lamport and Winternitz

The foundation of all hash-based signature schemes is the one-time signature (OTS). As the name suggests, an OTS key pair can sign exactly one message. Signing a second message with the same key reveals enough information to forge signatures.

### Lamport signatures

Leslie Lamport's scheme from 1979 is the simplest possible hash-based signature:

**Key generation**: For an n-bit message hash, generate 2n random values: x[0][0], x[0][1], ..., x[n-1][0], x[n-1][1]. Compute the public key y[i][b] = H(x[i][b]) for all i, b.

**Signing** message m (after hashing to n bits): For each bit position i, reveal x[i][m_i] — that is, reveal the preimage corresponding to the bit value in the message.

**Verification**: Check H(σ[i]) = y[i][m_i] for all i.

Security: revealing x[i][0] to sign a 0-bit at position i means the adversary cannot forge a 1-bit at position i without inverting H. Correct, but using 2n random values for an n-bit message means a 256-bit hash requires 512 secret values. Signatures are large.

### Winternitz OTS (W-OTS)

W-OTS reduces signature size by applying the hash function multiple times in a chain. Instead of signing individual bits, W-OTS signs w-bit blocks using hash chains of length 2^w − 1.

For a block value b ∈ [0, 2^w), the chain is: x → H(x) → H(H(x)) → ... (applied 2^w − 1 − b times from the secret key to produce the public key component, and applied b times during signing to produce the signature component).

Larger w means shorter signatures (fewer chains needed) but higher computational cost (longer chains). XMSS uses W-OTS+, a hardened variant; SLH-DSA uses WOTS+ with small modifications.

## Merkle trees: from one-time to many-time

A one-time signature is useless for real applications. The standard solution is a Merkle tree.

**Construction**: Generate a set of L OTS key pairs (sk₁, pk₁), ..., (sk_L, pk_L). Build a binary hash tree over the public keys: leaf nodes are H(pk_i), internal nodes are H(left child || right child). The root is the public key of the many-time scheme.

**Signing with leaf i**: Include the OTS signature under sk_i, the OTS public key pk_i, and the authentication path — the sibling hashes on the path from leaf i to the root. The verifier hashes pk_i up to the root using the authentication path and checks it matches the public root.

**Statefulness**: The signer must track which leaves have been used. Using leaf i twice exposes the OTS private key sk_i, enabling forgery. Tracking a counter across process restarts, migrations, and distributed deployments is operationally non-trivial.

XMSS (eXtended Merkle Signature Scheme) and LMS (Leighton-Micali Signatures) are stateful hash-based schemes standardised by NIST and the IETF. They are appropriate for controlled environments where state management is feasible — firmware signing, certificate authorities, HSM-backed key usage.

## SPHINCS+ and SLH-DSA

SPHINCS+ eliminates the statefulness requirement by using a hypertree — a tree of trees — combined with a FORS (Forest Of Random Subsets) few-time signature scheme. NIST standardised SPHINCS+ as SLH-DSA (FIPS 205).

### Architecture

SLH-DSA uses three components:

1. **WOTS+**: A Winternitz one-time signature scheme with domain separation and bitmask operations that strengthen the security proof.

2. **FORS**: A few-time signature scheme based on random subsets. FORS can safely sign a small number of messages with the same key (the exact count depends on parameters), which breaks the pure OTS statefulness requirement.

3. **Hypertree**: A d-layer binary tree where each internal node is signed by a WOTS+ key in the layer above. The leaves are FORS key pairs. The root is the public key.

**Signing**: To sign message m:

1. Derive a pseudorandom index from m and a secret random value R. This index determines which FORS key pair to use and which hypertree leaf to use — deterministically, without a counter.
2. Sign m using the selected FORS key pair
3. Sign the FORS public key using the WOTS+ key at the corresponding hypertree leaf
4. Produce WOTS+ signatures up each layer of the hypertree to the root

The critical property: the leaf index is derived pseudorandomly from (m, R) where R is part of the private key. The same (m, R) always produces the same index, but different messages produce different (pseudorandom) indices. The probability of two distinct messages hitting the same FORS leaf is negligible under the parameter settings — the hypertree is large enough that collision probability is below the target security level.

### Parameter sets

SLH-DSA offers two hash function families (SHA-256 and SHAKE-256) at each security level, and two variants per level: **-s** (small, prioritising smaller signatures at the cost of slower signing) and **-f** (fast, prioritising faster signing at the cost of larger signatures).

| Parameter set | Security | Public key | Private key | Signature | Sign time |
|---|---|---|---|---|---|
| SLH-DSA-SHA2-128s | ~128-bit | 32 bytes | 64 bytes | 7,856 bytes | slow |
| SLH-DSA-SHA2-128f | ~128-bit | 32 bytes | 64 bytes | 17,088 bytes | fast |
| SLH-DSA-SHA2-192s | ~192-bit | 48 bytes | 96 bytes | 16,224 bytes | slow |
| SLH-DSA-SHA2-192f | ~192-bit | 48 bytes | 96 bytes | 35,664 bytes | fast |
| SLH-DSA-SHA2-256s | ~256-bit | 64 bytes | 128 bytes | 29,792 bytes | slow |
| SLH-DSA-SHA2-256f | ~256-bit | 64 bytes | 128 bytes | 49,856 bytes | fast |

The public and private keys are tiny. The signatures are large — substantially larger than ML-DSA and orders of magnitude larger than Ed25519. The -s variants produce smaller signatures but signing can take tens of milliseconds on modern hardware. The -f variants sign in under a millisecond but signatures exceed 17 KB.

### Verification performance

Verification is fast for all parameter sets — comparable to Ed25519 in the 128-bit case. The verification algorithm processes a fixed number of hash evaluations regardless of variant. Signing is the expensive operation.

### Security confidence

SLH-DSA's security reduces entirely to the properties of the underlying hash function. No lattice assumptions, no code assumptions, no assumption about algebraic structure. If SHA-256 or SHAKE-256 is modelled as a random oracle and has 256-bit preimage resistance, SLH-DSA provides the stated security level.

This makes SLH-DSA the most conservative signature choice in the NIST portfolio. It is recommended as a secondary signature option in the NIST migration guidance — not the default (due to signature size) but a hedge against advances in lattice cryptanalysis.

## When to use hash-based signatures

**Use stateful hash-based signatures (XMSS, LMS) when:**
- Signing is done in a controlled, audited environment (firmware signing, CA operations)
- State management is guaranteed by HSM or hardware
- Very long-term security confidence is required
- Signature size is acceptable

**Use SLH-DSA when:**
- Statefulness is not acceptable
- Maximum security confidence is required regardless of signature size
- Verification performance matters more than signing performance
- Serving as a hedge alongside ML-DSA

**Do not use hash-based signatures when:**
- Signature size is a hard constraint (TLS handshakes, packed protocols)
- Signing frequency is high and latency is critical (the -s variants are slow)

In practice, most systems will deploy ML-DSA as the primary signature scheme and optionally SLH-DSA as a backup, depending on how conservatively they want to hedge their lattice assumptions.
