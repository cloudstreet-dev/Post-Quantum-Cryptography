# Chapter 9: Implementation and Performance

Correct implementation of post-quantum cryptography is more demanding than correct implementation of classical cryptography. The algorithms are more complex, the constant-time implementation requirements are stricter in some cases, and the parameter spaces are less forgiving of mistakes. This chapter covers what implementers need to know.

## Reference implementations and where to start

The starting point for any implementation should be the reference code accompanying the NIST submissions. These are available from the NIST PQC project page and from the algorithm designers' repositories. They are written for correctness and readability, not performance — but they are correct, and correct is the priority.

**Do not start from scratch.** The algorithms contain subtleties — the Gaussian sampler in FALCON, the rejection conditions in ML-DSA, the modular arithmetic in ML-KEM's NTT — that have caused implementation errors even in expert code. Start with a reviewed implementation and optimise from there.

### Established libraries

| Library | Language | Algorithms | Notes |
|---|---|---|---|
| liboqs | C | ML-KEM, ML-DSA, FN-DSA, SLH-DSA, and more | Open Quantum Safe project; wraps multiple implementations |
| PQClean | C | All NIST standards | Clean portable implementations, heavily reviewed |
| BouncyCastle | Java | ML-KEM, ML-DSA, SLH-DSA | Mature Java ecosystem library |
| pqcrypto (crate) | Rust | ML-KEM, ML-DSA, others | Bindings to PQClean implementations |
| ml-kem (crate) | Rust | ML-KEM | Pure Rust, formally verified components |
| aws-lc-rs | Rust | ML-KEM, ML-DSA | AWS LibCrypto Rust bindings |
| OpenSSL 3.4+ | C | ML-KEM, ML-DSA | Integrated into the mainstream TLS library |
| BoringSSL | C | ML-KEM (X-Wing) | Google's TLS library; used in Chrome |

For TLS integration, the practical choices are OpenSSL 3.4+ (for most server applications) and BoringSSL (for Chrome and Chromium-based applications). Both expose ML-KEM hybrid key exchange in their TLS 1.3 implementations.

## Performance benchmarks

Benchmark numbers are hardware- and implementation-specific. The following are representative figures from the Open Quantum Safe benchmarking project on an x86-64 server (Intel Xeon, AVX2 enabled) to give a sense of relative costs.

### ML-KEM (key exchange)

| Algorithm | KeyGen (µs) | Encap (µs) | Decap (µs) | Total round-trip |
|---|---|---|---|---|
| X25519 (classical) | 14 | 14 | 14 | ~28 |
| ML-KEM-512 | 8 | 9 | 9 | ~18 |
| ML-KEM-768 | 14 | 15 | 16 | ~31 |
| ML-KEM-1024 | 22 | 24 | 25 | ~49 |

ML-KEM is competitive with X25519 in raw performance. The NTT is highly parallelisable and maps well onto SIMD instruction sets. The overhead compared to classical ECDH is primarily in key and ciphertext sizes, not computation time.

### ML-DSA (signatures)

| Algorithm | KeyGen (µs) | Sign (µs) | Verify (µs) |
|---|---|---|---|
| Ed25519 (classical) | 18 | 37 | 73 |
| ML-DSA-44 | 38 | 90 | 35 |
| ML-DSA-65 | 64 | 155 | 60 |
| ML-DSA-87 | 96 | 230 | 89 |

ML-DSA signs slower than Ed25519 but verifies at comparable speed. The signing cost is dominated by the polynomial multiplications and the potential for rejection (the abort condition from the Fiat-Shamir with aborts protocol). The expected number of restarts is bounded by the parameters; in practice, most signatures complete in the first attempt.

### FN-DSA (signatures)

| Algorithm | KeyGen (µs) | Sign (µs) | Verify (µs) |
|---|---|---|---|
| FN-DSA-512 | 3,900 | 320 | 90 |
| FN-DSA-1024 | 15,000 | 600 | 160 |

FN-DSA key generation is expensive — the NTRU key generation algorithm involves computing an approximate basis and finding a suitable polynomial pair, which requires more work than ML-DSA. This matters for ephemeral keys (generated per session) less than for long-lived keys. Signing and verification are fast relative to key generation.

### SLH-DSA (signatures)

| Algorithm | Sign (ms) | Verify (ms) |
|---|---|---|
| SLH-DSA-SHA2-128s | 55 | 1.6 |
| SLH-DSA-SHA2-128f | 7 | 3.1 |
| SLH-DSA-SHA2-256s | 530 | 6 |
| SLH-DSA-SHA2-256f | 22 | 10 |

Signing times are in **milliseconds**, not microseconds. The -s variants are slow by design — more layers in the hypertree produce smaller signatures but require more hash evaluations. This makes SLH-DSA unsuitable for high-frequency online signing but acceptable for code signing, certificate issuance, and other offline or low-frequency operations.

## Constant-time implementation

Side-channel attacks — timing attacks, cache timing attacks, power analysis — are a realistic threat against cryptographic implementations. Post-quantum algorithms are not immune.

### What must be constant-time

**ML-KEM decapsulation**: The implicit rejection step must be implemented in constant time. A timing difference between "ciphertext valid" and "ciphertext invalid" paths would allow an adversary to distinguish valid from invalid ciphertexts, breaking CCA2 security. The comparison of re-encrypted ciphertext against the submitted ciphertext must use a constant-time equality check (not `memcmp`, which may short-circuit).

**ML-DSA signing**: The abort condition (rejecting and restarting when z exceeds a bound) must not leak timing information about the private key. The bound check itself must be constant-time.

**FN-DSA signing**: The Gaussian sampler is the hardest component to implement in constant time. The reference implementation uses 64-bit floating-point arithmetic, which has data-dependent execution time on some processors. A constant-time integer implementation is possible but requires careful attention. This is the primary reason FIPS 206 includes normative implementation guidance.

**Key generation for all schemes**: Private key sampling must not leak information about the secret through timing.

### What is acceptable to be variable-time

Public operations — encryption, encapsulation, signature verification — can generally be variable-time, since they operate only on public data. There is no secret to leak.

The exception: if a private key or secret value is used during a "public" operation, that operation must still be constant-time for those parts. ML-KEM decapsulation uses the private key; ML-DSA signing uses the private key. These are not "public" operations.

## Key size considerations

Key sizes affect protocol design, storage, and network overhead:

| Algorithm | Public key | Private key | Signature/CT |
|---|---|---|---|
| X25519 | 32 bytes | 32 bytes | 32 bytes (shared secret) |
| ML-KEM-768 | 1,184 bytes | 2,400 bytes | 1,088 bytes |
| Ed25519 | 32 bytes | 64 bytes | 64 bytes |
| ML-DSA-65 | 1,952 bytes | 4,000 bytes | 3,293 bytes |
| FN-DSA-512 | 897 bytes | 1,281 bytes | ~666 bytes |
| SLH-DSA-SHA2-128f | 32 bytes | 64 bytes | 17,088 bytes |

In TLS 1.3, the server's Certificate message carries the end-entity certificate and chain. A typical ECDSA P-256 certificate is ~900 bytes for the end-entity certificate plus similar for intermediates. ML-DSA-65 certificates will be substantially larger. Applications that have tuned MTU sizes or Nagle algorithm settings for current TLS handshake sizes will need revisiting.

## Deployment lessons from early adopters

### Cloudflare experiment (2019)

Cloudflare and Google ran an experiment connecting Chrome to Cloudflare's servers using CECPQ2 (a hybrid KEM using X25519 and HRSS, an NTRU variant). Results:
- No meaningful latency increase for most connections
- A small fraction of connections (~0.5%) failed due to middlebox interference with large ClientHello messages
- Connection failures clustered around specific network operators and geographic regions, suggesting ISP-level deep packet inspection

This established the "large ClientHello problem" as a concrete deployment risk. The failure mode is that TLS ClientHello messages grow by ~1 KB when adding a hybrid key share; some firewalls and transparent proxies reject or fragment these incorrectly.

**Mitigation**: Most TLS implementations now attempt to coalesce the ClientHello to fit in a single IP datagram when possible, and fall back gracefully when the hybrid key exchange is rejected.

### Chrome ML-KEM deployment (2024)

Chrome 131 (late 2024) added support for X-Wing (X25519 + ML-KEM-768) as a supported key exchange group. Telemetry from the deployment:
- ML-KEM handshakes complete ~1–2 ms slower end-to-end on average (dominated by the increased data transfer, not computation)
- Compatibility issues similar to the 2019 experiment, concentrated in the same categories of devices

The practical lesson: compatibility testing against real network paths, not just test environments, is necessary before broad deployment.

## Constrained environments

For IoT, embedded systems, and microcontrollers:

**ML-KEM performance on ARM Cortex-M4**: Reference C implementations achieve ML-KEM-768 key generation in ~1 ms, encapsulation/decapsulation in ~1 ms each. Optimised assembly implementations cut this to ~0.3 ms. Acceptable for most applications where network round-trips dominate latency.

**Memory constraints**: ML-KEM-512 requires approximately 6 KB of RAM for the reference implementation at stack peak. ML-KEM-768 requires ~9 KB. This is within range for Cortex-M4 class devices (typically 64–256 KB RAM).

**SLH-DSA on constrained devices**: The -f variants are feasible (fast signing reduces computation time); the -s variants with ~55 ms signing on a fast server translate to hundreds of milliseconds on a Cortex-M. Plan accordingly.

**FN-DSA on constrained devices**: The floating-point arithmetic requirement is problematic for devices without an FPU. A constant-time integer implementation exists but is slower. Hardware FPU is effectively a prerequisite for practical FN-DSA deployment on embedded systems.

## Testing strategy

**Known-answer tests (KATs)**: All NIST submissions include KAT files. Run your implementation against the KATs before deployment. A mismatch indicates a bug.

**Fuzzing**: Fuzz the decapsulation and decryption paths. Malformed ciphertexts should be rejected cleanly, not crash or produce undefined behaviour.

**Side-channel testing**: For high-assurance implementations, use tools like `dudect` or `tlsfuzzer`'s timing analysis to verify constant-time properties.

**Interoperability testing**: Test against at least one other independent implementation. The Open Quantum Safe project provides test vectors and interoperability tools.

**Regression testing**: After any algorithm update, re-run KATs. Library updates are a common source of silent incompatibilities.
