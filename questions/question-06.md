# Question 6

**Describe (with sufficient detail) the TLS (version 1.3) handshake protocol for a client with no certificate and a server using an RSA key (for digital signature only), where elliptic curve ephemeral Diffie-Hellman is used for the key exchange. Explain the mechanisms that are used against possible replay attacks.**

**What are the advantages and drawbacks of using a combination of ephemeral Diffie-Hellman key exchange and RSA digital signature compared to a situation where RSA would be used for both key exchange and authentication (which was possible in TLS version 1.2)?**

## Answer

### TLS 1.3 Handshake Protocol (ch3.6 p.7–8)

The setup: client has no certificate (client authentication is not required, the common case for web browsing). Server has an RSA key pair used for digital signatures only. Key exchange uses ECDHE (Elliptic Curve Ephemeral Diffie-Hellman, ch3.6 p.18).

---

**Phase 1 — ClientHello** (client → server, plaintext)

The client sends:
- Supported TLS version(s): TLS 1.3
- Supported cipher suites (e.g., `TLS_AES_256_GCM_SHA384`)
- Supported key share groups (e.g., X25519, P-256)
- **Client's ephemeral ECDH public key share** $\hat{C} = c \cdot G$ for each supported group (client generates ephemeral scalar $c$ and sends the corresponding point)
- **Client random**: a 32-byte random nonce $r_C$ (fresh per handshake)
- Extensions (e.g., supported signature algorithms, SNI)

---

**Phase 2 — ServerHello** (server → client, plaintext)

The server selects the cipher suite and key share group, then sends:
- Selected cipher suite and TLS version
- **Server's ephemeral ECDH public key share** $\hat{S} = s \cdot G$ (server generates ephemeral scalar $s$)
- **Server random**: a 32-byte random nonce $r_S$ (fresh per handshake)
- Session ID / key share extension

**At this point, both parties independently compute the shared secret**:
- Client: $Z = c \cdot \hat{S} = c \cdot (s \cdot G) = cs \cdot G$
- Server: $Z = s \cdot \hat{C} = s \cdot (c \cdot G) = cs \cdot G$

From $Z$ (specifically its $x$-coordinate $Z_x$), both parties derive the **handshake secret** and from it the **handshake traffic keys** using the TLS 1.3 key schedule (HKDF applied to the transcript hash). All subsequent messages from the server are encrypted with these keys.

---

**Phase 3 — Server's Encrypted Extensions, Certificate, CertificateVerify, Finished** (all encrypted with handshake key)

**(a) EncryptedExtensions**: server sends additional negotiated extensions (ALPN, etc.) encrypted.

**(b) Certificate**: server sends its X.509 certificate chain (ch3.2 p.28–29). The certificate contains the server's RSA public key $PU_S$ and is signed by a CA the client trusts. The client verifies the certificate chain up to a trusted root CA.

**(c) CertificateVerify**: server computes a digital signature over the **handshake transcript hash** (hash of all handshake messages up to and including the Certificate message):
$$\sigma_S = RSA\text{-}PSS\text{-}Sign(PR_S,\ H_{transcript})$$
The client verifies $\sigma_S$ using the server's RSA public key $PU_S$ from the certificate.

This proves two things:
1. The server holds the private key $PR_S$ corresponding to the certified public key $PU_S$ — confirming the server is the legitimate owner of the certificate
2. The signature is bound to the specific handshake transcript — a replay of a signature from a different session would fail because the transcript hash includes session-specific values ($r_C$, $r_S$, the ephemeral DH shares)

**(d) Finished**: server computes:
$$Finished_S = HMAC(finished\_key_S,\ H_{transcript\_including\_CertVerify})$$
where $finished\_key_S$ is derived from the handshake secret. The client verifies this HMAC.

Finished serves as **key confirmation**: it proves the server derived the same handshake keys from the same shared secret. If a MITM had tampered with the DH parameters, the server would have derived different keys and its Finished would be wrong.

---

**Phase 4 — Client Finished** (client → server, encrypted)

The client verifies the server's Finished, then computes and sends its own:
$$Finished_C = HMAC(finished\_key_C,\ H_{full\_handshake\_transcript})$$

Both parties then compute the **application traffic keys** from the master secret. The handshake is complete; application data exchange begins under symmetric encryption (AES-256-GCM with the negotiated cipher suite) (ch3.6 p.7–8).

---

### Replay Attack Prevention (ch3.1 p.3, ch3.1 p.7)

TLS 1.3 uses several mechanisms to prevent replay attacks:

**1. Fresh random nonces ($r_C$, $r_S$)**: both the client random and server random are 32-byte values generated freshly per handshake (ch3.1 p.3). These are included in all transcript hashes. A recording of a previous handshake has different $r_C$ and $r_S$ values — replaying it to a new server would not result in a valid Finished because the transcript hash would differ.

**2. Ephemeral DH key shares**: the ephemeral scalars $c$ and $s$ are discarded after each session. Even if an attacker replays $\hat{C}$ from a previous session, the server will choose a new ephemeral $s'$, producing a different shared secret $Z'$. The replayed handshake cannot establish the same session keys.

**3. CertificateVerify transcript binding**: the server's signature $\sigma_S$ covers the hash of the specific handshake transcript, including $r_C$, $r_S$, $\hat{C}$, $\hat{S}$. A signature from a different session has a different transcript hash and will fail verification.

**4. Finished HMAC transcript binding**: the Finished messages are HMACs keyed with keys derived from the session-specific shared secret. They bind the key confirmation to this specific handshake instance.

**Regarding 0-RTT (Early Data)**: TLS 1.3 supports 0-RTT session resumption where the client can send application data in the first flight (before the handshake completes). This feature uses a pre-shared key from a previous session and does NOT provide full replay protection — the server must accept the early data, and an attacker could replay it to a different server instance. 0-RTT is therefore only suitable for idempotent requests and requires application-level replay protection if used. The main handshake described above does not have this limitation.

---

### ECDHE + RSA Signing vs RSA for Both (TLS 1.2 Style)

**In TLS 1.2, RSA key exchange + RSA authentication** (ch3.6 p.5) worked as follows:
- Client uses the server's RSA public key $PU_S$ to encrypt a random pre-master secret: $C = RSA\text{-}Encrypt(PU_S, PMS)$
- Server decrypts: $PMS = RSA\text{-}Decrypt(PR_S, C)$
- Session keys are derived from PMS

**Advantage of this RSA-only approach**:
- Simplicity: one cryptographic primitive (RSA) handles both authentication and key establishment
- No DH group negotiation needed

**Disadvantage — no forward secrecy** (ch3.6 p.37):

This is the critical flaw. The pre-master secret $PMS$ is encrypted directly under the server's **long-term** RSA private key $PR_S$. If an adversary records all ciphertext traffic today and later obtains $PR_S$ (through a server compromise, legal compulsion, or cryptanalysis), they can retroactively decrypt all past sessions by:
1. Decrypt $C$ using $PR_S$ to recover $PMS$
2. Derive all session keys from $PMS$
3. Decrypt all recorded application data

This is a catastrophic retrospective breach — all sessions encrypted under the RSA key transport scheme are permanently at risk as long as the server's long-term key is ever compromised.

**Advantage of ECDHE + RSA signing (TLS 1.3)** — **forward secrecy** (ch3.6 p.37):

The session keys derive from the ephemeral DH shared secret $cs \cdot G$, where the ephemeral scalars $c$ and $s$ are discarded after the handshake. Even if $PR_S$ is later compromised:
- The attacker can verify the signature in CertificateVerify (they now have $PR_S$ / $PU_S$)
- But they cannot reconstruct $cs \cdot G$: they see $\hat{C} = c \cdot G$ and $\hat{S} = s \cdot G$ in the recorded traffic, but recovering $c$ or $s$ from these points requires solving the ECDLP — computationally infeasible

Past sessions are therefore permanently protected even after $PR_S$ is compromised.

| Criterion | RSA key exchange (TLS 1.2) | ECDHE + RSA signing (TLS 1.3) |
|---|---|---|
| Forward secrecy | **No** — compromised $PR_S$ decrypts all past sessions | **Yes** — ephemeral DH keys discarded; past sessions safe |
| Authentication | Via $PR_S$ (decryption proves key ownership) | Via RSA-PSS signature on transcript (explicit proof) |
| Key exchange security | Depends on RSA long-term key | Depends on ECDLP hardness (ephemeral) |
| Cipher agility | RSA determines key size/security | ECDHE and RSA decoupled; each sized independently |
| Complexity | Single primitive | Two primitives (ECDHE + RSA) |

**Drawbacks of ECDHE + RSA signing**:
- Slightly more complex: two distinct mechanisms must be implemented and negotiated correctly
- Both ECDHE group and RSA signature scheme must be agreed upon separately
- The server must perform an ECDH scalar multiplication (fast, but additional operation)

**Why the advantage outweighs the drawback**: the loss of forward secrecy in RSA key transport is a fundamental architectural risk — it converts a future server compromise into a retrospective breach of all past confidential communications. TLS 1.3 eliminated RSA key transport entirely for this reason. The additional complexity of ECDHE is modest compared to the security gain.

### Sources

- IS_UG_3_6_Appl_TLS (p.5: TLS 1.2 RSA key exchange; p.7–8: TLS 1.3 handshake; p.18: ECDHE key exchange; p.37: forward secrecy)
- IS_UG_3_1_Appl_Basics (p.3: nonces for replay prevention; p.7: challenge-response replay protection)
- IS_UG_3_2_Appl_AuthMeth (p.28–29: X.509 certificates; p.85–87: digital signatures)
- IS_UG_2_2_4_SecM_KeyExch (p.10: authenticated Diffie-Hellman and forward secrecy)

_Status: Complete_  
_Done by: William_
