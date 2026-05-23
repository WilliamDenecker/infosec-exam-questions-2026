# Question 27

Post-quantum cryptography introduces new practical challenges beyond purely theoretical security.

**Discuss the main practical challenges in deploying post-quantum cryptography, including:**

- **key and signature sizes,**
- **computation and memory requirements,**
- **interoperability and backward compatibility,**
- **long-term confidence in security assumptions.**

**Explain why quantum safety is still considered conjectural.**

## Answer

### Why Post-Quantum Cryptography Is Needed (ch2 PQCrypto p.1–5)

Current public-key cryptography (RSA, ECC, classical DH) relies on the computational hardness of problems — integer factorisation, discrete logarithm, elliptic curve discrete logarithm — that are efficiently solvable by a quantum computer using Shor's algorithm (ch2 PQCrypto p.5–7). A cryptographically relevant quantum computer (CRQC) would render all currently deployed public-key schemes insecure. Post-quantum cryptography (PQC) proposes alternative schemes based on problems believed to be hard for both classical and quantum computers.

---

### Challenge 1 — Key and Signature Sizes (ch2 PQCrypto p.10–14)

Classical cryptographic schemes benefit from compact key representations:
- RSA-2048: public key = 256 bytes; private key = ~1700 bytes; signature = 256 bytes
- ECDSA P-256: public key = 64 bytes; private key = 32 bytes; signature = 64 bytes

Post-quantum schemes are significantly larger:

**Lattice-based schemes** (e.g., CRYSTALS-Kyber for key exchange, CRYSTALS-Dilithium for signatures — NIST standards):
- Kyber-768: public key = 1184 bytes; ciphertext = 1088 bytes; private key = 2400 bytes
- Dilithium3: public key = 1952 bytes; private key = 4000 bytes; signature = 3293 bytes

**Hash-based signatures** (SPHINCS+):
- SPHINCS+-SHAKE-256s: public key = 64 bytes; private key = 128 bytes; but **signature = 29792 bytes** (~30 KB)

**Code-based schemes** (Classic McEliece):
- McEliece-8192128: public key = **1,357,824 bytes** (~1.3 MB); private key = 14080 bytes; ciphertext = 208 bytes

**Practical impact**:
- Larger public keys and ciphertext bloat TLS handshake messages — a single handshake with PQC can exceed the size of a TCP segment, requiring multiple round trips where TLS 1.3 with ECDHE currently fits in one
- DNS records carrying PQC certificates exceed standard UDP packet sizes, requiring TCP fallback or DNSSEC fragmentation
- Embedded systems and IoT devices with limited storage (a few KB) cannot accommodate 1.3 MB public keys (McEliece)
- Certificate chains become significantly larger when CAs use PQC signatures

---

### Challenge 2 — Computation and Memory Requirements (ch2 PQCrypto p.10–14)

**Computation**:
- Lattice-based schemes (Kyber, Dilithium) are computationally competitive with RSA and ECC — key generation and operations take microseconds on modern CPUs. This is the main advantage of lattice-based PQC.
- Hash-based schemes (SPHINCS+) are very slow for signing: SPHINCS+ with 128-bit classical security takes ~10 ms per signature — approximately 100× slower than ECDSA. This makes it unsuitable for high-throughput signing applications.
- Code-based encryption (McEliece) has fast encryption/decryption but requires 1+ MB of public key storage and transmission.

**Memory**:
- Lattice operations require polynomial arithmetic over $\mathbb{Z}_q$. For Kyber-768, this involves polynomials of degree 256 with 12-bit coefficients — manageable on most hardware.
- McEliece's 1.3 MB public key cannot be stored or processed on microcontrollers with tens of KB of RAM.
- Hash-based signature generation (SPHINCS+) requires maintaining a Merkle tree with many nodes in memory during signing.

**Hardware acceleration**: existing hardware acceleration (AES-NI, CLMUL for GCM) does not help PQC schemes. New instructions for Number Theoretic Transform (NTT) — the key operation in lattice-based schemes — would significantly accelerate Kyber/Dilithium but are not yet standardised in CPU instruction sets (as of 2026).

---

### Challenge 3 — Interoperability and Backward Compatibility (ch2 PQCrypto p.14–16)

**Protocol changes**: TLS, SSH, S/MIME, and other protocols were designed around compact public keys and ciphertext. Post-quantum keys don't fit in existing protocol message structures without extension:
- TLS 1.3 supports `key_share` extensions but their size was designed for ECDHE (32–65 bytes). Adding Kyber ciphertext (1088 bytes) requires updating all TLS implementations.
- PKIX (X.509) certificate format must accommodate new signature algorithms (Dilithium OIDs) and larger public key formats. Existing PKI infrastructure must be updated.

**Hybrid deployment**: the transition period requires running both classical and PQC schemes simultaneously — "hybrid" key exchange (e.g., X25519 + Kyber-768 in TLS). This doubles the key exchange overhead but ensures security even if one scheme is broken. IETF is standardising hybrid TLS extensions. This adds complexity and increases handshake size.

**Backward compatibility**: a server deploying PQC certificates must simultaneously serve legacy clients (which don't understand PQC signatures) and PQC-capable clients. This may require maintaining separate legacy certificate chains — doubling PKI management overhead.

**Harvest-now-decrypt-later attacks**: even before CRQCs exist, adversaries are recording encrypted traffic today to decrypt once a CRQC is available. Organisations with long-term confidentiality requirements (e.g., classified data, medical records) must deploy PQC **now** — before a CRQC exists — to protect current communications.

---

### Challenge 4 — Long-Term Confidence in Security Assumptions (ch2 PQCrypto p.8–10)

**Why quantum safety is conjectural** (ch2 PQCrypto p.8):

Post-quantum security claims rest on the assumed hardness of new mathematical problems (lattice problems like Learning With Errors, code-based problems, hash-based problems, isogeny problems). However:

1. **No unconditional proof of hardness**: no post-quantum scheme is proven secure under only the assumption that $P \neq NP$. All schemes assume specific mathematical problems are hard — assumptions that have not been proved. A breakthrough in mathematics or algorithmic design could undermine them.

2. **Limited cryptanalytic history**: RSA and ECC have been studied for 40+ years with massive effort. Lattice-based PQC schemes (standardised by NIST in 2024) have been studied intensively for only ~10 years. Shorter history means less confidence that all weaknesses have been found. The SIKE isogeny scheme was a NIST finalist — and was broken by a classical computer attack in 2022, just before standardisation.

3. **Quantum attacks on PQC**: while Shor's algorithm does not apply to lattice/code/hash problems, Grover's quantum algorithm does apply (to hash-based and symmetric schemes), halving their effective security. PQC schemes are designed to account for Grover's speedup (e.g., using larger parameters), but future quantum algorithms specifically targeting PQC mathematical structures could be more damaging.

4. **Unknown quantum algorithm landscape**: quantum computing theory is still young. An algorithm analogous to Shor's — but targeting lattice problems — cannot be ruled out. No mathematical theorem prevents such an algorithm from existing.

5. **Parameter confidence**: even within a given scheme, the specific parameters (lattice dimension, error distribution) chosen to achieve 128-bit post-quantum security are based on best current cryptanalytic knowledge. New classical or quantum attacks on specific parameter ranges could require re-parameterisation.

**Conclusion on conjecture**: "quantum safe" means "we believe no efficient quantum algorithm exists for this problem, based on current knowledge." It does not mean "provably secure against quantum computers." The phrase is a probabilistic confidence statement, not a mathematical guarantee.

---

### Summary

| Challenge | Description | Current mitigation |
|---|---|---|
| Key/signature sizes | 10–10000× larger than RSA/ECDSA | Lattice-based (Kyber/Dilithium) keeps sizes manageable; avoid McEliece/SPHINCS+ where size is critical |
| Computation | Hash-based signing slow; code-based keys huge | Prefer lattice-based; await hardware NTT support |
| Memory | MB-scale keys impossible on IoT | Select scheme by device profile; lattice-based works on moderate hardware |
| Interoperability | Existing protocols/infrastructure must change | Hybrid classical+PQC during transition; IETF/NIST standardisation |
| Security confidence | Shorter history; no proofs; SIKE was broken | Use NIST-standardised schemes; monitor cryptanalytic community; prefer schemes with diverse security assumptions |
| Quantum safety claim | Conjectural — no polynomial lower bound | Diversify: hybrid classical+PQC; design for algorithm agility |

### Sources

- IS_UG_2_2_SecM-adv-PQCrypto (p.1–16: post-quantum threat model, Shor's and Grover's algorithms, PQC scheme families, key sizes, security levels, NIST standardisation, conjectural nature of quantum safety)

_Status: Complete_  
_Done by: William_
