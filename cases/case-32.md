# Case 32

A mobile app allows users to share their real-time location with selected contacts for a limited time.

**Design the security architecture for this system. How do you enforce access control and expiration? How do you protect data confidentiality against the service provider itself? Which cryptographic mechanisms would you select? What legal and privacy constraints apply?**

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Real-time location is among the most sensitive personal data — it reveals home address, workplace, medical appointments, daily routine. The service provider must not be able to read it. | The service provider (or an attacker who breaches it) tracks every user's movements in real time. This data enables stalking, burglary during absence, and targeted physical attacks. |
| **Authentication** | Yes — critical | ch1 p.22 | The system must verify that the entity receiving a location update is actually the intended contact — not an impersonator. | An attacker registers with the same username as a contact and receives all location updates intended for that contact. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | Only explicitly selected contacts may view the sharer's location, and only during the permitted time window. | Any user of the platform can view anyone else's location. Or an expired sharing session continues indefinitely. |
| **Data integrity** | Yes | ch1 p.34 | Location data must arrive unmodified — a man-in-the-middle substituting a false location could mislead emergency responders or contacts. | Attacker replaces real location with a false one; a contact going to "meet up" arrives at the wrong location. |
| **Availability** | Yes | ch1 p.42 | Location sharing must be reliable for safety-relevant scenarios (child checking in, hiking trip tracking). | Service is unavailable during an emergency; a person in danger cannot be located. |

### Part 2 — Core Design Principle: End-to-End Encryption Against the Service Provider

**Option A — Server-stored plaintext:** The app sends location to the server; the server stores and forwards it. The server sees every location update for every user. Under legal compulsion, breach, or insider attack, all location history is exposed. **Rejected**: the service provider seeing real-time location of millions of users is an unacceptable privacy risk.

**Option B — Server-side encryption:** Server encrypts stored locations with a key it holds. Reduces data-at-rest exposure but the server still decrypts locations to forward them. **Rejected**: server still has access.

**Option C — End-to-end encryption (chosen):** The sharer's device encrypts location before sending to the server. The server stores and routes only ciphertext. Recipients decrypt on their own devices. The service provider cannot read any location, even under legal compulsion or after a breach.

### Part 3 — Key Establishment: ECDHE Per Sharing Session

For each sharing session (sharer A → contacts B, C, D), a shared symmetric key must be established between A and each permitted contact. Asymmetric key exchange is used to establish these symmetric keys.

**ECDHE** (ch2.2.4 p.10): Sharer A and contact B each have a long-term ECDSA P-256 keypair registered with the service (as part of their account identity). At the start of a sharing session, A performs an ephemeral Diffie-Hellman exchange with B:

```
Session start:
  A generates ephemeral keypair: (a_priv, a_pub)
  B has long-term keypair:       (b_priv, b_pub)

  A computes: K_AB = ECDHE(a_priv, b_pub)
  B computes: K_AB = ECDHE(b_priv, a_pub)
  → K_AB is the shared session key, known only to A and B
  → Server sees only { a_pub } encrypted for B; cannot derive K_AB
```

**Why ECDHE and not a server-generated symmetric key?** If the server generates and distributes K_AB, it knows K_AB and can decrypt every location update. ECDHE ensures K_AB is derived without the server ever learning it.

**Why ephemeral DH and not static long-term key?** ECDHE provides **forward secrecy** (ch3.6 p.37, ch2.2.4 p.10): if A's long-term private key is later compromised, past sharing sessions under ephemeral keys remain private. The ephemeral keypair is discarded after the session.

**Why ECDHE (P-256) and not DHE with large integer groups?** ECDHE provides equivalent security with much shorter keys (256 bits vs 2048-bit DH) — important for mobile devices transmitting key material (ch2.2.4 p.10).

### Part 4 — Location Encryption

Once K_AB is established, each location update from A to B is encrypted:

```
location_packet = AES-256-GCM encrypt(K_AB, { latitude, longitude, accuracy, timestamp }, nonce)
                  ↑ fresh 96-bit random nonce per update
```

**Why AES-256-GCM?**

- **Why AES-256 and not AES-128**: location data may be retained in logs for years (legal and service requirements). Against Grover's (ch2 PQCrypto p.16), AES-128 → 64-bit effective — insufficient for long-retained data. AES-256 → 128-bit post-quantum.
- **Why GCM and not CBC**: GCM is AEAD (ch2.2.3 p.70–75) — confidentiality and integrity in one pass. CBC requires a separate HMAC and has padding oracle risk. A falsified location (integrity attack) is a real threat; AEAD detects it at decryption.
- **Why fresh nonce per update**: AES-GCM is catastrophically broken under nonce reuse. Location updates are frequent (every few seconds) — each update must have a unique 96-bit random nonce.

**Timestamp in the plaintext**: the location update includes a timestamp inside the encrypted payload. The recipient can verify the location is fresh and not a replayed old position. The server cannot read or manipulate this timestamp.

### Part 5 — Access Control and Session Expiration

The server enforces **who** can retrieve location ciphertext and **when** — even though it cannot read the plaintext.

#### Access Control List

At session creation, the sharer specifies:

```
session = {
    session_ID,
    sharer_ID,
    allowed_contacts: [ contact_B_ID, contact_C_ID ],
    start_time,
    end_time
}
```

The server refuses to deliver location packets to any recipient not in `allowed_contacts`. Even if contact D somehow obtains the session_ID, the server rejects their requests.

**Why does server-enforced ACL provide meaningful security if the server can't read location?** The ACL protects the routing of ciphertext — an attacker not in the ACL cannot receive the location packets at all, not even in encrypted form. An attacker in the ACL could receive packets but would still need K_AB (established via ECDHE) to decrypt them.

#### Expiration

At `end_time`, the server immediately stops delivering location packets for this session. No new ECDHE-derived keys are established for expired sessions. The sharer's device also stops encrypting and sending after the session ends.

**Why time-limited and not on-demand?** "For a limited time" is in the problem statement. Time-limiting reduces the attack window: a compromised recipient account can only access location during the authorised window. Persistent location sharing would be a perpetual surveillance risk.

**Why server-enforced and not just client-enforced?** A compromised recipient's app could ignore client-side expiry. The server is the authoritative gatekeeper — it enforces expiry regardless of client behaviour.

### Part 6 — User Authentication

Password stored as `SHA-512(salt || password)` with 96-bit salt (ch3.2 p.11). Login via challenge-response (ch3.1 p.7) to defeat pass-the-hash:

```
Step 1 — Server → App:    { salt, nonce }
Step 2 — App computes:    HMAC(SHA-512(salt || password), nonce)
Step 3 — App → Server:    { response }
Step 4 — Server verifies: response matches → authenticated
```

**MFA recommended** (ch3.7 p.20): TOTP as second factor. For a location sharing app, account takeover is especially dangerous — an attacker who takes over the sharer's account can create sharing sessions with themselves.

All traffic over **TLS 1.3** (ch3.6 p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy. The server's X.509 certificate (ch3.2 p.28–29) is verified by the app.

### Part 7 — Why TLS Is Still Needed Even with E2E Encryption

TLS protects **metadata**: who is communicating with whom, session establishment timing, request sizes and patterns. Even if location content is E2E encrypted, an eavesdropper observing frequent small messages to a specific server IP can infer "this person is actively sharing location right now". TLS conceals this metadata from network-level observers.

TLS also protects the ECDHE ephemeral public keys exchanged at session setup — without TLS, a man-in-the-middle could substitute their own public key (ECDHE key substitution attack), establishing sessions with the attacker instead of the intended contact.

### Part 8 — Legal and Privacy Constraints (GDPR)

**Real-time location is personal data** under GDPR Article 4(1) — it directly identifies a natural person and their movements.

**Processing basis** (GDPR Art. 6): consent of the data subject. The sharer must explicitly consent to share their location with specific contacts for a specific time. The consent must be granular (per sharing session), revocable at any time, and not bundled with other service terms.

**Data minimisation** (GDPR Art. 5(1)(c)): only the minimum data necessary for the purpose must be processed. Location accuracy should be configurable — for "meeting up" purposes, city-level accuracy may suffice rather than GPS-precise coordinates. The server should not receive more precision than needed.

**Right to erasure** (GDPR Art. 17): the user must be able to delete their entire location history from the service at any time. Since the server stores only ciphertext, deletion of the session keys (held by the user's device) renders past ciphertext permanently unreadable — effective erasure even if ciphertext remains on server infrastructure.

**Purpose limitation** (GDPR Art. 5(1)(b)): location data collected for sharing with contacts must not be used for advertising profiling, sold to third parties, or used for any other purpose. The E2E encryption architecture technically enforces this — the provider cannot use data they cannot read.

**Data retention**: the server must not retain location ciphertext beyond the session end time. Expired sessions must be purged on a defined schedule.

### Part 9 — Remaining Vulnerabilities

- **Compromised recipient device**: if contact B's device is stolen or infected with malware, all location updates are visible to the attacker for the duration of the session. E2E encryption cannot protect against endpoint compromise. MFA and device lock mitigate this.
- **Metadata analysis**: the server cannot read location but can observe that sharer A is sending frequent messages to contact B. This metadata reveals that location sharing is active, and timing patterns may allow inference of movement rate. TLS conceals this from network observers but not from the service provider itself.
- **Sharer identity compromise**: if the sharer's account is taken over, the attacker can create sharing sessions with themselves as a contact and receive real-time location. Strong authentication (MFA) is the defence.
- **False contact registration**: if the contact verification system is weak, an attacker registers as a contact using a similar name. User care in verifying contact identities is required.
- **Relay attacks**: the server could relay session packets to additional recipients not in the ACL if the server is compromised. Recipients without K_AB cannot decrypt, but the routing bypass itself is a concern. This is mitigated by authenticated session metadata (ECDSA-signed session creation, ch2.2.3 p.85–87).

### Part 10 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Location confidentiality | E2E encryption — server never has plaintext | ch1 p.15 | Server breach / legal compulsion yields useless ciphertext; server-side encryption retains provider trust requirement |
| Session key establishment | ECDHE per session per contact | ch2.2.4 p.10; ch3.6 p.37 | Forward secrecy; server cannot derive K_AB; ephemeral key discarded after session |
| Location encryption | AES-256-GCM per update, fresh nonce | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | AEAD detects tampered location; 128-bit post-quantum security; per-update nonce prevents reuse |
| Access control | Server-enforced ACL + time window | ch1 p.30 | Server enforces routing regardless of client; expired sessions stopped server-side |
| User authentication | SHA-512 + challenge-response + TOTP MFA | ch3.2 p.11; ch3.1 p.7; ch3.7 p.20 | Pass-the-hash defeated; account takeover requires physical TOTP device |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Metadata protection; ECDHE key substitution attack prevented; forward secrecy |
| Legal | GDPR consent, data minimisation, right to erasure | GDPR Art. 5, 6, 17 | E2E encryption technically enforces purpose limitation; key deletion = effective erasure |

### Sources

- IS_UG_1_Introduction (p.5, p.10, p.15, p.22, p.30, p.34, p.42)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.63–66, p.70–75, p.85–87)
- IS_UG_2_2_4_SecM_KeyExch (p.10)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16–17)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.28–29)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.77, p.85)

_Status: Complete_  
_Done by: William_
