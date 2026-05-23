# Question 20

When browsing the (secure) home page of the healthcare platform CoZo in May 2026, these were the essential security properties I found in the chain of ceritifcates:

- The website (www.cozo.be) uses an RSA-4096 key pair ($PR_W, PU_W$). The public exponent of this key is $2^{16} + 1$.
- The key ($PU_W$) is certified in a X.509v3 certificate ($CA_T \ll W \gg$) by R12 (Let's Encrypt). This certificate states this key pair is intended for signing and key encipherment, and not for certification (critical V3 extensions). The validity of the certificate is from 2026-03-19 to 2026-06-17. This certificate has been signed using SHA2-256 as a hash function and the RSA-2048 private key ($PR_T$) of R12 (Let's Encrypt) (the public exponent of this key is also $2^{16} + 1$).
- The corresponding public key ($PU_T$) of R12 (Let's Encrypt) has in its turn been certified in a X.509v3 certificate ($CA_D \ll T \gg$) by ISRG Root X1. This certificate states this key pair is intended for signing, for signing CRLs, and for signing (final) certificates (as a CA) (critical V3 extensions). The validity of the certificate is from 2024-03-13 to 2027-03-12. This certificate has been signed using SHA2-256 as a hash function and the RSA-4096 private key ($PR_D$) of ISRG Root X1 (the public exponent of this key is also $2^{16} + 1$).
- Finally, the corresponding public key ($PU_D$) of ISRG Root X1 is certified in a self-signed X.509v3 certificate ($CA_D \ll D \gg$). This certificate states this key pair is intended for signing, for signing CRLs, and for signing (final or intermediate) certificates (as a CA) (critical V3 extensions). The validity of the certificate is from 2015-06-04 to 2035-06-04. This certificate has been signed using SHA2-256 as a hash function.
- All signatures use PKCS #1 v1.5 formatting.

**Explain why the different security choices (algorithms, key lengths, validity periods) do or don't make sense.**

**Explain why you would keep or change these security choices.**

## Answer

### Certificate Chain Overview (ch3.2 p.28–29)

Three levels:
1. **Leaf (www.cozo.be)**: RSA-4096, valid ~3 months (Mar–Jun 2026), signed by Let's Encrypt R12
2. **Intermediate CA (R12 / Let's Encrypt)**: RSA-2048, valid ~3 years (2024–2027), signed by ISRG Root X1
3. **Root CA (ISRG Root X1)**: RSA-4096 (self-signed), valid 20 years (2015–2035)

---

### Analysis of Security Choices

#### 1. Leaf Certificate — RSA-4096, 3-Month Validity

**RSA-4096 for leaf**: RSA-4096 provides approximately 140-bit classical security — substantially more than necessary for a 3-month certificate. For a leaf certificate valid for only 90 days, RSA-2048 (112-bit security) or RSA-3072 (128-bit security) would be entirely sufficient. RSA-4096 incurs unnecessary computational overhead: every TLS handshake must perform an RSA-4096 operation (signature verification by the client), which is approximately 4–8× slower than RSA-2048 verification. For a healthcare platform potentially serving many concurrent users, this is an unnecessary performance cost.

This is possibly an intentional conservative choice given that CoZo handles medical data (sensitive, long-term confidentiality) — but the key size is tied to the certificate lifetime, not to the sensitivity of the data it protects. The data itself is encrypted with AES (symmetric), not RSA.

**3-month validity**: this is excellent security practice (ch3.2 p.50–53). Short validity periods limit the exposure window if $PR_W$ is compromised — the certificate expires within 90 days regardless. Let's Encrypt's automation model makes frequent renewal operationally trivial. Modern industry practice is moving toward 90-day maximum for leaf certificates. This choice **makes sense**.

**Key usage (CA:FALSE, signing + key encipherment)**: correct. The leaf certificate cannot sign sub-certificates, limiting the blast radius of a compromise. The "key encipherment" usage is slightly outdated in the TLS 1.3 context (TLS 1.3 no longer uses RSA for key encipherment, only for signing via CertificateVerify), but is harmless. **Makes sense**.

#### 2. Intermediate CA — RSA-2048, 3-Year Validity (2024–2027)

**RSA-2048**: an intermediate CA valid until 2027 using RSA-2048 is approaching the limit of NIST's recommendation (RSA-2048 adequate through 2030). For a 3-year certificate expiring in 2027, RSA-2048 is technically within the acceptable range — just barely. Given that this is an actively used CA that may sign thousands of leaf certificates before 2027, the risk is manageable but not ideal.

**Assessment**: RSA-2048 for the intermediate CA **marginally makes sense** given the 2027 expiry, but RSA-3072 would have been a more comfortable choice (128-bit security adequate well beyond 2027). It would be reasonable to replace R12 with an RSA-3072 or RSA-4096 intermediate before renewal.

**3-year validity**: intermediate CA certificates must be distributed in TLS handshakes (they cannot typically be pre-installed in all clients). A 3-year lifetime requires periodic renewal (2024–2027 → replace with new intermediate) but is operationally manageable. Let's Encrypt typically rotates its intermediates before expiry to maintain continuity. **Makes sense**.

**Key usage (CA:TRUE for final certificates only, CRL signing)**: correctly constrains the intermediate to signing leaf certificates and CRLs but not sub-CAs. This path-length constraint prevents the intermediate from issuing sub-CAs, limiting the PKI hierarchy depth. **Makes sense**.

#### 3. Root CA — RSA-4096, 20-Year Validity (2015–2035)

**RSA-4096**: appropriate for a root CA valid until 2035. RSA-4096 provides a comfortable security margin for the remaining 9-year lifetime (as of 2026). Root CAs should use the largest practical key sizes because their compromise would be catastrophic (an attacker with $PR_D$ can issue certificates for any domain) and because their keys must remain secure for the full lifetime. **Makes sense**.

**20-year validity (2015–2035)**: root CAs require long lifetimes because pre-installing them in browser/OS trust stores takes years and cannot be rushed. A 20-year root CA lifetime is shorter than some older roots (GlobalSign's legacy root has ~30 years). For a root issued in 2015, validity through 2035 is reasonable — it gives plenty of time for successor roots to be distributed before expiry. **Makes sense**.

**Key usage (CA:TRUE for all certificates including sub-CAs, CRL signing)**: correct — the root CA can issue both intermediate CAs and leaf certificates (though in practice it should only issue intermediates). **Makes sense**.

#### 4. SHA-256 (All Levels)

SHA-256 provides 128-bit collision resistance (ch2.2.3 p.30–40). For certificates valid through 2027 and 2035, SHA-256 is appropriate and sufficient. SHA-1 is deprecated (see Question 18). SHA-256 is the industry standard and correct choice for all current certificates. **Makes sense** at all levels.

For the root CA valid until 2035 (~9 years remaining), SHA-384 would provide additional security margin (192-bit collision resistance), but SHA-256 is still considered adequate for this timeframe.

#### 5. PKCS #1 v1.5 Padding (All Levels)

This is the most problematic choice across all three levels. PKCS #1 v1.5 lacks a tight provable security reduction for signatures — RSA-PSS is the modern, provably secure alternative (ch2.2.3 p.85). TLS 1.3 mandates RSA-PSS for all certificate signatures in new deployments (ch3.6 p.7–8).

For all three levels in this chain: **PKCS #1 v1.5 does not make sense** and should be replaced with RSA-PSS. This is straightforward for new certificates and renewals — Let's Encrypt has supported RSA-PSS in issued certificates.

#### 6. Comparison: Leaf RSA-4096 vs Intermediate RSA-2048

There is a notable inconsistency: the leaf certificate (www.cozo.be) uses RSA-4096 while the intermediate CA (R12) that signed it uses RSA-2048. The leaf is stronger than the intermediate CA.

The security of the chain is limited by its weakest link. A valid certificate chain up to the root has the security of the weakest signature in the chain. Here, the intermediate CA's RSA-2048 signature on the leaf certificate is the binding link — an attacker who could forge RSA-2048 signatures (if RSA-2048 is broken) would be able to issue fraudulent leaf certificates regardless of whether the leaf itself uses RSA-4096. The leaf's RSA-4096 key does not compensate for the intermediate's RSA-2048.

**Assessment**: the RSA-4096 leaf under an RSA-2048 intermediate creates a false sense of security. The leaf's key strength should not exceed the intermediate CA's key strength — the chain is only as strong as its weakest link. RSA-3072 or RSA-4096 for the intermediate would be more consistent. **This choice does not fully make sense**.

---

### Summary of Changes

| Choice | Assessment | Action |
|---|---|---|
| Leaf RSA-4096 | Overkill for 3-month cert; inconsistent with RSA-2048 intermediate | Reduce to RSA-3072 or match intermediate strength; or upgrade intermediate to RSA-4096 |
| Leaf 3-month validity | Excellent — short, automate renewal | Keep |
| Intermediate RSA-2048 | Borderline for 2027 expiry; weaker than leaf (inconsistent) | Upgrade to RSA-3072 on next renewal |
| Intermediate 3-year validity | Reasonable for Let's Encrypt intermediate rotation | Keep |
| Root RSA-4096 | Appropriate for 2035 expiry | Keep |
| Root 20-year validity | Appropriate for trust anchor lifetime | Keep |
| SHA-256 (all levels) | Correct and current | Keep |
| PKCS #1 v1.5 (all levels) | Outdated; no tight security proof | **Change all levels to RSA-PSS** |
| Leaf CA:FALSE | Correct | Keep |
| Intermediate CA:TRUE (end-entity only) | Correct | Keep |

**Most important change**: replace PKCS #1 v1.5 with RSA-PSS across all three levels — this is the single most impactful security improvement and is straightforward to implement on certificate renewal.

**Second improvement**: upgrade the intermediate CA from RSA-2048 to RSA-3072 on its next renewal in 2027 to eliminate the chain inconsistency and extend the comfortable security margin.

### Sources

- IS_UG_3_2_Appl_AuthMeth (p.28–29: X.509v3 structure and extensions; p.50–53: validity periods and revocation)
- IS_UG_2_2_3_SecM_HashMac (p.77–85: RSA key sizes and padding; p.30–40: SHA-256 security)
- IS_UG_3_6_Appl_TLS (p.7–8: TLS 1.3 and RSA-PSS requirement)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16: key length recommendations)

_Status: Complete_  
_Done by: William_
