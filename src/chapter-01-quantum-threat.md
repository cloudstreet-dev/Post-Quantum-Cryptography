# Chapter 1: The Quantum Threat

## What actually breaks

Not everything breaks. Before cataloguing what quantum computers destroy, it is worth establishing what they do not.

Symmetric cryptography — AES, ChaCha20, SHA-256 — survives. Grover's algorithm provides a quadratic speedup for unstructured search, which halves the effective security level: AES-128 drops to roughly 64-bit security against a quantum adversary, AES-256 to 128-bit. The standard response is to double key lengths. AES-256 is already the conservative choice, and SHA-256 output truncation to 256 bits provides adequate preimage resistance. The symmetric world needs tuning, not replacement.

What breaks entirely is the asymmetric layer:

- **RSA** — security relies on the hardness of integer factorisation. Shor's algorithm factors an n-bit number in O(n³) quantum gate operations (or O(n² log n log log n) with optimised circuits). A 2048-bit modulus offers no meaningful security.
- **Diffie-Hellman and DSA** — security relies on the discrete logarithm problem in ℤ_p. Also O(n³) for Shor's algorithm on the multiplicative group.
- **ECDH and ECDSA** — security relies on the elliptic curve discrete logarithm problem. The same algorithm applies with minor modifications. A 256-bit elliptic curve key provides roughly the same quantum security as a 256-bit RSA key: none.

If you have deployed a system that uses RSA key exchange, ECDH key agreement, or any signature scheme based on integer factorisation or discrete logarithm, the relevant threat is not the ciphertext being broken now — it is the ciphertext being recorded now and decrypted later. This is the harvest-now-decrypt-later attack.

## Harvest now, decrypt later

The attack is straightforward. An adversary with network access records encrypted traffic today. The adversary does not need to break the encryption immediately. When a cryptographically relevant quantum computer becomes available — in ten years, in twenty years, at some point — the adversary applies Shor's algorithm to recover the session key and decrypts the stored ciphertext.

This makes the migration timeline urgent for any data with a long confidentiality requirement. Medical records, legal communications, classified information, intellectual property — anything that needs to remain confidential for more than a decade is already at risk if it travels over channels protected only by classical asymmetric cryptography.

For authentication — signatures, certificates, TLS mutual auth — the threat model is different. A signature is only useful at the moment of verification. Harvesting a signed message and breaking the key after the fact does not help an adversary forge new signatures; it only lets them verify (or potentially fabricate, if they can recover the private key) old ones. The forward secrecy requirement is less acute for authentication than for key exchange, but key forgery is still a serious threat.

## Shor's algorithm in brief

Shor's algorithm reduces integer factorisation to the problem of finding the period of a function. Given N = p × q, the algorithm:

1. Chooses a random a < N
2. Computes the period r of f(x) = a^x mod N using quantum phase estimation
3. If r is even and a^(r/2) ≢ -1 (mod N), computes gcd(a^(r/2) ± 1, N) to obtain factors

The quantum speedup comes entirely from step 2. Phase estimation uses the quantum Fourier transform to find the period of a function exponentially faster than any known classical algorithm. The rest of the algorithm is classical number theory.

The gate complexity for factoring an n-bit number scales as O(n²) to O(n³) depending on implementation optimisations. Physical qubit counts required to break RSA-2048 have been estimated at several million physical qubits when accounting for error correction overhead, given current error rates. Current devices have thousands of physical qubits with error rates that remain orders of magnitude too high. The gap is large. It is not obviously permanent.

## What quantum computers do not break

For completeness:

**Symmetric ciphers and hash functions** are not known to be broken by any quantum algorithm. Grover's algorithm provides a square-root speedup for searching over a space of 2^n elements, reducing 128-bit symmetric security to roughly 64-bit quantum security. This is a degradation, not a collapse.

**The new post-quantum schemes themselves** are designed with quantum adversaries in mind. Their security is not based on integer factorisation or discrete logarithm. Their underlying hard problems — finding short vectors in lattices, decoding random linear codes, inverting hash functions — do not have known polynomial-time quantum algorithms.

**One-time pads** are information-theoretically secure regardless of the adversary's computational model. They are also impractical for most applications for the same reasons they have always been impractical.

## The cryptographic response

The response to Shor's algorithm is to stop basing asymmetric cryptography on problems that Shor's algorithm solves. The cryptographic community has identified several families of mathematical problems that appear hard for both classical and quantum computers:

- **Lattice problems** — finding short or close vectors in high-dimensional lattices
- **Error-correcting code problems** — decoding random linear codes
- **Hash functions** — inverting collision-resistant hash functions
- **Multivariate polynomial systems** — solving overdetermined systems of multivariate polynomials over finite fields

Each of these forms the basis of one or more candidate post-quantum algorithms. The remaining chapters examine each family in detail.

## Timeline and urgency

No one can give you a precise date when cryptographically relevant quantum computers will exist. The theoretical requirements are understood; the engineering challenges are substantial and not fully characterised. The honest range of credible expert opinion spans from "within a decade" to "two or three decades, if ever at scale."

The migration argument does not depend on resolving that uncertainty. Cryptographic migrations are slow. Replacing the asymmetric layer across a large organisation's infrastructure takes years. Standardisation bodies, hardware security modules, TLS libraries, certificate authorities, and application code all need updating on different timescales. NIST's standardisation process took eight years. The HTTPS deployment of TLS 1.3 took longer than it should have. Waiting for the threat to materialise before beginning migration is not a strategy — it is an abdication.

NIST published final standards in 2024. The work of migration has begun.
