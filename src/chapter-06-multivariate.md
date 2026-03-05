# Chapter 6: Multivariate Cryptography

Multivariate cryptography builds public-key schemes around the difficulty of solving systems of multivariate polynomial equations over finite fields. The fundamental hard problem — solving an overdetermined system of quadratic equations over 𝔽_2 — is NP-complete. The engineering challenge is constructing public keys that hide a tractable special structure while appearing to be instances of the hard problem.

This chapter covers a family that has produced useful signature schemes and a cautionary tale about what happens when the hidden structure is more visible than expected.

## The hard problem: MQ

**Multivariate Quadratic (MQ) problem**: Given m polynomial equations p₁, ..., p_m ∈ 𝔽_q[x₁, ..., x_n] of degree 2, find a solution (a₁, ..., a_n) ∈ 𝔽_q^n such that p_i(a₁, ..., a_n) = 0 for all i.

Over 𝔽_2, MQ is NP-complete (Garey and Johnson, 1979). Over larger fields the problem is similarly hard in the worst case. The best known algorithms for random instances run in exponential time — XL (eXtended Linearisation), Groebner basis methods (F4, F5), and hybrid approaches. Quantum speedups via Grover's algorithm apply, roughly halving the security level, which is addressed by choosing larger parameters.

## Building a trapdoor: the oil and vinegar structure

The central technique in most practical multivariate signature schemes is the oil and vinegar (OV) structure, introduced by Patarin in 1997.

Partition the n variables into "oil" variables O = {x₁, ..., x_o} and "vinegar" variables V = {x_{o+1}, ..., x_n}. Define central polynomials of the form:

$$\bar{p}_k(x) = \sum_{i \in O, j \in V} \alpha_{ijk} x_i x_j + \sum_{i,j \in V} \beta_{ijk} x_i x_j + \sum_{i \in O} \gamma_{ik} x_i + \sum_{i \in V} \delta_{ik} x_i + \epsilon_k$$

Crucially: there are **no oil-oil terms** (no x_i x_j where both i, j ∈ O).

**Why this enables inversion (signing)**: Given a target value y ∈ 𝔽_q^m, fixing random vinegar values turns each central polynomial into a linear equation in the oil variables (since all remaining terms involving oil variables are now linear, given fixed vinegars). With o = m, the resulting linear system has a square coefficient matrix over 𝔽_q; the probability it is invertible is high, and if not, simply try different vinegar values.

The public key is the central map composed with invertible affine transformations **S**: 𝔽_q^m → 𝔽_q^m and **T**: 𝔽_q^n → 𝔽_q^n, hiding the oil-vinegar structure. The private key is **S**, **T**, and the central map.

## Rainbow

Rainbow extends oil and vinegar by applying multiple layers. In an L-layer Rainbow scheme, the variable space is partitioned into L+1 sets V₁ ⊂ V₂ ⊂ ... ⊂ V_{L+1}, where layer ℓ uses variables in V_{ℓ+1} as oils and variables in V_ℓ as vinegars.

Rainbow submitted to the NIST PQC competition and reached the third round as a signature finalist. It was broken.

### The Rainbow attack (2022)

In February 2022, Ward Beullens published an attack that broke Rainbow III (128-bit security) on a standard laptop in 53 hours. The attack exploited the rectangular MinRank structure inherent in Rainbow's layered construction — the linear relationships between the oil spaces of consecutive layers create a recoverable structure that allows recovery of an equivalent private key.

Beullens' attack demonstrated that the effective security of Rainbow III was closer to 50–60 bits rather than 128 bits. Rainbow was eliminated from the NIST process shortly after.

This is a recurring pattern in multivariate cryptography: the structured trapdoor that enables efficient signing also creates structured linear algebra in the public key that cryptanalysts can exploit. Getting this balance right is difficult. Rainbow had been considered secure for over a decade before the attack.

## GeMSS and other MQ-based schemes

GeMSS (Great Multivariate Short Signature) uses a different trapdoor: the HFEv− (Hidden Field Equations with vinegar and minus modifiers) structure. Instead of oil and vinegar, HFE represents the central map as a univariate polynomial over a large field extension — solving for preimages reduces to solving a univariate polynomial, which is efficient.

GeMSS survived the NIST process as an alternate candidate but was not standardised. Its parameter sizes (extremely large public keys: 352 KB to 1.2 MB) and relatively slow verification made it unattractive compared to lattice-based alternatives.

## MAYO and UOV: the current state

After the Rainbow break, the multivariate community largely converged on two directions:

**UOV (Unbalanced Oil and Vinegar)**: The original oil and vinegar scheme (no layering) has no known attack analogous to Beullens' Rainbow break. The trade-off is that pure UOV requires more vinegar variables relative to oil variables to achieve adequate security, resulting in larger keys.

**MAYO**: A variant submitted to the NIST Additional Signatures project that uses "whipping" — evaluating the public map multiple times with different inputs and combining results — to reduce signature sizes while using the UOV structure. MAYO has 256-byte to 838-byte public keys (much smaller than classic UOV) and is an active NIST candidate.

## Quantum resistance assessment

Multivariate schemes resist known quantum attacks reasonably well. The Grover speedup applies to the underlying MQ problem search, and parameters are chosen with this in mind. There are no known polynomial-time quantum algorithms for MQ analogous to Shor's algorithm for factorisation.

The practical concern with multivariate cryptography is different: the history of structural breaks. Nearly every specific instantiation of a multivariate trapdoor has eventually had its structure exploited — sometimes subtly, sometimes catastrophically. The community has learned from each break and the current generation of schemes (MAYO, UOV) is more carefully designed. But the confidence level derived from a five- to eight-year cryptanalytic history is qualitatively different from the decades behind lattice problems or random code decoding.

## Summary

| Scheme | Status | Signature | Public key | Notable |
|---|---|---|---|---|
| Rainbow III | Broken 2022 | 164 bytes | 157 KB | MinRank attack by Beullens |
| GeMSS | Not standardised | 33–65 bytes | 352 KB – 1.2 MB | Very large keys |
| MAYO (NIST Alt. Sigs.) | Active candidate | 321–838 bytes | 256–5,488 bytes | Smallest MQ keys |
| UOV | Standardised by NIST (Alt. Sigs.) | ~96–196 bytes | ~66–2,252 KB | Classic construction |

No multivariate scheme is in the current primary NIST PQC standards. MAYO and UOV are candidates in the NIST Additional Signatures project (a separate process for signatures beyond the initial three standards). Whether multivariate cryptography achieves mainstream standardisation depends on whether the current generation of schemes survives continued cryptanalytic scrutiny — the same test that Rainbow failed.
