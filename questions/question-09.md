# Question 9

The TLS 1.3 handshake achieves mutual agreement on keys while authenticating the server.

**Explain how server authentication is achieved in the TLS 1.3 handshake.**

**What is the purpose of the Certificate, CertificateVerify, and Finished messages?**

**Why is the Finished message cryptographically essential even after certificate verification?**

## Answer

### Server Authentication in TLS 1.3 (ch3.6 p.7–8)

Server authentication in TLS 1.3 is achieved through a three-message sequence — Certificate, CertificateVerify, and Finished — that collectively establish three distinct cryptographic properties:
1. The server's public key is certified by a trusted CA (from Certificate)
2. The server holds the corresponding private key (from CertificateVerify)
3. The server derived the same session keys from the same handshake (from Finished)

All three properties must hold simultaneously for authentication to succeed. No single message alone is sufficient.

---

### The Certificate Message

**Purpose**: the server sends its X.509v3 certificate chain (ch3.2 p.28–29), from the server's end-entity certificate up to (but typically not including) the trusted root CA.

**What it proves**: the server's public key $PU_S$ has been certified by a CA that the client already trusts (pre-installed in the client's trust store). The certificate contains:
- The server's public key $PU_S$
- The domain name(s) the server is authorised to serve (Subject Alternative Name extension)
- Validity period (the client checks that the current date is within validity)
- CA signature over the certificate contents

The client performs **certificate chain validation** (ch3.2 p.44–53):
1. Verify the CA's signature on the server certificate using the intermediate CA's public key
2. Verify the intermediate CA's certificate using the root CA's public key
3. Confirm the root CA is in the client's trusted root store
4. Check that no certificate in the chain is expired
5. Check that no certificate has been revoked (via CRL or OCSP, ch3.2 p.50–53)

**What Certificate does NOT prove**: that the entity sending this certificate actually holds the private key $PR_S$ corresponding to $PU_S$. Any party that obtained a copy of the server's certificate (a public document) could send it. Possession of the public key certificate alone proves nothing about key ownership.

---

### The CertificateVerify Message

**Purpose**: the server proves it holds the private key $PR_S$ corresponding to the certified public key $PU_S$, and binds this proof to the specific handshake transcript.

**Mechanism**: the server computes a digital signature over the **handshake transcript hash** $H_{transcript}$ (the hash of all handshake messages up to and including the Certificate message):
$$\sigma_S = Sign(PR_S,\ \text{"TLS 1.3, server CertificateVerify"} \| H_{transcript})$$

The string prefix ("TLS 1.3, server CertificateVerify") is a context label that domain-separates this signature from other potential uses of the same key, preventing cross-protocol attacks.

The client verifies: $Verify(PU_S,\ \text{same prefix} \| H_{transcript},\ \sigma_S)$.

**What CertificateVerify proves**:
1. **Key ownership**: the server has $PR_S$ — only the holder of the private key can produce a valid signature verifiable with $PU_S$
2. **Transcript binding**: the signature covers $H_{transcript}$, which includes the client random, server random, and both parties' DH key shares. An attacker who replays a CertificateVerify from a previous session would fail: the transcript hash includes session-specific values ($r_C$, $r_S$, $\hat{C}$, $\hat{S}$) that differ in every session (ch3.1 p.3)
3. **Binding certificate to DH exchange**: the signature covers the DH parameters — it is impossible for a MITM to present a valid certificate and swap the DH values, because the DH values are included in the transcript that the server signed

**Analogy**: Certificate says "this person's name is in the passport." CertificateVerify says "and I can prove it's my passport by signing with my private key."

---

### The Finished Message

**Purpose**: provides **key confirmation** — proves that the server derived the same handshake keys from the same shared ECDH secret, and that the entire negotiated handshake (including all extensions, cipher suite selection, and key shares) is intact.

**Mechanism**:
$$Finished_S = HMAC(finished\_key_S,\ H_{full\_transcript})$$
where $finished\_key_S$ is derived from the **handshake secret** (which itself derives from the ECDH shared secret $Z = cs \cdot G$). The full transcript includes all handshake messages up to and including the server's CertificateVerify.

The client verifies the Finished by independently computing the same HMAC using its own derived $finished\_key_S$.

---

### Why Finished Is Cryptographically Essential Even After CertificateVerify

CertificateVerify answers the question: "Did the legitimate server sign this transcript?" Finished answers a different and equally essential question: "Did the server and client arrive at the same session keys?"

**Three reasons Finished is essential**:

**Reason 1 — Key confirmation / proof of DH success**

CertificateVerify is a signature using the server's long-term RSA key — a key whose validity the client verified against the CA hierarchy. This proves the server identity. But it does not prove the server successfully computed the ECDH shared secret and derived the same handshake keys as the client.

The Finished message is computed using $finished\_key_S$, which derives from the ECDH shared secret $Z$. If the server failed to compute $Z$ correctly (e.g., because a MITM tampered with the DH key share before ServerHello, causing the server and client to compute different shared secrets), the server would derive a different $finished\_key_S$ and produce a wrong Finished. The client, computing the expected Finished independently, would detect the mismatch and abort.

In short: CertificateVerify proves server identity; Finished proves that the key exchange produced the same result on both sides.

**Reason 2 — Handshake integrity confirmation**

The HMAC in Finished covers the full handshake transcript hash, including all negotiated parameters: cipher suite, key share groups, extensions, DH values, and the random nonces. This provides end-to-end integrity for the entire handshake negotiation (ch3.1 p.7 — MAC for integrity).

If an active attacker tampered with any handshake message — for example, downgrading the cipher suite to a weaker one — the transcript hash would differ between client and server (they would have seen different messages), and the Finished HMAC would not match. The client would abort. CertificateVerify only covered the transcript up to the Certificate message; Finished covers everything.

**Reason 3 — Binding identity to keys (prevents identity-mismatch attack)**

Without Finished, it is theoretically conceivable that a MITM might somehow forward a valid CertificateVerify (from a legitimate server's signature on one transcript) while establishing different session keys with the client. The Finished MAC, keyed with material derived from the ECDH secret, ensures that the same entity that performed the DH exchange also holds the key material. The identity proved by CertificateVerify and the key material proved by Finished are cryptographically linked: both are over the same transcript, and Finished additionally uses the ECDH output.

**Summary of the three-message chain**:

| Message | Proves | Mechanism |
|---|---|---|
| Certificate | Server's public key is trusted by CA | X.509 chain validation to trusted root |
| CertificateVerify | Server holds private key; signature bound to this handshake transcript | RSA-PSS signature on transcript hash; transcript includes fresh nonces |
| Finished | Server derived same session keys; full handshake is untampered | HMAC keyed with material from ECDH shared secret over full transcript hash |

All three are necessary and none is sufficient alone. The combination provides authenticated key exchange: the client is certain that it shares session keys with the legitimate server and that no aspect of the handshake was tampered with.

### Sources

- IS_UG_3_6_Appl_TLS (p.7–8: TLS 1.3 handshake messages and purposes; p.18: ECDHE in TLS 1.3)
- IS_UG_3_2_Appl_AuthMeth (p.28–29: X.509 certificate chain; p.44–53: certificate validation and revocation)
- IS_UG_3_1_Appl_Basics (p.3: nonces for freshness; p.7: MAC for integrity)

_Status: Complete_  
_Done by: William_
