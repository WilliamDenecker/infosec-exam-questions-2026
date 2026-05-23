# Case 19

**Which approach would you suggest if you want to ensure the confidentiality of an electronic document for a long period of time (10 years or even more)? Which algorithms would be suitable to achieve this long term secure storage? What non-cryptographic measures would you suggest? What could be remaining threats?**

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | The document must remain unreadable to unauthorised parties for the entire storage period. Cryptographic algorithms can weaken over time. | An algorithm break after year 5 exposes a document stored for 10 years. All historical documents become retroactively readable. |
| **Data integrity** | Yes | ch1 p.34 | Silent bit-rot, media degradation, or deliberate tampering must be detectable over long storage periods. | The document is silently corrupted on aging media; by the time it is needed, the contents are wrong and the damage is undetectable. |
| **Access control / authorisation** | Yes | ch1 p.30 | Only authorised custodians may decrypt or access the document. Physical and logical access must be restricted. | An internal attacker decrypts the document. Without access control, confidentiality is defeated from the inside. |

### Part 2 — Encryption Algorithm Choice

#### Symmetric Encryption: AES-256-GCM

**AES-256-GCM** (ch2.2.3 p.70–75) is the correct choice for document encryption:

- **AES-256** (256-bit key) is explicitly **quantum-safe at 128 bits of effective security** after Grover's algorithm halves symmetric key strength (ch2 PQCrypto p.11, p.16). 128 bits of post-quantum security is currently considered sufficient.
- **AES-128 must not be used**: Grover's algorithm reduces it to only 64 bits of effective security — **obsolete** after a quantum breakthrough (ch2 PQCrypto p.16). A document encrypted with AES-128 today could be decryptable in the time remaining in its 10-year storage life.
- **GCM mode** provides both confidentiality and integrity in one operation (ch2.2.3 p.70–75). The 128-bit authentication tag detects any tampering or silent bit-rot.

**Why not AES-128?** The question specifically asks about 10+ year security. Grover's algorithm running on a quantum computer reduces AES-128's effective security to 64 bits — the slides explicitly label this as obsolete (ch2 PQCrypto p.16). AES-256 must be used.

For key derivation from a passphrase, use **SHA-512**: after Grover's algorithm, SHA-512 retains **256 bits of preimage resistance** (ch2 PQCrypto p.17), well above any practical threshold. SHA-256 retains only 128 bits (ch2 PQCrypto p.17).

### Part 3 — Key Protection: Symmetric Key Wrapping Only

The 256-bit document encryption key must itself be protected. Critically: **do not wrap the key using RSA or ECDH**. Shor's algorithm (ch2 PQCrypto p.12) solves integer factorisation and discrete logarithms in **polynomial time** — RSA and ECDH are fully broken by a quantum computer, regardless of key length (ch2 PQCrypto p.18, p.19). Wrapping a 256-bit AES key with RSA-2048 destroys the long-term security guarantee entirely.

**Why RSA-4096 does not help**: Shor's algorithm runs in polynomial time with respect to the key size. A larger RSA key does not change the complexity class — the algorithm still terminates efficiently on a sufficiently large quantum computer (ch2 PQCrypto p.12).

Instead, wrap the document key with another **AES-256** key (symmetric key wrapping) derived from a strong passphrase using HMAC key derivation (ch2.2.3 p.63–66). The master wrapping key is:
- Stored on offline encrypted media, physically secured and air-gapped
- Split across multiple custodians (two-of-three required) to prevent single-person insider risk

If asymmetric key distribution is required (e.g. multiple recipients need independent access), use **ML-KEM** (CRYSTALS-Kyber, standardised by NIST in August 2024, ch2 PQCrypto p.29) — a lattice-based key encapsulation mechanism resistant to Shor's algorithm. ML-KEM replaces ECDH for quantum-safe asymmetric key exchange (ch2 PQCrypto p.19, p.25).

**Caveat**: ML-KEM and all PQC algorithms are less tested than classical schemes. Quantum safety is **conjectured, not proven** — surprises remain possible (ch2 PQCrypto p.43). This is precisely why re-encryption planning (Part 4) remains essential even with PQC.

### Part 4 — Re-encryption Before Algorithm Weakening

No algorithm can be guaranteed secure for 10+ years with absolute certainty. The approach:

1. Monitor NIST and BSI algorithm recommendations continuously.
2. No algorithm breaks overnight — warnings precede breaks by years (ch2 PQCrypto p.15; examples: MD5, SHA-1, RSA-768 all had years of warnings).
3. When warnings appear about AES-256 or GCM: **decrypt with the current key → immediately re-encrypt with the then-current recommended algorithm and a new key**.
4. Securely destroy the old key after successful re-encryption.

This is analogous to the TTP timestamp chain renewal in case 12 — the strategy is to stay ahead of any algorithm weakening rather than trying to pick one algorithm that will never weaken.

### Part 5 — Non-Cryptographic Measures

Cryptography alone is insufficient for 10+ year storage:

- **Media durability**: hard disks fail after 3–5 years. Use magnetic tape (30+ years in controlled storage) or archival-grade optical media. Refresh onto new media every 5 years.
- **Multiple geographic copies**: at least two physically separate locations. Protects against fire, flood, and physical destruction of the primary site.
- **Format longevity**: store in a well-documented, open file format. Proprietary formats become unreadable when the software is discontinued.
- **Integrity verification** (ch1 p.34): periodically verify the AES-256-GCM authentication tag to detect silent bit-rot or media corruption before it becomes unrecoverable. A corrupted document discovered after 9 years may have been deteriorating silently for years.
- **Inventory and key escrow documentation**: maintain a written record of storage locations, which key version was used, and custodian identities. A perfectly encrypted document is permanently inaccessible if no one knows where the key is.

### Part 6 — Remaining Threats

- **Quantum computing breakthrough before re-encryption**: if a large-scale quantum computer arrives before the key is migrated away from any RSA/ECDH wrapping, the wrapped key is exposed (ch2 PQCrypto p.19). This is why asymmetric key wrapping must use ML-KEM or be avoided entirely. AES-256 itself drops to 128-bit effective security post-quantum — still acceptable, not broken.
- **PQC algorithm failure**: ML-KEM and other PQC algorithms are newer and less battle-tested (ch2 PQCrypto p.43). A cryptanalytic break is possible. A hybrid approach — wrapping the key with both AES-256 symmetric and ML-KEM asymmetric — ensures breaking one does not expose the key.
- **Key loss**: if the encryption key is permanently lost (all custodians unavailable, media failure), the document is permanently unrecoverable. Key backup and custodian succession planning are as important as the cryptographic choices.
- **Insider threat**: a custodian with key access can decrypt. Split custody (two-of-three) and access logs (ch3.7 p.83) mitigate but cannot eliminate insider risk (ch1 p.30).
- **Legal compulsion**: a court order may compel key disclosure. This is a legal threat, not solvable with stronger cryptography.

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Document encryption | AES-256-GCM | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | 128-bit effective security post-quantum; AES-128 gives only 64-bit (obsolete) |
| Key derivation hash | SHA-512 | ch2 PQCrypto p.17 | 256-bit preimage resistance post-quantum; SHA-256 gives only 128-bit |
| Key wrapping | AES-256 symmetric wrapping (no RSA/ECDH) | ch2 PQCrypto p.12, p.18–19 | RSA/ECDH broken by Shor's regardless of key size |
| Asymmetric distribution | ML-KEM (Kyber, NIST Aug 2024) | ch2 PQCrypto p.25, p.29 | Lattice-based; resistant to Shor's algorithm |
| Long-term strategy | Re-encrypt before algorithm weakens | ch2 PQCrypto p.15 | No algorithm guaranteed 10+ years; renewals keep protection current |

### Sources

- IS_UG_1_Introduction (p.15, p.30, p.34)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.63–66, p.70–75)
- IS_UG_3_1_Appl_Basics (p.6)
- IS_UG_2_2_SecM-adv-PQCrypto (p.11–12, p.15–17, p.18–19, p.25, p.29, p.43)

_Status: Complete_  
_Done by: William_
