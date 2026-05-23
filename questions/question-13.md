# Question 13

Explain with sufficient detail how the dialogue between a client (C) and a Kerberos ticket granting server (TGS) works.

**What are the elements that guarantee the authenticity of the involved entities (C and TGS)? What are the possible consequences if an intruder succeeds in capturing the ticket granting ticket (TGT)? How is replay prevented? What cryptographic algorithms could be used (algorithm, mode, key length, etc.)?**

*Note: you may assume that only symmetric encryption mechanisms are used in this dialogue.*

## Answer

### Kerberos Context (ch3.2 p.16–27)

Kerberos is a centralised mutual authentication system based on symmetric cryptography and a Trusted Third Party (TTP) called the Key Distribution Center (KDC) (ch3.1 p.6). The KDC has two logical components:
- **AS** (Authentication Server): handles the initial client authentication and issues Ticket Granting Tickets (TGTs)
- **TGS** (Ticket Granting Server): exchanges TGTs for service tickets to specific application servers

This question focuses on the **client ↔ TGS dialogue** (the second phase), which occurs after the client has already obtained a TGT from the AS.

---

### Prerequisites: What the Client Has Before Approaching TGS

After the AS exchange (ch3.2 p.18–20), the client possesses:
- **TGT** = $E_{K_{TGS}}(C_{ID}, C_{addr}, validity, K_{C,TGS})$: a ticket encrypted under $K_{TGS}$ (the TGS's long-term key, shared only between the AS and TGS). The client cannot decrypt this ticket — it is opaque to the client.
- **Session key $K_{C,TGS}$**: a symmetric session key shared between the client and the TGS, delivered to the client encrypted under the client's long-term key $K_C$ (derived from the client's password). The client can decrypt this.

---

### The Client ↔ TGS Dialogue (ch3.2 p.20–24)

**Step 1 — Client's request to TGS** (client → TGS):

The client sends:
1. The **TGT** (opaque, encrypted under $K_{TGS}$): proves the client was authenticated by the AS; the TGS can decrypt it using $K_{TGS}$
2. An **authenticator** $Auth_C$: encrypted under the session key $K_{C,TGS}$:
   $$Auth_C = E_{K_{C,TGS}}(C_{ID}, timestamp)$$
   The authenticator contains the client's identity and the current timestamp.
3. The **requested service ID** $S_{ID}$: identifies which application server the client wants to access

**Step 2 — TGS processes the request**:

1. TGS decrypts the TGT using $K_{TGS}$, extracting: $C_{ID}$, $C_{addr}$, validity period, $K_{C,TGS}$
2. TGS verifies the TGT is not expired (validity period check)
3. TGS decrypts the authenticator $Auth_C$ using the extracted $K_{C,TGS}$, obtaining: $C_{ID}'$ and $timestamp$
4. TGS verifies:
   - $C_{ID}$ (from TGT) = $C_{ID}'$ (from authenticator): proves the entity presenting the authenticator is the same entity named in the TGT
   - $C_{addr}$ matches the client's network address (optional, may be omitted for mobile clients)
   - $timestamp$ is within a small tolerance window (typically ±5 minutes) of the TGS's current time: replay prevention (ch3.1 p.3)
   - The $(C_{ID}, timestamp)$ pair has not been seen before in the recent replay cache (ch3.1 p.7)

**Step 3 — TGS's response** (TGS → client):

If verification succeeds, the TGS sends:
1. A **service ticket** $ST_S$: encrypted under $K_S$ (the session key shared between TGS and the application server S — the client cannot decrypt this):
   $$ST_S = E_{K_S}(C_{ID}, C_{addr}, validity_S, K_{C,S})$$
2. A new **session key** $K_{C,S}$ (to be used between client C and server S): delivered to the client encrypted under $K_{C,TGS}$:
   $$E_{K_{C,TGS}}(K_{C,S}, S_{ID}, validity_S)$$

The client decrypts the second item using $K_{C,TGS}$ to obtain $K_{C,S}$. The client then uses $ST_S$ (opaque to the client) and $K_{C,S}$ to authenticate to the application server S in the subsequent exchange.

---

### What Guarantees Authenticity of Each Entity

**Authenticity of the client (C) to the TGS**:

1. **TGT**: the TGT was issued by the AS (the trusted KDC component) and is encrypted under $K_{TGS}$ — a key known only to the KDC. The TGS can verify the TGT is genuine by successfully decrypting it with $K_{TGS}$. A forged TGT would not decrypt correctly.

2. **Authenticator**: the authenticator is encrypted under $K_{C,TGS}$, which was never transmitted in the clear — it was only ever available inside the TGT (encrypted under $K_{TGS}$) and delivered to the client encrypted under $K_C$. Only the legitimate client who was authenticated by the AS and knows $K_C$ (their password-derived key) can extract $K_{C,TGS}$ and produce a valid authenticator.

3. **Matching identity**: the identity in the authenticator ($C_{ID}'$) must match the identity in the TGT ($C_{ID}$). This prevents a client from presenting someone else's TGT (they couldn't produce a matching authenticator without $K_{C,TGS}$).

**Authenticity of the TGS to the client**:

The client cannot directly verify TGS identity in a strict sense — the TGS does not sign its response. However, implicit authentication is provided: the session key $K_{C,S}$ delivered in the TGS response is encrypted under $K_{C,TGS}$. Only an entity that knows $K_{C,TGS}$ (= the TGS, which decrypted it from the TGT) can produce this encrypted response. An impostor TGS does not know $K_{C,TGS}$ and cannot produce a correctly encrypted response. (ch3.2 p.21 — mutual authentication property of symmetric key distribution.)

---

### Consequences of TGT Capture

If an intruder captures the client's TGT (ch3.2 p.24):

1. **Impersonation**: the TGT is a signed/encrypted credential that proves the holder was authenticated. An attacker with the TGT can present it to the TGS to request service tickets for any service the legitimate client could access — without knowing the client's password.

2. **Access to all services**: within the TGT's validity period, the attacker can request tickets for any application server in the Kerberos realm accessible to the client, gaining the same access rights as the legitimate client.

3. **The attacker still cannot decrypt the TGT itself** (it is encrypted under $K_{TGS}$) — but they do not need to. The TGT is presented opaquely to the TGS.

4. **The attacker needs $K_{C,TGS}$ to use the TGT**: the authenticator must be encrypted under $K_{C,TGS}$, which the attacker does not have (it was delivered to the client encrypted under $K_C$). Therefore, **capturing the TGT alone is insufficient** — the attacker also needs to capture $K_{C,TGS}$ (which may be stored in the client's memory alongside the TGT). If both are captured (e.g., from memory), full impersonation is possible.

5. **Time-limited damage**: TGTs have short validity periods (typically 8–10 hours in standard deployments). After expiry, the captured TGT is worthless.

6. **Mitigation**: the client address field $C_{addr}$ in the TGT, if enforced, limits replay to the same IP address. Modern networks (NAT, mobile) make this impractical to enforce, so TGTs are typically address-independent.

---

### Replay Prevention (ch3.1 p.3, ch3.1 p.7)

Kerberos prevents replay attacks through two mechanisms (ch3.2 p.21):

**1. Timestamps** (ch3.1 p.3): the authenticator contains a current timestamp. The TGS accepts only authenticators with a timestamp within a small window (e.g., ±5 minutes) of the current time. A replayed authenticator from more than 5 minutes ago is rejected. This requires all Kerberos participants to have synchronised clocks — typically via NTP.

**2. Replay cache** (ch3.1 p.7): the TGS maintains a cache of recently accepted $(C_{ID}, timestamp)$ pairs within the current tolerance window. Even within the ±5-minute window, each $(C_{ID}, timestamp)$ pair can only be accepted once. If the same authenticator is replayed within the window, the TGS detects it in the cache and rejects it.

The combination of these two mechanisms means:
- Old authenticators (>5 minutes) are rejected by the timestamp check
- Recent authenticators used twice are rejected by the replay cache
- A valid window of approximately 10 minutes is covered by the cache (±5 minutes), limiting cache size

---

### Cryptographic Algorithms (ch2.2.3 p.60–75)

Kerberos uses symmetric encryption exclusively (as stated in the question). Recommended choices:

**Session keys ($K_{C,TGS}$, $K_{C,S}$)**:
- **AES-256**: 256-bit key, providing 128-bit effective security post-quantum (ch2 PQCrypto p.16). Appropriate for session keys used for a single session lifetime (a few hours).

**Encryption mode for tickets and authenticators**:
- **AES-256-GCM** (ch2.2.3 p.70–75): AEAD mode providing both confidentiality and integrity in a single pass. The integrity tag prevents ticket tampering — even if an attacker modifies a ciphertext byte, decryption produces an authentication failure rather than silently corrupt data. 96-bit random nonce per encryption.
- Alternative: **AES-256-CBC** with a separate **HMAC-SHA256** (ch2.2.3 p.63–66) — also provides confidentiality and integrity but requires two separate operations. Less efficient than GCM.

**MAC for authenticators** (if separate from encryption):
- **HMAC-SHA256** (ch2.2.3 p.63–66): 256-bit key, 256-bit tag.

**Key derivation** (for long-term key $K_C$ from password):
- Password is hashed to derive $K_C$ using a slow, cost-parametrised function (ch3.2 p.11) — in practice, Kerberos uses string-to-key functions; for modern deployments this should use bcrypt or similar (see Question 11). The key length must match the encryption algorithm (256 bits for AES-256).

**Timestamps**: included verbatim in authenticators; encrypted under the session key, so their authenticity is protected. The timestamp itself must be precise to at least 1-second resolution to avoid replay within the tolerance window.

---

### Summary of the TGS Dialogue

| Step | Content | Key Used | Purpose |
|---|---|---|---|
| C → TGS | TGT (opaque) + Authenticator + $S_{ID}$ | TGT: $K_{TGS}$; Auth: $K_{C,TGS}$ | Request service ticket; prove identity |
| TGS verifies | Decrypt TGT; decrypt Auth; check IDs match, timestamp fresh | $K_{TGS}$, $K_{C,TGS}$ | Authenticate client; prevent replay |
| TGS → C | Service ticket $ST_S$ + $K_{C,S}$ | $ST_S$: $K_S$; $K_{C,S}$: $K_{C,TGS}$ | Grant access credential; establish new session key |

### Sources

- IS_UG_3_2_Appl_AuthMeth (p.16–27: Kerberos protocol; p.18: AS exchange; p.20–24: TGS exchange; p.11: password-based key derivation)
- IS_UG_3_1_Appl_Basics (p.3: timestamps and freshness; p.6: trusted third party; p.7: replay prevention and challenge-response)
- IS_UG_2_2_3_SecM_HashMac (p.63–66: HMAC-SHA256; p.70–75: AES-256-GCM)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16: AES-256 post-quantum security)

_Status: Complete_  
_Done by: William_
