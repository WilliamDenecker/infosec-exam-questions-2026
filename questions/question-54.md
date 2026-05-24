# Question 54

When browsing the website of Bank A you can observe that the website is secured using TLS 1.2, using elliptic curve ephemeral Diffie-Hellman for the key exchange, RSA-2048 authentication for the handshake (the certificate is signed with SHA-2-256), and AES-128-GCM for the encryption of further traffic.

The website of Bank B uses slightly different security parameters: it uses TLS 1.3 with RSA-2048 authentication for the handshake (the certificate is signed with SHA-2-256), and AES-256-GCM for the encryption of further traffic.

**Compare the security of the websites for Banks A and B. How significant are the differences? Which bank website offers a better level of security?**

## Answer

### The Two Configurations

| Parameter | Bank A | Bank B |
|---|---|---|
| TLS version | TLS 1.2 | TLS 1.3 |
| Key exchange | ECDHE (elliptic curve ephemeral DH) | DHE/ECDHE (TLS 1.3 mandatory) |
| Server authentication | RSA-2048, certificate signed SHA-256 | RSA-2048, certificate signed SHA-256 |
| Symmetric encryption | AES-128-GCM | AES-256-GCM |

---

### 1. TLS Version: 1.2 vs 1.3 (ch3.6 p.5, p.7–8)

**TLS 1.3** (Bank B) is a significant improvement over TLS 1.2 (ch3.6 p.7–8):

- **Removed weak cipher suites**: TLS 1.3 eliminates RSA key exchange (no forward secrecy), static DH, RC4, DES, 3DES, CBC-mode ciphers with HMAC, MD5, SHA-1 — all of which remain negotiable in TLS 1.2 if misconfigured
- **Mandatory forward secrecy**: TLS 1.3 requires ECDHE or DHE in all handshakes — no server-side static key exchange is possible; forward secrecy is a mandatory property, not a configuration option
- **Encrypted handshake**: TLS 1.3 encrypts more of the handshake (Certificate, CertificateVerify, Finished are all encrypted); TLS 1.2 sends Certificate in the clear, leaking the server's identity to passive observers
- **Fewer round trips**: TLS 1.3 completes the handshake in one round trip (1-RTT) vs TLS 1.2's two (2-RTT), with optional 0-RTT for resumed sessions
- **No renegotiation**: TLS 1.3 removes the renegotiation mechanism, which was a source of vulnerabilities in TLS 1.2 (e.g., TLS renegotiation attack)
- **No compression**: TLS 1.3 removes TLS-layer compression (which enabled CRIME and BREACH attacks in TLS 1.2)

**Assessment**: Bank B's use of TLS 1.3 provides a materially more secure protocol, primarily because it eliminates an entire class of configuration errors and legacy weaknesses that remain possible in TLS 1.2. The difference is **significant** from a protocol hardening perspective.

---

### 2. Key Exchange: ECDHE (TLS 1.2) vs TLS 1.3 (ch3.6 p.7–8; ch2.2.4 p.11)

Bank A explicitly uses **ECDHE** in TLS 1.2. Bank B uses TLS 1.3, which mandates DHE/ECDHE.

**Both provide forward secrecy**: ECDHE generates a fresh ephemeral scalar per session; discarding the session key after use means a future compromise of the server's RSA private key cannot retroactively decrypt past sessions.

**In practice, no security difference**: Bank A has chosen ECDHE explicitly (which is the same family of key exchange as TLS 1.3 mandates). The only advantage Bank B has here is that TLS 1.3 cannot fall back to a non-forward-secret key exchange even if an attacker modifies the ClientHello — Bank A's TLS 1.2 could in principle be downgraded to RSA key exchange if the server is misconfigured.

**Assessment**: functionally equivalent if Bank A's server is correctly configured to require ECDHE. TLS 1.3 (Bank B) provides additional protection against downgrade attacks.

---

### 3. Server Authentication: RSA-2048 with SHA-256 (ch3.2 p.28–29, p.50–53; ch2.2.2 p.14)

**Both banks use identical authentication**: RSA-2048 certificate signed with SHA-256.

- RSA-2048 is currently considered secure against classical computers. It is recommended until at least 2030 by NIST (ch3.2 p.50–53)
- SHA-256 for the certificate signature is appropriate and not deprecated
- Post-quantum: both RSA-2048 signatures are equally broken by Shor's algorithm (ch2.2 PQCrypto p.12, p.18) — this is a shared weakness

**Assessment**: **no difference** between Bank A and Bank B on authentication. Both face the same post-quantum threat.

---

### 4. Symmetric Encryption: AES-128-GCM vs AES-256-GCM (ch2.2.3 p.70–75; ch2.2 PQCrypto p.16)

**Classical security**: 
- AES-128-GCM: 128-bit security against classical adversaries — there are no known practical attacks
- AES-256-GCM: 256-bit security against classical adversaries — equally unbroken
- **For current classical threats: no meaningful difference** — neither is remotely close to being broken

**Post-quantum security** (ch2.2 PQCrypto p.16):
- Grover's algorithm halves the effective key length: AES-128 → 64-bit security (obsolete); AES-256 → 128-bit security (acceptable)
- AES-128-GCM (Bank A) would provide only 64-bit post-quantum security — **insufficient** after a quantum computer arrives
- AES-256-GCM (Bank B) retains 128-bit post-quantum security — **sufficient** (ch2.2 PQCrypto p.16)

**Assessment**: for **current classical security** there is no practical difference. For **post-quantum future-proofing**, AES-256 (Bank B) is significantly more durable. The significance depends on the threat model: 0 for today's threats, material for quantum-aware threat modelling.

---

### 5. Authentication Tag (GCM): Both Use AES-GCM

Both banks use GCM mode, which provides authenticated encryption (ch2.2.3 p.70–75) — confidentiality and integrity in a single primitive. The 128-bit authentication tag is the same length in both AES-128-GCM and AES-256-GCM. No difference in integrity protection.

---

### Overall Assessment

**Which bank is more secure?** Bank B (TLS 1.3 + AES-256-GCM) is marginally to moderately more secure, for two reasons:

1. **TLS 1.3 eliminates legacy weaknesses**: Bank A's TLS 1.2 can in principle negotiate weaker cipher suites, use RSA key exchange if misconfigured, and leaks more handshake data to passive observers. TLS 1.3 removes these risks by design.

2. **AES-256 provides post-quantum readiness**: Bank A's AES-128 would be inadequate after a quantum computer breakthrough; Bank B's AES-256 maintains sufficient security.

**How significant are the differences?**

| Aspect | Difference significance |
|---|---|
| TLS 1.3 vs TLS 1.2 (with ECDHE) | Moderate — better protocol hardening, fewer attack surface, encrypted handshake |
| Mandatory forward secrecy | Low (Bank A explicitly uses ECDHE) — effectively equivalent |
| AES-128 vs AES-256 | Low for current threats; High post-quantum |
| RSA-2048 authentication | None — identical |

**In practice today**: both banks are providing **reasonable security** for current classical threats. A well-configured Bank A (TLS 1.2 + ECDHE + AES-128-GCM) is not materially weaker than Bank B for today's adversaries. The advantages of Bank B are principally:
- Protocol discipline (TLS 1.3 removes legacy weaknesses by design rather than configuration)
- Post-quantum preparedness for symmetric encryption

For a complete security assessment, one would also need to examine: certificate chain quality (CA selection, validity period, key usage), HSTS/HPKP headers, cipher suite ordering, and implementation-level security (which is invisible from TLS version and cipher suite negotiation alone).

### Sources

- IS_UG_3_6_Appl_TLS (p.5: TLS overview; p.7–8: TLS 1.3 — cipher suite restrictions, mandatory ECDHE, encrypted handshake, 1-RTT; p.37: key exchange and authentication)
- IS_UG_2_2_3_SecM_HashMac (p.70–75: AES-GCM — authenticated encryption, 128-bit tag, confidentiality + integrity)
- IS_UG_2_2_2_SecM_AsymmEncr (p.14: RSA key length security; p.34: public exponent choice)
- IS_UG_3_2_Appl_AuthMeth (p.28–29: X.509 certificates and RSA-2048; p.50–53: certificate security and key strength recommendations)
- IS_UG_2_2_SecM-adv-PQCrypto (p.12: Shor's breaks RSA; p.16: AES-128 → 64-bit post-quantum, AES-256 → 128-bit post-quantum)

_Status: Complete_  
_Done by: William_
