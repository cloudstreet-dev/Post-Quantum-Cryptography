# Chapter 4: Code-Based Cryptography

Code-based cryptography is the oldest post-quantum candidate family. Robert McEliece proposed the first code-based public-key encryption scheme in 1978, and it has remained unbroken — classically and quantumly — for nearly fifty years. That longevity is the primary argument in its favour. The primary argument against it is that the public keys are enormous.

## The McEliece cryptosystem

The original McEliece scheme instantiates with binary Goppa codes, which have the property of efficient decoding (there is a known polynomial-time algorithm for correcting up to t errors) despite being indistinguishable from random linear codes to an observer who does not know the code structure.

### Hard problem

The McEliece scheme's security reduces to the hardness of:

1. **Code indistinguishability**: The public key matrix is a random-looking generator matrix. Distinguishing it from a truly random linear code requires recovering the Goppa structure, which is equivalent to solving the code equivalence problem — believed hard for large parameters.

2. **Syndrome decoding**: Given the disguised generator matrix and a ciphertext with intentional errors added, recovering the message requires correcting those errors without knowing the Goppa structure. This is an instance of the syndrome decoding problem, which is NP-complete for random linear codes.

### Key generation

1. Select a t-error-correcting binary [n, k, d≥2t+1] Goppa code with generator matrix **G**
2. Choose a random k×k invertible matrix **S** (scrambling)
3. Choose a random n×n permutation matrix **P**
4. Public key: **G'** = **SGP** (size k×n); Private key: (**S**, **G**, **P**)

The public key is a scrambled, permuted version of the original generator matrix. The Goppa structure is hidden.

### Encryption and decryption

**Encryption** of message **m** ∈ 𝔽_2^k:

1. Compute **c** = **mG'** + **e** where **e** ∈ 𝔽_2^n is a random vector of weight exactly t

**Decryption**:

1. Compute **cP⁻¹** = **mSG** + **eP⁻¹**
2. Apply the Goppa decoder to **cP⁻¹** — it corrects t errors regardless of the scrambling **S**
3. Recover **mS**, then apply **S⁻¹** to get **m**

The decoder works on **mSG** + **eP⁻¹** because **eP⁻¹** still has weight t (permutations preserve weight), and the Goppa decoder corrects any error pattern of weight ≤ t.

### Niederreiter variant

Harald Niederreiter proposed a dual formulation using parity check matrices instead of generator matrices. In the Niederreiter scheme, the public key is a parity check matrix **H'** = **SHP**, and encryption is syndrome-based. The Niederreiter scheme is equivalent in security to McEliece, produces smaller ciphertexts, and is used as the basis for Classic McEliece.

## Classic McEliece

Classic McEliece is the NIST submission that survived to the fourth round without standardisation — NIST ultimately chose not to standardise it in the first wave due to its large public keys, but acknowledged it as the most conservative and best-understood code-based candidate. NIST's security considerations document describes it as a strong candidate for future standardisation.

### Parameters

Classic McEliece uses binary Goppa codes with parameters chosen to resist the best known attacks (primarily information set decoding attacks, with both classical and quantum variants):

| Parameter set | n | k | t | Public key | Private key | Ciphertext |
|---|---|---|---|---|---|---|
| mceliece348864 | 3488 | 2720 | 64 | 261,120 bytes | 6,452 bytes | 128 bytes |
| mceliece460896 | 4608 | 3360 | 96 | 524,160 bytes | 13,568 bytes | 188 bytes |
| mceliece6688128 | 6688 | 5024 | 128 | 1,044,992 bytes | 13,932 bytes | 240 bytes |
| mceliece8192128 | 8192 | 6528 | 128 | 1,357,824 bytes | 14,120 bytes | 240 bytes |

Public keys ranging from 261 KB to 1.3 MB are not suitable for most TLS-style key exchange scenarios. For offline key establishment, pre-provisioned public keys, or systems where bandwidth constraints are secondary to long-term security confidence, Classic McEliece is a defensible choice.

The ciphertexts, by contrast, are tiny — 128 to 240 bytes, smaller than any competing PQC scheme.

### Why the long track record matters

The original McEliece construction was published in 1978. The specific algebraic structure of binary Goppa codes has been the subject of cryptanalytic attention for nearly five decades. Major attack improvements:

- **1988**: Lee-Brickell algorithm — improved information set decoding, practical speedup
- **1988**: Stern's algorithm — probabilistic ISD, asymptotic improvement
- **2008–2011**: BJMM and May-Meurer-Thomae algorithms — improved birthday-based ISD
- **2015–2018**: Various ball-collision decoding improvements

Each of these attacks improved the exponent of the exponential — reducing the effective security level by a constant number of bits — but none found a polynomial-time algorithm. The parameters in Classic McEliece are chosen to provide adequate margin against the best known ISD attacks, including quantum-accelerated variants.

The confidence in McEliece comes from this history: the construction has been publicly analysed by a large portion of the cryptographic community for nearly fifty years, and the best known attack remains exponential.

## Quasi-cyclic code-based schemes: BIKE and HQC

Both BIKE and HQC were NIST alternate candidates (fourth round). They use quasi-cyclic codes to achieve dramatically smaller keys at the cost of a shorter cryptanalytic history and reliance on structured variants of syndrome decoding.

### BIKE (Bit Flipping Key Encapsulation)

BIKE uses quasi-cyclic moderate density parity check (QC-MDPC) codes. The key insight: a random-looking parity check matrix with quasi-cyclic structure enables efficient decoding while appearing indistinguishable from random to an adversary.

**Structure**: The parity check matrix has the form **H** = [H₀ | H₁] where H₀ and H₁ are r×r circulant matrices (each row is a cyclic rotation of the previous). A circulant matrix is fully specified by its first row — storing one row of length r suffices to represent an r×r matrix. This reduces key storage from O(r²) to O(r).

**Decoder**: BIKE uses a bit-flipping decoder (Black-Gray-Flip, a variant of Gallager's original algorithm) rather than an algebraic decoder. The decoder is probabilistic — it fails to decode with small probability δ, which creates a subtle security issue: decryption failure events can leak information about the private key. BIKE's design carefully bounds δ and the information leakage.

**Parameters** (approximate):

| Parameter set | Security | Public key | Secret key | Ciphertext |
|---|---|---|---|---|
| BIKE-L1 | ~128-bit | 1541 bytes | 281 bytes | 1573 bytes |
| BIKE-L3 | ~192-bit | 3083 bytes | 418 bytes | 3115 bytes |
| BIKE-L5 | ~256-bit | 5122 bytes | 580 bytes | 5154 bytes |

### HQC (Hamming Quasi-Cyclic)

HQC is also quasi-cyclic but uses a different algebraic structure and a Reed-Muller decoder for the inner code. The security reduction is cleaner than BIKE's: HQC's security reduces to the hardness of the syndrome decoding problem for quasi-cyclic codes, without the decryption failure complication.

**Parameters** (approximate):

| Parameter set | Security | Public key | Secret key | Ciphertext |
|---|---|---|---|---|
| HQC-128 | ~128-bit | 2249 bytes | 2289 bytes | 4481 bytes |
| HQC-192 | ~192-bit | 4522 bytes | 4562 bytes | 9026 bytes |
| HQC-256 | ~256-bit | 7245 bytes | 7285 bytes | 14,469 bytes |

### The structured code risk

The critical difference between Classic McEliece and the quasi-cyclic schemes is the nature of the hard problem. Classic McEliece rests on syndrome decoding for random linear codes — a problem proven NP-complete in 1978 with fifty years of cryptanalysis. BIKE and HQC rest on syndrome decoding for quasi-cyclic codes, a structured variant whose hardness is not known to reduce to the random case.

Structured problems in cryptography have a mixed historical record. The quasi-cyclic structure creates algebraic relationships that might admit attacks unavailable for unstructured codes. This is not a hypothetical concern — several earlier structured code-based schemes were broken when the structure was successfully exploited.

BIKE and HQC have survived the NIST process without structural breaks, but their cryptanalytic history is measured in years rather than decades.

## Practical recommendation

For most applications, ML-KEM (lattice-based) is the right choice. Code-based cryptography is worth considering for:

1. **Diversity**: An organisation that wants to hedge against a breakthrough in lattice cryptanalysis can deploy Classic McEliece as an alternative or in addition to ML-KEM.
2. **Long-horizon threat models**: Classic McEliece's fifty-year security history is reassuring for applications with multi-decade confidentiality requirements.
3. **Embedded/constrained environments where small ciphertexts matter**: Classic McEliece's ciphertext size is tiny; only the key generation and storage is painful.

For general-purpose network security, the 261 KB–1.3 MB public key is a decisive disadvantage that the lattice-based schemes do not have.
