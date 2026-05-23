# Question 18

When looking at the root certificates of my browser (Firefox, May 2026) I found the following properties for the GlobalSign Root CA X.509v3 certificate:

- The certificate is valid from 1998-09-01 to 2028-01-28.
- The public key is a 2048 bit RSA key (the public exponent is $2^{16} + 1$).
- The certificate is self-signed using SHA-1 as a hash function and using PKCS #1 v1.5 formatting.
- The critical extensions mention this is a certificate for a certificate authority and the key pair is intended for certificate signing and for CRL signing.

**Explain why the different security choices (algorithms, key lengths, validity periods, extensions) do or don't make sense.**

**Explain why you would keep or change these security choices.**

## Answer

### Context: Root CA Certificate Analysis (ch3.2 p.28–29)

This GlobalSign root CA was originally issued in 1998 — extremely early in the commercial internet's history. Many of its security choices reflect what was considered strong in 1998. Evaluating them requires understanding both their historical context and their current acceptability.

---

### Analysis of Each Security Choice

#### 1. RSA-2048 Public Key

**Historical context**: RSA-2048 was not standard in 1998 — the original GlobalSign roots used RSA-1024. The 2048-bit version was issued later as part of a security upgrade. RSA-2048 provides approximately 112-bit classical security (ch2.2.3 p.80), which NIST considered adequate until approximately 2030.

**Current status**: RSA-2048 is now approaching the end of its recommended lifetime. NIST's key management guidelines place RSA-2048 as acceptable through 2030. For a certificate **valid until 2028**, RSA-2048 is still within the margin of acceptability — just barely. However, given post-quantum developments (ch2 PQCrypto p.16), Grover's algorithm does not directly threaten RSA (Shor's algorithm does), and the timeline for cryptographically relevant quantum computers attacking RSA-2048 is uncertain but plausible within a decade.

**Public exponent $e = 65537 = 2^{16} + 1$**: the standard choice (ch2.2.3 p.77). Efficient for verification (few multiplications), avoids small-exponent attacks. Correct.

**Assessment**: RSA-2048 with validity to 2028 is marginally acceptable but conservative. It is **not recommended for any new certificates** — RSA-3072 or RSA-4096 would be more appropriate for the remaining lifetime.

#### 2. SHA-1 Signature Hash Function

This is the most critical problem. SHA-1 was deprecated for digital signature use due to practical collision attacks (ch2.2.3 p.14). A SHA-1 collision attack has been demonstrated (SHAttered, 2017 — two different inputs producing the same SHA-1 hash). For certificate signatures, a collision attack could allow an attacker to craft a fraudulent certificate with the same SHA-1 hash as a legitimate one, potentially undermining the signature's validity.

**For CA certificates specifically**: the CA signature over a certificate's content uses the hash function to produce the value that is signed with the CA's private key. If an attacker can find two certificate payloads with the same SHA-1 hash, they might be able to get the CA to sign one while using the signature for the other (chosen-prefix collision attack). This is particularly dangerous for CA certificates.

**However, for a self-signed root CA certificate**: the root CA is a **trust anchor** in the browser's trust store — it is trusted absolutely, not because of its signature but because it is pre-installed and trusted by policy. The self-signature on a root CA certificate is verified against the root's own public key — an attacker would need to compromise the private key to forge it regardless. The SHA-1 self-signature of a root CA is therefore of lower direct risk than SHA-1 in intermediate or leaf certificates.

Nevertheless: SHA-1 should not be used in any new certificate, and browsers actively distrust SHA-1 signed certificates in non-root positions. Using SHA-1 for a root CA self-signature is a historical artefact that should be modernised.

**Assessment**: SHA-1 in this root CA **does not make sense** for 2026. It should be replaced with SHA-256 (or SHA-384 for extra margin).

#### 3. PKCS #1 v1.5 Padding for Signatures

PKCS #1 v1.5 is the older padding scheme. For signatures specifically, it lacks a tight provable security reduction to the RSA hardness assumption — RSA-PSS (Probabilistic Signature Scheme) is the modern, provably secure alternative (ch2.2.3 p.85). TLS 1.3 mandates RSA-PSS for new certificates (ch3.6 p.7–8).

For a self-signed root CA, the same argument applies as for SHA-1: the self-signature's integrity depends on the private key being secure, not primarily on the signature scheme. However, from a forward-looking and standards-compliance perspective, PKCS #1 v1.5 should be replaced with RSA-PSS.

**Assessment**: PKCS #1 v1.5 **does not make sense** for any new certificate in 2026. Change to RSA-PSS.

#### 4. Validity Period (1998–2028, approximately 30 years)

For a root CA, long validity periods are a practical necessity (ch3.2 p.50–53): root certificates are pre-installed in browser trust stores and operating systems. Updating them across billions of devices takes years of coordinated effort. A root CA with a short validity period would require emergency re-distribution of trust anchors every few years — operationally infeasible at scale.

The 30-year lifetime (1998–2028) was ambitious even in 1998. By 2026, the certificate is 28 years old — its RSA-2048 key has been in use for nearly three decades. The long lifetime creates a risk: if RSA-2048 is broken before 2028 (through advances in classical or quantum computing), the root is compromised for the remainder of its life.

Modern root CAs typically have 25–30 year validities, which is consistent with this certificate. The validity period itself is not unusual for a root CA issued in the 1990s.

**Assessment**: the long validity period **made sense** when issued and is typical for root CAs. However, the age of the certificate (28 years in 2026) combined with RSA-2048 and SHA-1 creates cumulative risk that would ideally be addressed by issuing a new root CA before the 2028 expiry.

#### 5. Key Usage Extensions

The critical extensions correctly specify:
- **CA: TRUE**: the certificate can sign other certificates — correct for a root CA (ch3.2 p.28–29)
- **Key usage: certificate signing + CRL signing**: the root's key pair is used to sign issued certificates and revocation lists — the correct and complete set of permissions for a root CA

**Assessment**: the extensions are **correctly configured**.

---

### What to Keep and What to Change

| Choice | Assessment | Action |
|---|---|---|
| RSA-2048 key | Borderline acceptable until 2028 expiry | Replace with RSA-4096 or ECDSA P-384 in next root CA |
| Public exponent 65537 | Correct | Keep |
| SHA-1 signature | Deprecated; collision attacks demonstrated | **Change to SHA-256 or SHA-384 immediately** |
| PKCS #1 v1.5 | Not provably secure; superseded by RSA-PSS | **Change to RSA-PSS** |
| Validity 1998–2028 | Typical for legacy root CAs; nearly expired | Let expire; issue replacement with shorter lifetime |
| Key usage extensions | Correctly configured | Keep |

**Recommended action for 2026**: this root CA should be approaching planned retirement. A replacement root CA should already be in production, issued with:
- RSA-4096 or ECDSA P-384 (ch2.2.3 p.85–87)
- SHA-384 signature hash
- RSA-PSS or ECDSA signature scheme
- 20–25 year validity from issuance date
- Identical CA:TRUE and key usage extensions

Browsers should have already begun cross-signing the replacement root CA from this legacy root so that trust transitions smoothly before the 2028 expiry.

**Overall assessment of the certificate**: entirely acceptable for its era (1998), now significantly outdated. The SHA-1 + PKCS #1 v1.5 combination is the most problematic pair, though for a self-signed root trust anchor the direct attack risk is lower than for intermediate/leaf certificates. The RSA-2048 key is approaching the end of its recommended lifetime. All three should be modernised in the successor root certificate.

### Sources

- IS_UG_3_2_Appl_AuthMeth (p.28–29: X.509v3 certificate structure; p.50–53: certificate validity and revocation)
- IS_UG_2_2_3_SecM_HashMac (p.14: SHA-1 collision resistance failures; p.77–85: RSA key sizes, padding schemes, RSA-PSS)
- IS_UG_3_6_Appl_TLS (p.7–8: TLS 1.3 and RSA-PSS requirement)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16: key length recommendations in quantum context)

_Status: Complete_  
_Done by: William_
