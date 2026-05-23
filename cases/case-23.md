# Case 23

An organization operates a classical RSA/ECC-based PKI but wants to prepare for post-quantum threats without breaking compatibility.

**Design a migration strategy. Which hybrid mechanisms would you deploy? How would certificates, signatures, and TLS be adapted? What are the risks of early PQC deployment?**

## Answer

### Part 1 — Why Migration Is Necessary

**Shor's algorithm** (ch2 PQCrypto p.12) runs in **polynomial time** on integer factorisation and discrete logarithms. A sufficiently large quantum computer breaks RSA, DSA, ECDSA, DH, and ECDH completely — regardless of key size (ch2 PQCrypto p.18, p.19). The entire existing PKI based on these algorithms must be replaced.

**Why not simply increase RSA key size?** Shor's algorithm scales polynomially with input size, not exponentially. Doubling the key size does not change the complexity class — a quantum computer still terminates in polynomial time (ch2 PQCrypto p.12). Larger RSA keys buy no long-term protection against quantum attacks.

The **harvest-now-decrypt-later** threat (ch2 PQCrypto p.20) is already active: an adversary captures today's TLS traffic and stores it. Once a quantum computer becomes available, they decrypt it retroactively. TLS key exchange is therefore the **most urgent** component to migrate, even before quantum computers are practical.

### Part 2 — NIST-Standardised PQC Algorithms (August 2024)

NIST standardised three algorithms in August 2024 (ch2 PQCrypto p.29):
- **ML-KEM** (CRYSTALS-Kyber, lattice-based) — key encapsulation / key exchange; replaces ECDH
- **ML-DSA** (CRYSTALS-Dilithium, lattice-based) — digital signatures; replaces ECDSA
- **SLH-DSA** (SPHINCS+, hash-based) — digital signatures; most conservative option

These algorithms are believed to be resistant to both classical and quantum attacks. However, **PQC quantum safety is still conjectured, not proven** — surprises remain possible (ch2 PQCrypto p.43). This is why a hybrid strategy is used rather than an immediate cutover.

### Part 3 — Hybrid Approach: Compatibility First

Because PQC algorithms are less battle-tested than RSA/ECC, and because many existing clients understand only classical algorithms, a **hybrid strategy** deploys both classical and PQC algorithms together. Security holds as long as **at least one** of the two algorithms remains secure. This hedges against both a quantum break of classical algorithms and a cryptanalytic break of a new PQC algorithm.

#### Hybrid TLS Key Exchange (Immediate — Addresses Harvest-Now-Decrypt-Later)

Replace ECDHE alone with a **hybrid key exchange**: run ECDHE (ch2.2.4 p.10) and ML-KEM in parallel in the same TLS 1.3 handshake (ch3.6 p.7–8). The session key is derived from both shared secrets:

```
session_key = KDF(ECDHE_shared_secret || ML-KEM_shared_secret)
```

- A **classical attacker** must break ECDHE — which classical computers cannot do in practice.
- A **quantum attacker** running Shor's can break ECDHE but not ML-KEM (lattice problems are not solved by Shor's, ch2 PQCrypto p.25).
- The session key is secure if **either** algorithm holds.

TLS 1.3 (ch3.6 p.7–8) with hybrid key exchange is backwards-compatible — classical-only clients fall back to ECDHE alone. Harvested traffic from today becomes undecryptable retroactively once ML-KEM is in use.

**Why TLS 1.3 and not 1.2?** TLS 1.2 allows static RSA key exchange (no forward secrecy) and legacy algorithms. TLS 1.3 mandates ephemeral key exchange (ch3.6 p.37) — a prerequisite for hybrid key exchange to provide forward secrecy.

#### Hybrid Certificates and Dual-Signature Scheme

Issue **dual-signature certificates**: each certificate carries two signatures — one from the existing ECDSA P-256 (ch2.2.3 p.85–87) CA and one from a new ML-DSA CA. The certificate contains two public keys.

- A classical-only client verifies the ECDSA signature → accepted.
- A PQC-aware client verifies the ML-DSA signature → accepted with quantum security.
- Both client types are compatible without any flag day.

For long-lived documents and code signing, sign with both ECDSA and ML-DSA simultaneously. Verifiers check whichever they support.

**Why ML-DSA for signatures and not SLH-DSA?** ML-DSA produces smaller signatures (~2.4 KB) than SLH-DSA (~8–50 KB) for everyday certificate signing. For the most critical signing contexts (root CA signatures), **SLH-DSA is more conservative**: its security reduces purely to hash function collision resistance — a simpler and more battle-tested assumption than lattice hardness (ch2 PQCrypto p.43).

#### Symmetric and Hash Algorithms: No Migration Needed

Symmetric encryption (**AES-256-GCM**, ch2.2.3 p.70–75) and hashing (**SHA-512**, ch2.2.3 p.24–32) **do not require replacement**: Grover's algorithm halves symmetric key strength, but AES-256 retains 128-bit effective security (ch2 PQCrypto p.16) and SHA-512 retains 256-bit preimage resistance (ch2 PQCrypto p.17). Only asymmetric algorithms (broken by Shor's) require migration.

### Part 4 — Three-Phase Migration

**Phase 1 (now — immediate)**: Deploy hybrid TLS key exchange (ECDHE + ML-KEM) on all servers. Addresses harvest-now-decrypt-later without breaking any existing client. No changes to certificate infrastructure yet.

**Phase 2 (near term)**: Issue dual-signature certificates from both the existing ECDSA CA and a new ML-DSA CA. Internal services begin accepting PQC-only connections from updated clients. Key management procedures updated to manage dual key pairs.

**Phase 3 (long term)**: Once PQC client support is widespread and ML-DSA/ML-KEM have accumulated sufficient cryptanalytic review, deprecate RSA/ECDSA. Revoke all classical certificates and migrate to PQC-only infrastructure.

### Part 5 — Risks of Early PQC Deployment

- **Unproven security**: ML-DSA and ML-KEM are based on lattice problems that are believed quantum-resistant but have not received decades of scrutiny. A cryptanalytic break is possible (ch2 PQCrypto p.43). The hybrid approach mitigates this — a break of ML-DSA does not break the ECDSA backup in a dual-signature certificate.
- **Performance overhead**: PQC algorithms have larger keys and signatures. ML-DSA signatures (~2.4 KB) vs ECDSA P-256 (~64 bytes). This increases certificate and handshake sizes, impacting constrained networks.
- **Implementation complexity**: maintaining dual algorithms doubles key management, certificate lifecycle, and revocation complexity. Operational errors are more likely when managing two parallel PKI hierarchies simultaneously.

### Part 6 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| TLS key exchange | Hybrid ECDHE + ML-KEM | ch2.2.4 p.10; ch2 PQCrypto p.25, p.29 | Defeats harvest-now-decrypt-later; secure against both classical and quantum attackers |
| Certificate signatures | Dual ECDSA + ML-DSA | ch2.2.3 p.85–87; ch2 PQCrypto p.29 | Backwards compatible; ML-DSA break → ECDSA holds; Shor's break → ML-DSA holds |
| Root CA signing | SLH-DSA (hash-based) | ch2 PQCrypto p.27, p.29, p.43 | Simplest security assumption; only hash collision resistance required |
| Symmetric / hash | AES-256-GCM + SHA-512 (no change) | ch2 PQCrypto p.16–17 | Grover's halves strength; AES-256 retains 128-bit; SHA-512 retains 256-bit |
| Why not RSA-4096 | Broken by Shor's regardless of key size | ch2 PQCrypto p.12, p.18 | Polynomial time; key size irrelevant |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.70–75, p.85–87)
- IS_UG_2_2_4_SecM_KeyExch (p.10)
- IS_UG_3_1_Appl_Basics (p.19)
- IS_UG_3_2_Appl_AuthMeth (p.28–29, p.50–53)
- IS_UG_3_6_Appl_TLS (p.7–8, p.18, p.37)
- IS_UG_2_2_SecM-adv-PQCrypto (p.12, p.16–20, p.25, p.27, p.29, p.43)

_Status: Complete_  
_Done by: William_
