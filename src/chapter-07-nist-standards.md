# Chapter 7: The NIST PQC Standards

In August 2024, NIST published three Federal Information Processing Standards for post-quantum cryptography. A fourth is in final stages. This chapter covers what was standardised, why, and what the standards actually specify.

## The standardisation process

NIST announced its Post-Quantum Cryptography Standardisation Project in December 2016, calling for nominations of public-key cryptographic algorithms resistant to attacks by both classical and quantum computers. The submission deadline was November 2017; 69 complete submissions were received.

The process ran in four rounds:

- **Round 1** (2017–2019): 69 submissions evaluated; 26 advanced
- **Round 2** (2019–2020): 26 candidates; 15 advanced (7 finalists, 8 alternates)
- **Round 3** (2020–2022): 4 KEM/encryption finalists, 3 signature finalists, 5 alternates
- **Round 4** (2022–2024): Additional analysis; BIKE and HQC retained for further study

NIST selected for standardisation:
- CRYSTALS-Kyber → **ML-KEM** (FIPS 203)
- CRYSTALS-Dilithium → **ML-DSA** (FIPS 204)
- FALCON → **FN-DSA** (FIPS 206, draft)
- SPHINCS+ → **SLH-DSA** (FIPS 205)

Several candidates were broken during the process — most notably SIKE (isogeny-based, broken in 2022 using a classical 62-minute attack) and Rainbow (broken 2022, as discussed in Chapter 6). The survivors were subjected to eight years of public cryptanalysis.

## ML-KEM (FIPS 203)

**Full name**: Module-Lattice-Based Key-Encapsulation Mechanism Standard
**Based on**: CRYSTALS-Kyber
**Function**: Key Encapsulation Mechanism (KEM)

### What FIPS 203 specifies

FIPS 203 specifies the complete ML-KEM algorithm including:

- Three parameter sets: ML-KEM-512, ML-KEM-768, ML-KEM-1024
- Exact specification of the CRYSTALS-Kyber algorithm with minor modifications from the submission for clarity and consistency
- The Number Theoretic Transform (NTT) implementation requirements
- Deterministic key generation using seeded PRG (allowing reproducible key pairs)
- Hash and PRF function usage (SHA3-256, SHA3-512, SHAKE-128, SHAKE-256)
- The Fujisaki-Okamoto transform providing CCA2 security

**Minimum recommendation**: ML-KEM-768 for general use, ML-KEM-1024 for applications with higher security requirements. ML-KEM-512 is retained for constrained environments where 128-bit security is adequate.

### What ML-KEM is and is not

ML-KEM is a **key encapsulation mechanism**, not a general-purpose public-key encryption scheme. It generates a shared secret; applications use that shared secret as a key derivation input. It does not support encryption of arbitrary messages directly.

This is intentional and correct. KEM + symmetric encryption is the proper composition. Applications that want to send an encrypted message should use ML-KEM to establish a shared key, then encrypt with AES-GCM or ChaCha20-Poly1305.

## ML-DSA (FIPS 204)

**Full name**: Module-Lattice-Based Digital Signature Standard
**Based on**: CRYSTALS-Dilithium
**Function**: Digital signatures

### What FIPS 204 specifies

FIPS 204 specifies:

- Three parameter sets: ML-DSA-44, ML-DSA-65, ML-DSA-87
- Complete Dilithium algorithm with modifications: the signing procedure is now fully deterministic (message and private key uniquely determine the signature; no per-signature randomness is required, though a hedged variant is also defined)
- The Fiat-Shamir with aborts protocol
- Hash algorithm requirements (SHAKE-128, SHAKE-256)
- Message hashing procedure using a context string to support domain separation

**Minimum recommendation**: ML-DSA-65 for general use (192-bit security level). ML-DSA-44 for applications tolerant of 128-bit post-quantum security.

### Deterministic vs. hedged signing

The base ML-DSA signing algorithm is deterministic: given the same private key and message, it always produces the same signature. This eliminates the risk of catastrophic failure from a weak random number generator (a real vulnerability in ECDSA — the Sony PlayStation 3 private key was extracted due to a fixed nonce in deterministic signing that was accidentally made non-deterministic).

FIPS 204 also defines a hedged variant that mixes in additional randomness: signature = Sign(sk, message, rnd) where rnd ∈ {0,1}^256 can be random bytes. The hedged variant adds protection against fault attacks and side-channel attacks at the cost of requiring a CSPRNG.

## SLH-DSA (FIPS 205)

**Full name**: Stateless Hash-Based Digital Signature Standard
**Based on**: SPHINCS+
**Function**: Digital signatures

### What FIPS 205 specifies

FIPS 205 specifies:

- Twelve parameter sets across two hash function families (SHA-2 and SHAKE) and two performance profiles (small signatures -s, fast signing -f) at three security levels
- The complete WOTS+, FORS, and hypertree algorithms
- Randomised and deterministic signing modes
- Pure and pre-hash variants (supporting different message hashing workflows)

**Minimum recommendation**: SLH-DSA-SHA2-128s or SLH-DSA-SHAKE-128s for 128-bit security applications where signature size is acceptable. Use as a diversification hedge alongside ML-DSA, not necessarily as a primary replacement.

### The pure vs. pre-hash distinction

FIPS 205 defines two signing interfaces:

- **Pure SLH-DSA**: The full message is passed to the signing algorithm, which internally hashes it. The signature commits to the full message.
- **HashSLH-DSA**: An approved hash function is applied to the message first; the hash is passed to the signing algorithm. This supports streaming and situations where the full message is unavailable to the signing primitive.

The two interfaces are not interoperable — a HashSLH-DSA signature verifies differently from a pure SLH-DSA signature over the same message. Applications must agree on which interface they use.

## FN-DSA (FIPS 206)

**Full name**: FALCON-Based Digital Signature Standard (draft)
**Based on**: FALCON
**Function**: Digital signatures

FIPS 206 was published as a draft in 2024 and is expected to be finalised in 2025. The content is substantively stable; the draft status reflects ongoing editorial review.

### What FIPS 206 specifies

- Two parameter sets: FN-DSA-512 (Falcon-512, ~128-bit) and FN-DSA-1024 (Falcon-1024, ~256-bit)
- The NTRU key generation procedure and the specific recursive lattice algorithm
- The Fast Fourier sampling algorithm with explicit requirements on the floating-point implementation
- Signature encoding and compression

**On implementation**: FIPS 206 is explicit that implementers must follow the reference implementation's Gaussian sampler specification. The standard includes normative language on the precision of floating-point operations. This is unusual for a FIPS standard and reflects the genuine difficulty of implementing the sampler correctly.

## What was not standardised and why

**SIKE**: Broken. A classical attack by Castryck and Decru (2022) recovered the private key from the public key in 62 minutes using algebraic geometry. Isogeny-based cryptography remains an active research area, but SIKE specifically is dead.

**Rainbow**: Broken. See Chapter 6.

**NTRU**: NTRU (the original patent-encumbered scheme) and NTRU Prime were not selected. NTRU's parameter choices have been a recurring source of cryptanalytic attention (the subfield attack, the hybrid attack), and NIST preferred the cleaner security reduction of FALCON's NTRU lattice variant as standardised in FN-DSA.

**Classic McEliece**: Not in the initial standards due to large public keys, but NIST's report notes it as the most conservative candidate with the strongest security argument. Additional standardisation is possible in future rounds.

**BIKE and HQC**: Retained for study in a separate NIST process. Not in the 2024 standards, but neither is broken.

## Using the standards together

The NIST guidance is:

1. For key establishment: use **ML-KEM**. A hybrid construction pairing ML-KEM with X25519 (classical ECDH) is recommended during the transition period (see Chapter 8).

2. For signatures: use **ML-DSA** as the primary scheme. Consider **SLH-DSA** as a diversified backup if the application's security model calls for hedging against lattice cryptanalysis. Use **FN-DSA** when signature size is a hard constraint and the implementation team can handle it correctly.

3. For symmetric operations: use **AES-256** and **SHA-384/SHA-512** to maintain 192-bit+ post-quantum security. AES-128 drops to ~64-bit quantum security under Grover's algorithm.

The standards are available from NIST at no cost. Reading the actual specifications is recommended — they are well-written and more accessible than most cryptographic standards.
