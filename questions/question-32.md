# Question 32

Post-quantum cryptography (PQC) should be resistant to quantum computer attacks like Shor's algorithm, which would make non-PQ asymmetric encryption (e.g. RSA or ECC) obsolete.

However, PQC has not been as thoroughly tested as non-PQ cryptography and might be vulnerable to non-quantum attacks. This is why so-called "hybrid encryption" has been proposed, which combines a PQC algorithm with a more proven non-PQ algorithm. The goal is to guarantee at least protection against non-quantum attacks and, hopefully (if the PQC algorithm holds), also quantum-resistant security.

**Consider the case of a digital signature.**

**Suggest a hybrid encryption solution that is secure against both non-quantum attacks and post-quantum attacks against authentication and (partial) data-integrity.**

**What are the advantages, risks, and drawbacks of this hybrid encryption approach?**

## Answer

### The Hybrid Signature Approach (ch2 PQCrypto p.14–16)

**Problem**: a pure classical signature (ECDSA P-256) is secure today against classical computers but would be broken by a CRQC (Shor's algorithm). A pure PQC signature (Dilithium) is believed quantum-safe but has a shorter history and could have undiscovered classical weaknesses.

**Solution**: **dual-signature** — compute both a classical signature and a PQC signature over the same message and publish both. The security guarantee is:
- **Against classical attackers**: ECDSA P-256 is unbroken → authentication and integrity are maintained by ECDSA
- **Against quantum attackers**: if ECDSA is broken by Shor's algorithm → Dilithium provides authentication and integrity (if Dilithium itself is quantum-safe)
- **Both broken simultaneously**: only if both ECDSA and Dilithium are broken — which requires a classical break of Dilithium AND a quantum computer — a much stronger requirement than breaking either alone

The combined scheme is secure if **at least one** of the two signatures is secure.

---

### Concrete Scheme: ECDSA P-256 + CRYSTALS-Dilithium3

**Key generation**:
- Classical: generate ECDSA P-256 key pair $(sk_C, pk_C)$
- PQC: generate Dilithium3 key pair $(sk_Q, pk_Q)$ (NIST standard, ch2 PQCrypto p.14)
- Distribute both public keys in a combined certificate or alongside the signed message

**Signing** (of message $M$):
1. Compute message hash: $h = SHA384(M)$ (ch2.2.3 p.30–40)
2. Compute classical signature: $\sigma_C = ECDSA\text{-}Sign(sk_C, h)$
3. Compute PQC signature: $\sigma_Q = Dilithium3\text{-}Sign(sk_Q, h)$
4. Publish: $(M, \sigma_C, \sigma_Q, pk_C, pk_Q)$

*Alternative*: sign the same raw message (not just the hash) with Dilithium, since Dilithium has its own internal hashing.

**Verification**:
1. Compute $h = SHA384(M)$
2. Verify $\sigma_C$ using $pk_C$ (ECDSA P-256)
3. Verify $\sigma_Q$ using $pk_Q$ (Dilithium3)
4. Accept if **both** verify (strong security); or accept if **either** verifies (relaxed — see below)

**Security policy choice**:
- **Both must verify** (AND policy): accepts only if both signatures are valid. Protects against forgery attempts that present only one valid signature while the other is forged or missing. This is the stronger policy — correct for most applications.
- **Either suffices** (OR policy): maintains availability if one scheme fails or is not supported by the verifier. Weaker — an attacker needs to forge only one signature.

For authentication and data integrity, the **AND policy** is recommended: both classical and PQC signatures must be present and valid.

---

### Advantages of Hybrid Signatures (ch2 PQCrypto p.14–16)

**1. Future-proof against quantum attacks**: if a CRQC is eventually built and breaks ECDSA, the Dilithium signature still protects authenticity and integrity. The transition to post-quantum can happen gradually without emergency re-signing of historical documents.

**2. Defense against unknown classical weaknesses in PQC**: if Dilithium has an undiscovered classical break (as happened with SIKE in 2022, ch2 PQCrypto p.9), ECDSA P-256 still provides authentication based on 40+ years of cryptanalytic confidence.

**3. Harvest-now-decrypt-later mitigation**: adversaries are storing signed data today to re-verify under a future quantum computer. Hybrid signatures ensure that even if ECDSA becomes verifiable under quantum attack, the Dilithium signature remains quantum-safe.

**4. Algorithm agility**: the hybrid structure makes it straightforward to swap out one component (e.g., replace Dilithium with a stronger PQC scheme) without changing the overall protocol structure.

**5. Incremental deployment**: legacy clients that don't support Dilithium can still verify ECDSA; modern clients verify both. The system works in mixed environments during the transition period.

---

### Risks and Drawbacks

**1. Increased signature size**:
- ECDSA P-256 signature: 64 bytes
- Dilithium3 signature: 3293 bytes
- Combined: ~3357 bytes + both public keys (~2016 bytes total for both public keys)
- **Impact**: significantly increases the size of signed objects (software updates, certificates, TLS handshakes, code signing). For embedded systems with constrained bandwidth or flash storage, this may be prohibitive. A TLS certificate with a Dilithium signature alone is substantially larger than with ECDSA.

**2. Increased computation**:
- Dilithium3 signing: ~0.5 ms on a modern CPU (fast for a lattice scheme)
- Dilithium3 verification: ~0.25 ms
- Combined: roughly doubles the signing/verification computation. For high-volume signature applications (code signing at scale, CA signing millions of certificates), this doubles computational load.
- On constrained hardware (IoT, smart cards): Dilithium may require more memory and compute than the device supports.

**3. Larger attack surface**: using two cryptographic systems means two implementations that could have bugs. Each implementation must be correct, constant-time, and free of side-channel vulnerabilities. The probability of an implementation error is higher with two schemes than one.

**4. Key management complexity**: managing two key pairs (classical + PQC) per entity doubles certificate size, key generation requirements, key storage, and revocation management (ch3.2 p.50–53). Both pairs must be protected, renewed, and revoked independently.

**5. Interaction effects (theoretical)**: if the classical and PQC signature schemes interact in unexpected ways — e.g., if an attacker can use information from one scheme to attack the other — the combined security could be lower than either scheme alone. In practice, signing the same hash with two independent schemes has no known interaction, but this is an area of ongoing research.

**6. Backward compatibility**: systems that only support ECDSA will ignore or reject the PQC signature component. Protocols must be updated to carry both signatures, which requires coordination across all implementations.

---

### Summary

| Property | ECDSA only | Dilithium only | ECDSA + Dilithium (hybrid) |
|---|---|---|---|
| Classical security | Yes (40+ years) | Unknown (short history) | Yes — ECDSA covers this |
| Quantum security | No (broken by Shor's) | Yes (conjectural) | Yes — Dilithium covers this |
| Security if PQC is broken classically | Yes | No | Yes — ECDSA still valid |
| Security if classical is quantum-broken | No | Yes | Yes — Dilithium still valid |
| Signature size | 64 bytes | 3293 bytes | ~3357 bytes |
| Key management | Simple (1 key pair) | Simple (1 key pair) | More complex (2 key pairs) |
| Implementation risk | Mature, stable | Newer, less audited | Higher (two implementations) |

**Recommended scheme**: ECDSA P-256 (with SHA-384 hash) + CRYSTALS-Dilithium3 (NIST standard, ch2 PQCrypto p.14), AND-policy verification, for applications where both authentication and data integrity against future quantum attacks are required.

### Sources

- IS_UG_2_2_SecM-adv-PQCrypto (p.8–16: post-quantum threat model, NIST PQC standardisation, hybrid encryption rationale, Dilithium, key sizes and performance)
- IS_UG_2_2_3_SecM_HashMac (p.85–87: ECDSA P-256 classical signatures)
- IS_UG_3_2_Appl_AuthMeth (p.50–53: certificate and key management considerations)

_Status: Complete_  
_Done by: William_
