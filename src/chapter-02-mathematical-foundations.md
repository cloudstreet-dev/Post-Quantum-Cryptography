# Chapter 2: Mathematical Foundations

Post-quantum cryptographic constructions rest on a small set of mathematical problems whose hardness survives quantum computation. Understanding those problems at a working level — not a research level, but enough to evaluate security arguments and read the literature without confusion — requires covering lattices, error-correcting codes, and the specific computational problems derived from each.

This chapter covers the foundations. Later chapters apply them.

## Lattices

A lattice is the set of all integer linear combinations of a set of basis vectors. Given linearly independent vectors **b**₁, **b**₂, ..., **b**_n ∈ ℝ^m, the lattice they generate is:

$$\mathcal{L}(\mathbf{B}) = \left\{ \sum_{i=1}^{n} x_i \mathbf{b}_i \;\middle|\; x_i \in \mathbb{Z} \right\}$$

The matrix **B** = [**b**₁ | **b**₂ | ... | **b**_n] is called a basis. A lattice has infinitely many valid bases — any basis obtained by applying an integer unimodular transformation (determinant ±1) generates the same lattice.

This non-uniqueness of bases is the structural feature that cryptographic constructions exploit. A "good" basis (short, nearly orthogonal vectors) makes hard lattice problems easy to solve. A "bad" basis (long, highly correlated vectors) makes those same problems computationally intractable. A trapdoor in lattice cryptography is typically knowledge of a good basis for a lattice that is publicly specified by a bad one.

### Hard lattice problems

**Shortest Vector Problem (SVP)**: Given a lattice basis **B**, find the shortest non-zero vector in ℒ(**B**).

SVP is NP-hard under randomised reductions. The best known classical algorithms for exact SVP run in 2^O(n) time and space. The best known quantum algorithms offer a constant factor improvement but not an asymptotic one — the exponent remains exponential in the dimension n.

**Closest Vector Problem (CVP)**: Given a lattice basis **B** and a target vector **t**, find the lattice vector closest to **t**.

CVP is at least as hard as SVP and is also NP-hard. Most cryptographic constructions reduce to approximate variants of SVP or CVP, where the approximation factor is a polynomial in n — these approximate versions are not known to be NP-hard but are conjectured to require 2^Ω(n) time.

### Learning With Errors (LWE)

LWE is the workhorse problem of lattice-based cryptography. It was introduced by Oded Regev in 2005, along with a proof that solving it (on average, over random instances) is at least as hard as solving approximate SVP in the worst case. This worst-case to average-case reduction is an unusually strong security guarantee — it means breaking an LWE-based scheme on typical instances is as hard as solving the hardest possible lattice instances.

**The LWE problem**: Let n be the dimension, q be a modulus, and χ be an error distribution (typically a discrete Gaussian over ℤ). The decisional LWE problem asks to distinguish the following two distributions:

- **LWE samples**: Choose **s** ∈ ℤ_q^n uniformly at random (the secret). For each sample, choose **a** ∈ ℤ_q^n uniformly and e ∈ χ, and output (**a**, **a**·**s** + e mod q).
- **Uniform samples**: Choose (**a**, b) ∈ ℤ_q^n × ℤ_q uniformly at random.

The error term e is the essential ingredient. Without it, recovering **s** from (**a**, b = **a**·**s** mod q) is a linear algebra problem solvable in polynomial time. With a small error added, recovering **s** or even distinguishing LWE samples from uniform appears exponentially hard.

The search variant asks to recover **s** given polynomially many LWE samples. Under mild parameter conditions, search and decisional LWE are equivalent.

**Ring-LWE (RLWE)**: LWE requires key material proportional to n² (an n×n matrix **A** is part of the public key). Ring-LWE improves efficiency by working in the polynomial ring R_q = ℤ_q[x] / (xⁿ + 1), where n is a power of 2. Multiplication in this ring is efficiently computable via the Number Theoretic Transform (NTT), which is the modular analogue of the Fast Fourier Transform. RLWE achieves essentially the same security guarantees with O(n log n) operations instead of O(n²), and key sizes that scale as O(n) instead of O(n²).

**Module-LWE (MLWE)**: A middle ground between LWE and RLWE. The secret and error are vectors of ring elements — small matrices over R_q rather than the full n×n matrix of plain LWE. CRYSTALS-Kyber and CRYSTALS-Dilithium use MLWE and its signature analogue, Module Learning With Errors over rings. The module structure provides parameter flexibility: the module rank k trades off between security and performance.

## Error-Correcting Codes

An [n, k, d] linear code over a finite field 𝔽_q is a k-dimensional subspace of 𝔽_q^n, where d is the minimum Hamming distance between any two distinct codewords. The rate is k/n (the fraction of bits that carry information), and the code can correct up to ⌊(d−1)/2⌋ errors.

Codes are defined by a generator matrix **G** ∈ 𝔽_q^(k×n) (encoding: **m** → **mG**) and a parity check matrix **H** ∈ 𝔽_q^((n−k)×n) (syndrome: **Hc**ᵀ = **0** for valid codewords). The syndrome **Hs**ᵀ of a received word **s** = **c** + **e** (codeword plus error) equals **He**ᵀ, which is used to locate and correct errors.

### The hard problem: syndrome decoding

**Syndrome Decoding Problem (SDP)**: Given a parity check matrix **H** and a syndrome **s**, find a low-weight error vector **e** such that **He**ᵀ = **s**.

For a random linear code, SDP is NP-complete. This has been known since 1978 (Berlekamp, McEliece, van Tilborg). The best classical algorithms for SDP use information set decoding (ISD), and their complexity is exponential in the code's error-correction capacity. Quantum algorithms improve the exponent by a constant factor via Grover-style search over ISD, but the speedup is sub-exponential — parameters can be increased to compensate.

The key point: unlike lattice problems, the hardness of syndrome decoding for random linear codes does not require a worst-case to average-case reduction argument. Random linear codes are provably hard on average, almost directly from the NP-hardness result.

### Specific code families

**Goppa codes** (McEliece scheme): Binary Goppa codes have efficient decoding algorithms despite being drawn from a family that "looks" random. The public key is a randomly scrambled and permuted generator matrix; the trapdoor is the Goppa structure that enables efficient decoding. The original McEliece scheme from 1978 remains unbroken against both classical and quantum adversaries, though its large key sizes make it unwieldy.

**Quasi-cyclic codes** (BIKE, HQC): Applying cyclic or quasi-cyclic structure reduces key sizes substantially, at the cost of relying on the hardness of structured variants of syndrome decoding. These structured variants lack the same long history of cryptanalysis as random linear codes.

## Hash Functions as Cryptographic Primitives

Hash functions require less mathematical setup — the hard problem is empirical rather than reducible to a clean worst-case problem. A collision-resistant hash function H: {0,1}* → {0,1}^n is one for which:

- **Preimage resistance**: Given y, finding x such that H(x) = y costs O(2^n) time.
- **Second preimage resistance**: Given x, finding x' ≠ x such that H(x) = H(x') costs O(2^n) time.
- **Collision resistance**: Finding any pair (x, x') with H(x) = H(x') costs O(2^(n/2)) time (by the birthday bound).

Against quantum adversaries, Grover's algorithm finds preimages in O(2^(n/2)) time. The birthday bound for collisions is unaffected by Grover — collision finding is not directly a search problem over an unstructured space. Hash-based signature schemes built on SHA-256 or SHAKE-256 with 256-bit output provide around 128-bit quantum security for preimage resistance.

## The Short Integer Solution Problem (SIS)

Alongside LWE, the Short Integer Solution problem appears frequently in signature schemes:

**SIS_(n,m,q,β)**: Given a uniformly random matrix **A** ∈ ℤ_q^(n×m), find a non-zero integer vector **z** ∈ ℤ^m with ‖**z**‖ ≤ β such that **Az** = **0** mod q.

SIS is the "signature-side" analogue of LWE. Solving SIS is equivalent to finding short vectors in certain lattices (specifically, in the kernel lattice of **A**). Regev and Micciancio showed reductions between SIS and approximate SVP, giving SIS a similar worst-case hardness guarantee to LWE.

CRYSTALS-Dilithium and FALCON both base their signature security on variants of SIS (combined with LWE for the full scheme).

## Summary of hard problems and their users

| Hard Problem | Algorithm Family | Users |
|---|---|---|
| Module-LWE / Module-SIS | Lattice | ML-KEM, ML-DSA |
| NTRU / RLWE over ℤ[x]/(xⁿ + 1) | Lattice | FN-DSA (FALCON) |
| Syndrome Decoding (Goppa codes) | Code-based | Classic McEliece |
| Syndrome Decoding (QC codes) | Code-based | BIKE, HQC |
| Hash function preimage resistance | Hash-based | SLH-DSA (SPHINCS+) |
| Multivariate quadratic systems | Multivariate | (none standardised) |

The chapters that follow examine each family with the above foundations in place.
