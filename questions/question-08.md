# Question 8

When browsing the (secure) home page of the university in May 2022, these were the essential security properties I found in the chain of ceritifcates:

- The website (www.ugent.be) uses an RSA-4096 key pair ($PR_W, PU_W$). The public exponent of this key is $2^{16} + 1$.
- The key ($PU_W$) is certified in a X.509v3 certificate ($CA_S \ll W \gg$) by GEANT TLS RSA 1. This certificate states this key pair is intended for signing and key encipherment, and not for certification (critical V3 extensions). The validity of the certificate is from 2026-04-28 to 2026-11-1.3 This certificate has been signed using SHA2-256 as a hash function and the RSA-3072 private key ($PR_S$) of GEANT TLS RSA 1 (the public exponent of this key is also $2^{16} + 1$).
- The corresponding public key ($PU_S$) of GEANT TLS RSA 1 has in its turn been certified in a X.509v3 certificate ($CA_U \ll S \gg$) by HARICA TLS RSA Root CA 2021. This certificate states this key pair is intended for signing, for signing CRLs, and for signing (final) certificates (as a CA) (critical V3 extensions). The validity of the certificate is from 2025-01-03 to 2039-12-31. This certificate has been signed using SHA2-256 as a hash function and the RSA-4096 private key ($PR_U$) of HARICA TLS RSA Root CA 2021 (the public exponent of this key is also $2^{16} + 1$).
- Finally, the corresponding public key ($PU_U$) of HARICA TLS RSA Root CA 2021 is certified in a self-signed X.509v3 certificate ($CA_U \ll U \gg$). This certificate states this key pair is intended for signing, for signing CRLs, and for signing (final or intermediate) certificates (as a CA) (critical V3 extensions). The validity of the certificate is from 2021-02-19 to 2045-02-13. This certificate has been signed using SHA2-256 as a hash function.
- All signatures use PKCS #1 v1.5 formatting.

**Explain why the different security choices (algorithms, key lengths, validity periods) do or don't make sense.**

**Explain why you would keep or change these security choices.**

## Answer

### Overview of the Certificate Chain (ch3.2 p.28–29)

The chain has three levels:
1. **Leaf certificate**: www.ugent.be — RSA-4096, valid ~6 months (Apr–Nov 2026)
2. **Intermediate CA**: GEANT TLS RSA 1 — RSA-3072, valid ~15 years (2025–2039)
3. **Root CA**: HARICA TLS RSA Root CA 2021 — RSA-4096, valid ~24 years (2021–2045), self-signed

---

### Analysis of Each Security Choice

#### 1. RSA Key Lengths

**Leaf (RSA-4096)**: A 6-month web server certificate. RSA-4096 provides approximately 140-bit classical security — significantly stronger than the ~112-bit security of RSA-2048. For a short-lived leaf certificate, RSA-2048 would be entirely adequate. RSA-4096 is not wrong (more security is never harmful here), but it is overkill for a 6-month certificate and incurs unnecessary computational cost: RSA operations scale with key size, and RSA-4096 handshakes require approximately 8× more computation than RSA-2048 for signature verification (ch2.2.3 p.80–84). The choice of RSA-4096 for a short-lived leaf certificate provides no meaningful security benefit over RSA-2048 in the current threat environment.

**Intermediate CA (RSA-3072)**: Valid until 2039 (~15 years). RSA-2048 is recommended only until approximately 2030; for a certificate valid to 2039, RSA-2048 would be borderline. RSA-3072 provides approximately 128-bit classical security and is appropriate for this lifetime. This choice **makes sense** (ch2.2.3 p.80 — key length recommendations by lifetime).

**Root CA (RSA-4096)**: Valid until 2045 (~24 years). A root CA has the longest lifetime and its compromise would be catastrophic (an attacker with $PR_U$ can issue certificates for any domain). RSA-4096 with a 24-year validity period is justified — the longer the lifetime, the larger the key should be to resist advances in cryptanalysis. This choice **makes sense**.

**Overall RSA key length observation**: the RSA-4096 leaf certificate is unnecessarily large; it would be more appropriate at RSA-2048 or RSA-3072. The intermediate and root CA key sizes are proportionate to their lifetimes.

#### 2. Public Exponent $e = 2^{16} + 1 = 65537$ (All Keys)

This is the standard RSA public exponent (ch2.2.3 p.77). The value 65537 = $2^{16} + 1$ is a Fermat prime with exactly two 1-bits in binary (10000000000000001), making modular exponentiation by $e$ efficient (only one modular multiplication using square-and-multiply). It is small enough for fast verification but large enough to avoid small-exponent attacks (e.g., $e = 3$ with small messages). This choice **makes sense** and is universally recommended.

#### 3. Signature Hash Function (SHA-256 at All Levels)

SHA-256 provides 128-bit collision resistance (ch2.2.3 p.30–40). For certificates valid until 2039 and 2045, SHA-256 is appropriate by current standards. SHA-1 is completely deprecated for certificate signing (collision attacks are practical); SHA-256 is the correct replacement.

For the root CA valid until 2045 (approximately 20 more years), SHA-384 or SHA-512 would provide a larger security margin (192-bit and 256-bit collision resistance respectively), especially given the trend toward post-quantum migration. SHA-256 is currently acceptable but a longer-lived CA might benefit from SHA-384 for future-proofing.

This choice largely **makes sense** but could be improved at the root CA level.

#### 4. PKCS #1 v1.5 Signature Padding (All Levels)

This is the most questionable security choice. PKCS #1 v1.5 (ch2.2.3 p.77–79) is the older RSA padding scheme. For **encryption**, PKCS #1 v1.5 is definitively broken (Bleichenbacher adaptive chosen-ciphertext attacks). For **signatures**, PKCS #1 v1.5 is less directly vulnerable, but it is not tightly secure in the provable-security sense.

The modern, recommended alternative is **RSA-PSS** (Probabilistic Signature Scheme) (ch2.2.3 p.85). RSA-PSS is provably secure in the random oracle model — its security reduces directly to the hardness of the RSA problem, whereas PKCS #1 v1.5 signatures have no such tight reduction. TLS 1.3 mandates RSA-PSS for all new signatures; PKCS #1 v1.5 is only supported for backward compatibility in TLS 1.3 (ch3.6 p.7–8).

The continued use of PKCS #1 v1.5 for all signatures in this chain **does not make sense** — RSA-PSS should be used instead. This is a clear improvement to make.

#### 5. Validity Periods

**Leaf certificate (~6 months)**: short validity periods are good practice (ch3.2 p.50–53). If the private key $PR_W$ is compromised, the certificate is invalid within 6 months at most. Frequent rotation limits exposure. Modern browser standards are pushing for even shorter leaf certificate lifetimes (1–3 months). The 6-month period **makes sense** and is conservative by current industry trends.

**Intermediate CA (~15 years, to 2039)**: intermediate CAs are harder to rotate (their public keys must be included in TLS handshakes or pre-installed in clients). A 15-year lifetime is longer than ideal but within accepted industry practice for intermediate CAs. The intermediate CA certificate can be revoked via CRL (ch3.2 p.50–53) if compromised. This choice is acceptable but somewhat long.

**Root CA (~24 years, to 2045)**: root CA certificates are embedded in browsers and operating systems — their rotation requires coordinated distribution across billions of devices, taking years. Long lifetimes are a practical necessity. The HARICA root being valid until 2045 is within normal range for root CAs. This choice **makes sense**.

#### 6. X.509v3 Extensions — Key Usage Constraints (ch3.2 p.28–29)

The critical extensions correctly constrain what each certificate can be used for:
- **Leaf certificate**: signing + key encipherment, CA:FALSE — correctly prevents the leaf key from signing certificates. A compromised $PR_W$ cannot be used to issue fraudulent sub-certificates. This constraint **makes sense**.
- **Intermediate CA**: signing + CRL signing + CA:TRUE for end-entity certificates — correctly allows the intermediate to sign leaf certificates and CRLs but cannot delegate CA authority further. **Makes sense**.
- **Root CA**: signing + CRL signing + CA:TRUE for all certificates (including intermediate CAs) — the root can do everything. Self-signed. **Makes sense** for the trust anchor.

---

### Summary of Changes

| Choice | Assessment | Change |
|---|---|---|
| Leaf RSA-4096 | Overkill for 6-month cert; safe but wasteful | Consider RSA-2048 or RSA-3072 for performance |
| Intermediate RSA-3072 | Appropriate for 15-year lifetime | Keep |
| Root RSA-4096 | Appropriate for 24-year lifetime | Keep |
| Public exponent 65537 | Standard and correct | Keep |
| SHA-256 (all levels) | Correct; root CA could use SHA-384 for margin | Keep for intermediate/leaf; upgrade root to SHA-384 |
| PKCS #1 v1.5 padding | Outdated; no provable security reduction for signatures | **Change to RSA-PSS** |
| Leaf validity 6 months | Good practice; rotate frequently | Keep (or shorten further) |
| Key usage extensions | Correctly constrained | Keep |

**The single most important change**: replace PKCS #1 v1.5 signature padding with RSA-PSS across the entire chain. RSA-PSS is the modern, provably secure alternative and is already mandated by TLS 1.3 for new certificates (ch3.6 p.7–8).

**Secondary consideration**: for new deployment, the entire RSA hierarchy could be replaced with an ECDSA P-384 hierarchy. ECDSA P-384 provides 192-bit security with much smaller keys (384-bit vs 3072/4096-bit), faster operations (especially on embedded clients), and smaller certificate sizes that reduce TLS handshake overhead (ch2.2.3 p.85–87). The main reason to keep RSA is backward compatibility with legacy clients that may not support elliptic curve certificates.

### Sources

- IS_UG_3_2_Appl_AuthMeth (p.28–29: X.509v3 certificate structure and extensions; p.50–53: certificate revocation; p.85–87: digital signature schemes)
- IS_UG_2_2_3_SecM_HashMac (p.30–40: hash function security; p.77–84: RSA operations, key sizes, padding)
- IS_UG_3_6_Appl_TLS (p.7–8: TLS 1.3 certificate requirements, RSA-PSS mandate)

_Status: Complete_  
_Done by: William_
