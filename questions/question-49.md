# Question 49

Post-quantum cryptography (PQC) should be resistant to quantum computer attacks like Shor's algorithm, which would make non-PQ asymmetric encryption (e.g. RSA or ECC) obsolete.

However, PQC has not been as thoroughly tested as non-PQ cryptography and might be vulnerable to non-quantum attacks. This is why so-called "hybrid encryption" has been proposed, which combines a PQC algorithm with a more proven non-PQ algorithm. The goal is to guarantee at least protection against non-quantum attacks and, hopefully (if the PQC algorithm holds), also quantum-resistant security.

**Consider the case of the encapsulation of a secret key (to be used for the symmetric encryption of a file).**

**Suggest a hybrid encryption solution that is secure against both non-quantum attacks and post-quantum attacks against data confidentiality.**

**What are the advantages, risks, and drawbacks of this hybrid encryption approach?**

## Answer

### Problem Statement (ch2.2 PQCrypto p.19–20, p.29)

We need to encapsulate a symmetric key (to be used for symmetric file encryption) such that:
1. An attacker with a powerful classical computer cannot recover the key (protection against non-quantum attacks)
2. An attacker with a large-scale quantum computer cannot recover the key (protection against Shor's algorithm)

No single algorithm currently satisfies both requirements with full confidence: classical algorithms (RSA, ECDH) are broken by Shor's; PQC algorithms (Kyber/ML-KEM) are not as battle-tested and may have undiscovered classical attacks.

---

### Hybrid Key Encapsulation Scheme (ch2.2 PQCrypto p.29, p.43)

**Principle**: combine a classical key encapsulation mechanism (KEM) with a post-quantum KEM. Derive the final symmetric key by combining both shared secrets through a key derivation function (KDF). The final key is secure if **at least one** of the two KEMs is secure.

**Proposed construction**:

**Sender side (encapsulation)**:
1. Generate an ephemeral ECDH key pair: $(d_S, Q_S)$ where $Q_S = d_S \cdot G$ on NIST P-256 (or P-384)
2. Look up recipient's ECDH public key $Q_R$ and compute classical shared secret: $S_C = d_S \cdot Q_R$ (the x-coordinate)
3. Use recipient's ML-KEM-768 (CRYSTALS-Kyber) public key to encapsulate: $(CT_{PQ}, S_{PQ}) = \text{ML-KEM.Encaps}(pk_{PQ})$ where $S_{PQ}$ is the PQC shared secret and $CT_{PQ}$ is the ciphertext
4. Derive the session key: $K = \text{KDF}(S_C \| S_{PQ})$ — e.g., HKDF-SHA-256
5. Encrypt the file symmetric key $K_{file}$ using $K$: $C_{file} = E_K(K_{file})$
6. Send to recipient: $(Q_S, CT_{PQ}, C_{file})$

**Recipient side (decapsulation)**:
1. Recompute classical shared secret: $S_C = d_R \cdot Q_S$
2. Decapsulate PQC: $S_{PQ} = \text{ML-KEM.Decaps}(sk_{PQ}, CT_{PQ})$
3. Derive session key: $K = \text{KDF}(S_C \| S_{PQ})$
4. Decrypt: $K_{file} = D_K(C_{file})$

---

### Security Analysis (ch2.2 PQCrypto p.19, p.29, p.43)

**Protection against classical attacks**:
- The classical ECDH component (NIST P-256) relies on the elliptic curve discrete logarithm problem — computationally hard with today's classical computers
- Even if ML-KEM is broken by a classical attack (novel cryptanalysis), the ECDH component still protects $K$: an attacker would need to break both components to recover $K$

**Protection against quantum attacks**:
- Shor's algorithm breaks ECDH (ch2.2 PQCrypto p.12, p.19) — the ECDH component provides no quantum protection
- If ML-KEM-768 holds against quantum attacks (as currently conjectured), the PQC component protects $K$: a quantum adversary would need to break both components

**Security guarantee**: $K$ is secure against any attacker (classical or quantum) as long as at least one of the two KEMs is secure. This is a "belt and suspenders" approach.

---

### Advantages (ch2.2 PQCrypto p.29, p.43)

1. **Defence in depth**: two independent mathematical problems must be broken simultaneously to recover $K$. Compromising one algorithm does not compromise the key
2. **Future-proof without disruption**: if PQC is later found to be broken (as happened with SIKE in 2022, ch2.2 PQCrypto p.30), the classical component still protects existing encrypted files; if quantum computers arrive, the PQC component protects against them
3. **NIST standardised**: ML-KEM (CRYSTALS-Kyber) is standardised (FIPS 203, ch2.2 PQCrypto p.29) — regulatory compliance and vetted implementations available
4. **Backward compatible**: ECDH component is already deployed everywhere; the hybrid adds ML-KEM on top

---

### Risks and Drawbacks (ch2.2 PQCrypto p.43)

1. **Increased key and ciphertext sizes**: ML-KEM-768 public key = 1,184 bytes; encapsulation ciphertext = 1,088 bytes — significantly larger than ECDH (64 bytes each). For key encapsulation this is acceptable (encapsulating a 256-bit symmetric key), but relevant for bandwidth-constrained systems

2. **Implementation complexity**: two separate KEM implementations must be correctly integrated. Each implementation can have bugs; the KDF combination must be implemented correctly (wrong concatenation order, wrong KDF → security failure). Complexity is proportional to risk of implementation errors

3. **Quantum safety is conjectured, not proven** (ch2.2 PQCrypto p.43): ML-KEM's security rests on the hardness of Module-LWE (MLWE), which is believed to be quantum-safe but has not been proven. New attacks (classical or quantum) may appear; surprises are still possible

4. **No hardware acceleration for PQC** (ch2.2 PQCrypto p.42): classical hardware has AES-NI and SHA-NI but no NTT (Number Theoretic Transform) hardware acceleration for lattice-based operations — PQC is software-only and slower on current hardware

5. **Larger performance overhead**: two key generation operations, two encapsulation/decapsulation operations, one KDF — higher CPU and memory usage compared to a single classical KEM

---

### Summary

| Property | Classical KEM (ECDH) | PQC KEM (ML-KEM-768) | Hybrid |
|---|---|---|---|
| Classical security | ✓ Hard (ECDLP) | Conjectured | ✓ At least as strong as ECDH |
| Quantum security | ✗ Broken by Shor's | ✓ Conjectured | ✓ At least as strong as ML-KEM |
| Public key size | 64 bytes | 1,184 bytes | 1,248 bytes total |
| Ciphertext size | 64 bytes | 1,088 bytes | 1,152 bytes total |
| Standardisation | Mature (ECDH/X25519) | FIPS 203 (2024) | Proposed in IETF drafts |

### Sources

- IS_UG_2_2_SecM-adv-PQCrypto (p.12: Shor's breaks RSA/DH/ECDH; p.19: key exchange mechanisms — all current ones obsolete; p.20: requirements after quantum breakthrough; p.29: ML-KEM (CRYSTALS-Kyber) standardised as FIPS 203; p.30: 4th round — SIKE withdrawn after classical attack; p.43: remaining issues — quantum safety conjectured, uncertainty about PQC strength, larger key sizes)
- IS_UG_2_2_4_SecM_KeyExch (p.11–12: authenticated DH and key exchange; p.13: MITM attacks without authentication)
- IS_UG_2_2_2_SecM_AsymmEncr (p.67–70: ECC for key exchange)

_Status: Complete_  
_Done by: William_
