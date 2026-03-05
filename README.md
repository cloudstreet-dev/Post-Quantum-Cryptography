# Post-Quantum Cryptography

A CloudStreet technical reference on post-quantum cryptography: the algorithms, the standards, and the migration work that is coming whether you are ready for it or not.

## What this book covers

Quantum computers capable of breaking RSA-2048 or ECDH-256 do not exist yet. The timeline for when they will exist is contested, with estimates ranging from "ten years" to "twenty years" depending on which researcher you ask and how optimistic they are feeling. The cryptographic community has decided, reasonably, that waiting to find out is a poor strategy.

NIST completed its Post-Quantum Cryptography standardization process in 2024, publishing three standards:

- **ML-KEM** (FIPS 203) — a key encapsulation mechanism based on the Module Learning With Errors problem
- **ML-DSA** (FIPS 204) — a digital signature scheme based on the Module Learning With Errors problem
- **SLH-DSA** (FIPS 205) — a stateless hash-based digital signature scheme

A fourth standard, **FN-DSA** (FIPS 206), based on the FALCON lattice signature scheme, is in final stages.

This book covers the mathematical foundations behind these standards, the competing algorithm families that did not make the cut (and why), practical implementation considerations, and how to migrate existing systems without introducing new categories of problems.

## Structure

| Chapter | Topic |
|---------|-------|
| 1 | The Quantum Threat — what breaks and what does not |
| 2 | Mathematical Foundations — lattices, codes, and hard problems |
| 3 | Lattice-Based Cryptography — LWE, CRYSTALS-Kyber, Dilithium, FALCON |
| 4 | Code-Based Cryptography — McEliece and friends |
| 5 | Hash-Based Signatures — Merkle trees to SPHINCS+ |
| 6 | Multivariate Cryptography — polynomial systems and their failure modes |
| 7 | The NIST PQC Standards — ML-KEM, ML-DSA, SLH-DSA, FN-DSA |
| 8 | Hybrid Approaches and Migration — running old and new in parallel |
| 9 | Implementation and Performance — key sizes, benchmarks, real deployments |
| 10 | Acknowledgements |

## Reading online

The rendered book is available at: [cloudstreet-dev.github.io/Post-Quantum-Cryptography](https://cloudstreet-dev.github.io/Post-Quantum-Cryptography/)

## Building locally

```sh
cargo install mdbook
mdbook build
mdbook serve  # serves at http://localhost:3000
```

## License

Released under [CC0 1.0 Universal](LICENSE). No rights reserved.
