# Chapter 3: Lattice-Based Cryptography

Lattice-based cryptography produces the largest family of post-quantum algorithms and accounts for three of the four NIST standards. The mathematical foundations were covered in Chapter 2. This chapter covers the constructions built on those foundations, focusing on the three algorithms that achieved standardisation: CRYSTALS-Kyber (now ML-KEM), CRYSTALS-Dilithium (now ML-DSA), and FALCON (now FN-DSA).

## From LWE to encryption: the basic construction

The simplest LWE-based encryption scheme illustrates how the problem translates into cryptography. Let n be the security parameter, q a modulus, and χ an error distribution.

**Key generation**: Sample **s** ←$ ℤ_q^n (private key), **A** ←$ ℤ_q^(m×n), **e** ←$ χ^m, compute **b** = **As** + **e** mod q. Public key is (**A**, **b**).

**Encryption** of bit μ ∈ {0,1}: Sample **r** ←$ {0,1}^m (sparse), compute **u** = **Aᵀr** mod q, v = **bᵀr** + μ·⌊q/2⌋ mod q. Ciphertext is (**u**, v).

**Decryption**: Compute v − **sᵀu** = **bᵀr** − **sᵀ(Aᵀr)** + μ·⌊q/2⌋ = (**As** + **e**)ᵀ**r** − **sᵀAᵀr** + μ·⌊q/2⌋ = **eᵀr** + μ·⌊q/2⌋. If **eᵀr** is small (which it is, since both **e** and **r** are small), round to recover μ.

This basic construction is IND-CPA secure under the decisional LWE assumption. Key sizes and ciphertext sizes scale as O(n²) due to the matrix **A**. Ring-LWE and Module-LWE reduce this to O(n) and O(kn) respectively, where k is the module rank.

## CRYSTALS-Kyber / ML-KEM

Kyber is a key encapsulation mechanism (KEM) based on Module Learning With Errors (MLWE). It was selected by NIST and standardised as ML-KEM (FIPS 203) in 2024.

### Structure

Working over the polynomial ring R_q = ℤ_q[x] / (x^256 + 1) with q = 3329, Kyber uses a module of rank k. Three parameter sets are defined:

| Parameter set | k | Security level | Public key | Secret key | Ciphertext |
|---|---|---|---|---|---|
| ML-KEM-512 | 2 | ~128-bit | 800 bytes | 1632 bytes | 768 bytes |
| ML-KEM-768 | 3 | ~192-bit | 1184 bytes | 2400 bytes | 1088 bytes |
| ML-KEM-1024 | 4 | ~256-bit | 1568 bytes | 3168 bytes | 1568 bytes |

For comparison, an X25519 public key is 32 bytes and a Diffie-Hellman shared secret exchange produces a 32-byte result. The size increase is the cost of lattice-based security.

### Key generation

1. Sample matrix **A** ←$ R_q^(k×k) from a hash of a random seed ρ (the matrix is derived pseudorandomly, not transmitted in full)
2. Sample secret **s** ←$ χ^k and error **e** ←$ χ^k (small coefficients drawn from a binomial distribution)
3. Compute **t** = **As** + **e**
4. Public key: (ρ, **t**); Private key: **s**

### Encapsulation

1. Sample message m ∈ {0,1}^256
2. Derive randomness from m and the public key hash (making it deterministic given m)
3. Sample **r**, **e₁**, e₂ from χ
4. Compute **u** = **Aᵀr** + **e₁**, v = **tᵀr** + e₂ + ⌈q/2⌉·m
5. Ciphertext: (**u**, v); Shared secret: KDF(m, H(ciphertext))

### Decapsulation

1. Compute m' = Decompress(v − **sᵀu**)
2. Re-encapsulate with m' to produce ciphertext (**u**', v')
3. If (**u**', v') = (**u**, v), output the shared secret; otherwise output a pseudorandom value derived from a rejection token

The implicit rejection in step 3 is critical for CCA2 security (the Fujisaki-Okamoto transform applied to the base scheme). An active adversary who submits a malformed ciphertext receives a pseudorandom output indistinguishable from a legitimate shared secret, learning nothing.

### Number Theoretic Transform

The dominant cost in Kyber is polynomial multiplication in R_q. The NTT is the efficient algorithm: since q = 3329 and n = 256, q ≡ 1 (mod 512), so 512th roots of unity exist in ℤ_q, enabling a length-256 NTT. Polynomial multiplication reduces to point-wise multiplication in the transform domain in O(n log n) operations, versus O(n²) for schoolbook multiplication.

This is the reason behind the specific value q = 3329: it is the smallest prime satisfying q ≡ 1 (mod 512) that provides adequate security margin.

## CRYSTALS-Dilithium / ML-DSA

Dilithium is a digital signature scheme standardised as ML-DSA (FIPS 204). It uses the same MLWE foundation as Kyber, plus the Module-SIS problem for the signing security argument.

### Fiat-Shamir with aborts

Dilithium uses a Fiat-Shamir identification scheme converted to a signature by the Fiat-Shamir transform. The "with aborts" technique is essential for lattice-based Fiat-Shamir — unlike classical schemes, the signer must sometimes restart the signing process to avoid leaking information about the private key through the distribution of signature components.

The signing algorithm:

1. Sample masking vector **y** ←$ S_γ₁^ℓ (uniform over small coefficients)
2. Compute **w** = **Ay** mod q, let **w₁** = HighBits(**w**, 2γ₂)
3. Compute challenge c ← H(μ, **w₁**) where μ is the message hash
4. Compute **z** = **y** + c**s₁**
5. If ‖**z**‖_∞ ≥ γ₁ − β or LowBits(**Ay** − c**s₂**, 2γ₂) is large: **abort and restart**
6. Output signature (**z**, c, hint)

The abort condition ensures that **z** reveals no information about **s₁** — the masking **y** completely hides **s₁·c** only when **z** falls within the safe range. The abort probability is tuned to be low (around 1/7 for ML-DSA-65) while ensuring the distribution of accepted signatures leaks nothing.

### Parameter sets

| Parameter set | Security level | Public key | Private key | Signature |
|---|---|---|---|---|
| ML-DSA-44 | ~128-bit | 1312 bytes | 2528 bytes | 2420 bytes |
| ML-DSA-65 | ~192-bit | 1952 bytes | 4000 bytes | 3293 bytes |
| ML-DSA-87 | ~256-bit | 2592 bytes | 4864 bytes | 4595 bytes |

For context, an Ed25519 signature is 64 bytes and a public key is 32 bytes. Lattice-based signatures are larger; the tradeoff is post-quantum security.

## FALCON / FN-DSA

FALCON is a lattice-based signature scheme standardised as FN-DSA (FIPS 206). It uses NTRU lattices — a different lattice structure from Dilithium — and produces significantly smaller signatures at the cost of substantially more complex implementation.

### NTRU lattices

An NTRU lattice is defined by polynomials f, g ∈ R = ℤ[x]/(xⁿ + 1) where f is invertible mod q. The lattice has a particularly useful structure: the Gram matrix has block form with efficient representation. FALCON uses this structure to sample short lattice vectors during signing via the Fast Fourier sampling algorithm — a randomised algorithm that produces vectors whose distribution is an approximation of a discrete Gaussian over the lattice.

### The Fast Fourier Sampling trapdoor

The private key in FALCON is a "good" basis for the NTRU lattice — specifically the pair (f, g) and their NTRU-conjugates F, G satisfying fG − gF = q (the NTRU equation). Key generation finds such a pair using a recursive structure over towers of rings.

Signing reduces to finding a short lattice vector close to a target determined by the message hash. The Fast Fourier Sampler uses the private basis to sample from a discrete Gaussian distribution centred near that target. An adversary with only the public key — a single polynomial h = gf⁻¹ mod q, the "bad" basis — cannot perform this sampling efficiently.

### Parameter sets

| Parameter set | Security level | Public key | Private key | Signature |
|---|---|---|---|---|
| FN-DSA-512 | ~128-bit | 897 bytes | 1281 bytes | ~666 bytes (variable) |
| FN-DSA-1024 | ~256-bit | 1793 bytes | 2305 bytes | ~1280 bytes (variable) |

FALCON signatures are variable-length because the discrete Gaussian output is compressed before transmission. The average sizes quoted are typical. The signature is substantially smaller than ML-DSA for equivalent security levels — the penalty is implementation complexity.

### Implementation hazard: the Gaussian sampler

FALCON's correctness and security depend critically on the quality of the discrete Gaussian sampler. A biased sampler can leak private key information through signature distributions. The reference implementation uses a carefully designed sampler with rejection sampling and polynomial arithmetic over ℂ encoded as 64-bit floats, which introduces floating-point correctness requirements that are non-trivial on constrained hardware.

The NIST standard (FIPS 206) is explicit about this: implementers must use the reference sampler or an implementation proven to be equivalent. Rolling your own Gaussian sampler is a category of mistake with a poor historical record.

## Comparative summary

| Property | ML-KEM | ML-DSA | FN-DSA |
|---|---|---|---|
| Function | KEM | Signatures | Signatures |
| Hard problem | MLWE | MLWE + MSIS | NTRU + SIS |
| Signature size | N/A | ~2.4–4.6 KB | ~0.7–1.3 KB |
| Implementation complexity | Moderate | Moderate | High |
| Constant-time implementation | Straightforward | Straightforward | Requires care |
| NTT-friendly | Yes | Yes | Yes (sort of) |

ML-KEM and ML-DSA are the pragmatic defaults. FN-DSA is for applications where signature size is a binding constraint and the implementer has sufficient expertise to handle the Gaussian sampler correctly.
