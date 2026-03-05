# Chapter 8: Hybrid Approaches and Migration

Migrating to post-quantum cryptography is not a flag day operation where everything changes at once. It is an incremental process that must run across a heterogeneous environment of existing systems, hardware that cannot be updated, and protocol versions that cannot be negotiated up. The standard approach during the transition period is hybrid cryptography: running classical and post-quantum algorithms simultaneously, such that breaking either one is insufficient to break the system.

## Why hybrid

The argument for hybrid construction is straightforward:

1. Post-quantum algorithms are newer and have shorter cryptanalytic histories than classical algorithms
2. A hybrid scheme provides the security of whichever underlying algorithm is stronger
3. If the post-quantum algorithm has an undiscovered flaw, the classical algorithm prevents complete failure
4. If the classical algorithm is broken by a quantum computer, the post-quantum algorithm prevents complete failure

The cost of hybrid is overhead: larger keys, larger ciphertexts or signatures, additional computation. For most applications during the transition period, this tradeoff is correct.

## Hybrid KEM construction

The canonical construction for hybrid key encapsulation combines two KEMs:

**Hybrid KEM**: Given KEM₁ and KEM₂ with encapsulation functions Encap₁, Encap₂:

1. Encapsulate with KEM₁: (ct₁, ss₁) ← Encap₁(pk₁)
2. Encapsulate with KEM₂: (ct₂, ss₂) ← Encap₂(pk₂)
3. Combine: ss = KDF(ss₁ || ss₂ || ct₁ || ct₂)
4. Output ciphertext: (ct₁, ct₂) and shared secret ss

The KDF step ensures that the final shared secret depends on both ss₁ and ss₂. Including the ciphertexts in the KDF input prevents certain multi-user attacks. The construction is secure if either KEM₁ or KEM₂ is secure — it is not necessary that both are secure, only that at least one is.

### X25519Kyber768 / X-Wing

The most widely deployed hybrid KEM is the combination of X25519 (classical ECDH over Curve25519) with CRYSTALS-Kyber / ML-KEM. Multiple specifications exist:

**draft-ietf-tls-hybrid-design**: The IETF TLS working group's general hybrid KEM design for TLS 1.3. Defines NamedGroup codepoints for hybrid key exchange.

**X-Wing** (RFC 9539): A specific, opinionated hybrid combining X25519 and ML-KEM-768. X-Wing is designed to be simple to implement correctly — it fixes the KDF construction rather than leaving it to the implementer. The combined key is 1216 bytes (32 bytes X25519 + 1184 bytes ML-KEM-768). The ciphertext is 1120 bytes. X-Wing is IND-CCA2 secure under the assumption that either X25519 or ML-KEM-768 is secure.

**Google Chrome / Cloudflare**: Chrome deployed X25519Kyber768 in August 2023 and reported that approximately 1% of TLS connections from Chrome used the hybrid key exchange within weeks of deployment. Cloudflare reported seeing it across millions of connections. The experiments revealed a small but non-negligible number of network middleboxes that fail on large ClientHello messages (the ML-KEM public key adds ~1 KB), causing connection failures. This is a known deployment hazard.

### HPKE with hybrid KEMs

HPKE (Hybrid Public Key Encryption, RFC 9180) is the standard framework for hybrid encryption in modern protocols. It composes a KEM, a key derivation function (HKDF), and an authenticated encryption scheme (AEAD) into a clean API. HPKE with an ML-KEM-based KEM is the expected pattern for new application-layer protocols.

## Hybrid signatures

Hybrid signatures are less standardised than hybrid KEMs. Several approaches exist:

**Concatenation**: Sign the message with both algorithms; the combined signature is the concatenation. A verifier must verify both signatures. This doubles signature size and computation.

**Composite signatures**: A single algorithm identifier refers to a combined key pair and signing procedure. Proposed in the IETF LAMPS working group for use in X.509 certificates. The composite approach hides the hybrid nature from applications that only understand a single signature algorithm identifier.

**Multiple certificates**: Issue separate certificates for classical and post-quantum keys; the TLS handshake negotiates which to use. This provides algorithm agility but requires infrastructure support for multiple end-entity certificates.

For TLS, the IETF's current direction favours hybrid KEMs (well-specified and already deployed) with migration to PQC-only signatures in a later phase, since signature forgery has a different threat model than key exchange (no harvest-now-decrypt-later for signatures).

## TLS 1.3 migration

TLS 1.3 has well-defined extension points for key exchange algorithm negotiation. Migrating the key exchange to a hybrid PQC KEM is:

1. Deployed and working in major browsers (Chrome, Firefox) and CDNs (Cloudflare, Fastly)
2. Standardised or near-standardised by IETF
3. Backward-compatible: clients advertise support for hybrid groups; servers that don't understand them fall back to classical

Migrating TLS signatures is harder:

1. Certificate authorities must issue PQC or composite certificates
2. The certificate chain must verify at each level (root → intermediate → end-entity)
3. Root certificate updates are slow (browser root programs, OS trust stores)
4. OCSP stapling and certificate transparency also require updates

The expected sequence for TLS:
- **Now**: Deploy hybrid KEM (X-Wing or equivalent) for key exchange
- **2025–2027**: Begin deploying PQC signatures in parallel (dual-signing or composite certificates)
- **2027–2030**: Phase out classical-only certificate chains as PQC infrastructure matures

## SSH migration

SSH has had quantum-resistant key exchange support in OpenSSH since version 9.0 (May 2022), which added sntrup761x25519-sha512 (a hybrid of the NTRU-based Streamlined NTRU Prime and X25519). OpenSSH 9.0 made this the default key exchange algorithm.

ML-KEM-based key exchange for SSH is in progress in the IETF SSH working group. The migration path is similar to TLS: key exchange is first (already deployed), host key and user authentication key migration follows.

## PKI and certificate migration

Public Key Infrastructure is the most painful part of the migration:

**Certificate chain length**: Adding PQC signatures to a certificate chain increases total signature size substantially. A 3-level chain with ML-DSA-65 signatures adds roughly 10 KB to a TLS handshake (versus ~1 KB for ECDSA P-256). SLH-DSA is worse. FN-DSA is better.

**Lifetime mismatches**: Root certificates have lifetimes of 20–30 years. A root certificate issued today with an RSA key has significant harvest-now-decrypt-later exposure for the signatures it issues, even if those signatures are only valid for short periods. PQC-signed root certificates should be deployed as soon as CAs support them.

**Hardware Security Modules**: HSMs must support PQC algorithms to be useful in PQC certificate authorities. HSM firmware updates, certification (FIPS 140-3), and procurement cycles add years to the timeline.

**IETF LAMPS**: The primary IETF working group for PKI migration. Working on PQC algorithm identifiers for X.509, CMS, and related standards. Composite algorithm identifiers (hybrid public keys in a single certificate) are a major workstream.

## Inventory and prioritisation

A practical migration begins with an inventory:

1. **What uses asymmetric cryptography?** TLS endpoints, SSH, VPNs, code signing, document signing, certificate authorities, key management systems, HSMs, firmware update mechanisms, authentication tokens, and anything else that touches a public key.

2. **What has long confidentiality requirements?** Systems protecting data that must remain confidential for 10+ years are the highest priority. Apply the harvest-now-decrypt-later threat model.

3. **What is on the critical path?** Root CAs and HSMs often have the longest procurement and certification cycles. Start there.

4. **What can be updated quickly?** TLS key exchange on internet-facing services is often the easiest: update library (if the library supports it), update configuration. Do this first for the quick win and to gain operational experience.

## Crypto-agility as a design principle

A lesson from the TLS 1.0/1.1 deprecation experience, and from SHA-1 sunset, and from MD5 sunset, and from SSLv3, and from every other cryptographic migration of the past thirty years: systems that hard-code algorithm identifiers and do not support negotiation are expensive to update.

Post-quantum migration is the time to build crypto-agility into systems that lack it:

- Algorithm identifiers should be data, not code
- Key and signature formats should have version fields
- Protocol negotiation should support advertising multiple algorithms
- Symmetric keys should be derivable at different lengths

A system built with crypto-agility in 2025 will be substantially cheaper to migrate in 2035, when the next required algorithm update arrives.

## Timelines and regulatory context

- **NSA**: Issued guidance in 2022 requiring US national security systems to plan migration by 2035, with interim milestones
- **NIST**: Published migration guidance alongside FIPS 203/204/205; recommends beginning transition immediately for high-value systems
- **EU**: ENISA published PQC recommendations in 2024; EU financial regulators have begun incorporating PQC requirements into supervisory guidance
- **CNSA 2.0**: The US Commercial National Security Algorithm Suite 2.0 (NSA, 2022) mandates ML-KEM, ML-DSA, and SLH-DSA as the required algorithms for national security systems

Regulatory deadlines create implementation deadlines. If your organisation operates in a regulated sector, the question is not whether to migrate but when the regulator will require it.
