# Question 19

(Perfect) Forward Secrecy for a communication channel means previously exchanged session keys remain secret, even when a fixed secret or private key that was used in the key exchange for these session keys is compromised.

**Why can't this be achieved using an RSA key exchange? How could an attacker who knows the compromised RSA private key decrypt previous key exchanges?**

**Which key exchange mechanisms do provide (Perfect) Forward Secrecy? What happens in these cases when the (fixed) private key used in the key exchange is compromised?**

## Answer

### Why RSA Key Exchange Cannot Provide Forward Secrecy (ch3.6 p.37)

In an RSA-based key exchange (as used in TLS 1.2 with the `TLS_RSA_*` cipher suites), the key establishment works as follows (ch3.6 p.5):

1. The client generates a random **pre-master secret** (PMS), typically 48 bytes.
2. The client encrypts the PMS under the server's long-term RSA public key $PU_S$:
   $$C = RSA\text{-}PKCS1\text{-}v1.5\text{-}Encrypt(PU_S,\ PMS)$$
3. The client sends $C$ to the server in the ClientKeyExchange message.
4. The server decrypts with its long-term private key $PR_S$: $PMS = RSA\text{-}Decrypt(PR_S,\ C)$.
5. Both parties derive the session keys from PMS using a PRF.

**The long-term private key $PR_S$ is used directly for key establishment.** The session keys are derived entirely from the PMS, which is encrypted under $PR_S$. This creates a direct, permanent cryptographic binding: anyone who later obtains $PR_S$ can retroactively compute every PMS from every recorded session.

**How an attacker decrypts previous sessions** (ch3.6 p.5, p.37):

An attacker who records all ciphertext traffic (a passive eavesdropper) and later obtains $PR_S$ (through server breach, legal compulsion, or cryptanalysis) can:

1. Retrieve the recorded ClientKeyExchange ciphertext $C$ from the traffic log.
2. Compute $PMS = RSA\text{-}Decrypt(PR_S,\ C)$ — now possible with the compromised private key.
3. Derive all session keys from $PMS$ using the same PRF the parties used.
4. Decrypt all recorded application data using the recovered session keys.

**This attack is retroactive**: the attacker did not need the private key at the time of the session — they can perform the decryption months or years later once they obtain $PR_S$. Any recorded session from the past is permanently at risk as long as $PR_S$ is ever compromised.

**Why forward secrecy is impossible with RSA key exchange**: the session key is encrypted directly under the long-term private key. There is no ephemeral component — the same $PR_S$ is used for every session, so every session's key material is permanently derivable from $PR_S$. The security of all past sessions depends entirely on the permanent secrecy of $PR_S$.

---

### Key Exchange Mechanisms That Provide Forward Secrecy (ch2.2.4 p.10, ch3.6 p.37)

Forward secrecy requires that session keys depend on **ephemeral (per-session) secrets** that are discarded after the session ends, not on long-term keys.

**Mechanism 1 — Ephemeral Diffie-Hellman (DHE)** (ch2.2.4 p.10):

For each session:
- Client generates ephemeral DH scalar $a$ (random, discarded after session)
- Server generates ephemeral DH scalar $b$ (random, discarded after session)
- Client sends $g^a \bmod p$; server sends $g^b \bmod p$ (both authenticated with long-term signatures)
- Shared secret: $g^{ab} \bmod p$
- Session key derived from $g^{ab} \bmod p$

The long-term private key $PR_S$ is used only to **sign** the server's ephemeral $g^b$ — it is used for authentication, not for key establishment. The session key depends on the ephemeral values $a$ and $b$, which are discarded immediately after the session.

**Mechanism 2 — Ephemeral Elliptic Curve Diffie-Hellman (ECDHE)** (ch3.6 p.18, ch2.2.4 p.10):

Same principle with elliptic curve arithmetic (see Question 4):
- Client generates ephemeral scalar $c$; computes $c \cdot G$; discards $c$ after session
- Server generates ephemeral scalar $s$; computes $s \cdot G$; discards $s$ after session
- Shared secret: $cs \cdot G$ (x-coordinate)
- Session key derived from shared secret

$PR_S$ signs the server's ephemeral ECDH public key for authentication (CertificateVerify in TLS 1.3, ch3.6 p.7–8), but does not participate in session key derivation.

**What Happens When $PR_S$ Is Compromised with DHE/ECDHE**:

| Consequence | RSA key exchange (no FS) | DHE/ECDHE (forward secrecy) |
|---|---|---|
| Past session keys recoverable? | **Yes** — attacker decrypts $C$ to get PMS | **No** — ephemeral values ($a, b$ or $c, s$) were discarded |
| Attacker can impersonate server in future? | Yes — can sign new DH values fraudulently | Yes — can now forge CertificateVerify signatures |
| Attacker can authenticate to clients as server? | Yes | Yes — must revoke certificate immediately |
| Attacker can decrypt future sessions? | Yes (can act as server) | Only if active MITM; cannot decrypt if not intercepting |

With forward secrecy, compromising $PR_S$ allows:
1. **Future impersonation**: the attacker can now create a fraudulent server (because they can produce valid CertificateVerify signatures). This requires immediate certificate revocation (ch3.2 p.50–53).
2. **No retroactive decryption**: past sessions cannot be decrypted because the session keys derived from ephemeral DH secrets that no longer exist anywhere. The recorded ciphertext is permanently unrecoverable.

**Why forward secrecy works**: the session key is cryptographically independent of $PR_S$. Recovering $PR_S$ gives the attacker the ability to break authentication (forge signatures) but not the ability to derive past session keys. The two cryptographic functions — authentication (long-term key) and key establishment (ephemeral key) — are decoupled.

---

### Practical Implementation

TLS 1.3 (ch3.6 p.7–8) **mandates** ECDHE (or DHE) for key exchange and completely removes RSA key exchange from the protocol. This was a deliberate design decision to ensure all TLS 1.3 connections have forward secrecy by construction. The server's RSA or ECDSA key pair is used only for the CertificateVerify signature — authentication only.

TLS 1.2 supported both RSA key exchange (no forward secrecy) and DHE/ECDHE (forward secrecy) via cipher suite negotiation. The recommended cipher suites in TLS 1.2 all used ECDHE (`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`, etc.).

---

### Summary

| Property | RSA key exchange | DHE / ECDHE |
|---|---|---|
| Session key depends on | Long-term $PR_S$ (directly used for decryption) | Ephemeral values $a, b$ (or $c, s$) — discarded |
| Past sessions recoverable if $PR_S$ compromised | **Yes** — retrospective decryption | **No** — ephemeral values gone |
| Forward secrecy | No | **Yes** |
| Role of long-term key $PR_S$ | Key establishment (encrypt PMS) | Authentication only (sign ephemeral DH values) |
| TLS 1.3 support | **Removed entirely** | **Mandatory** |

### Sources

- IS_UG_3_6_Appl_TLS (p.5: TLS 1.2 RSA key exchange; p.7–8: TLS 1.3 mandates ECDHE; p.37: forward secrecy definition and requirement)
- IS_UG_2_2_4_SecM_KeyExch (p.10: DHE authenticated key exchange and forward secrecy property)

_Status: Complete_  
_Done by: William_
