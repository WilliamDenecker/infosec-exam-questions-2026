# Case 8

Design a system of single sign-on (SSO) for multiple independent websites. This system should enable the user to login (using a username-password combination) once to the SSO service (this may be a trusted third party) after which the user has access to all the websites for a certain period of time (e.g. 1 hour).

**What security mechanisms would you use? What would the messages that are exchanged to achieve the SSO service look like? How would you secure the access to the SSO service? How would you secure password storage and transmission?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions. Be sufficiently specific in your implementation (algorithms, key lengths, modes, etc.).*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | The SSO service must verify that only the legitimate user receives a session ticket. Each website must be certain the ticket came from the trusted KDC, not a forger. Mutual authentication — client and server — is required. | A forged ticket grants access to all websites at once; a single successful impersonation compromises the entire SSO domain. |
| **Confidentiality** | Yes — critical | ch1 p.15 | Session keys and tickets in transit must not be readable by any party. Tickets contain user identity and session key material. | An eavesdropper captures a session ticket and replays it to access any website for its remaining validity period. |
| **Data integrity** | Yes | ch1 p.34 | Tickets and authenticators must arrive unmodified. A tampered ticket could grant elevated privileges. | An attacker modifies a service ticket to escalate their role within the SSO domain. |
| **Non-repudiation** | Partial | ch1 p.40 | For audit purposes the SSO service must be able to prove which user accessed which website at what time. | An employee denies having accessed a restricted resource; no audit trail exists. |
| **Availability** | Yes | ch1 p.42 | The KDC is the single point of authentication for all websites. Its unavailability blocks access to all protected resources. | KDC downtime means no users can access any website, even with valid credentials. |

### Part 2 — Design Choice: Kerberos

The SSO system is implemented using **Kerberos** (ch3.2 p.16–27). Kerberos is a centralised authentication protocol that:
- provides **mutual authentication** — both client and server verify each other (ch3.2 p.18)
- issues **time-limited session tickets** — naturally implements the "1 hour access" requirement
- uses a **KDC (Key Distribution Center)** as the trusted third party (ch3.1 p.6)
- prevents **replay attacks** using timestamps and nonces (ch3.1 p.3, p.7)

All communications run over **TLS 1.3** (ch3.6 p.5, p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37). TLS is application-independent (ch3.6 p.5) — the same transport security layer protects all Kerberos messages without changes to the Kerberos protocol itself.

**Why Kerberos over a simpler token-based approach?** A simpler approach (e.g. server-issued JWT tokens) requires every website to either contact the SSO server on every request or to trust a shared signing key. Kerberos provides built-in mutual authentication, per-service session keys, and replay protection via timestamps — all properties that a simpler token system must re-implement separately (ch3.2 p.16–27).

### Part 3 — Password Storage

The user's password is stored on the KDC as `SHA-512(salt || password)` with a random per-user salt of at least 96 bits (ch3.2 p.11 — improved Linux scheme). The salt ensures each password must be attacked individually; rainbow table attacks are impossible because each user has a unique salt (ch3.2 p.10). A password strength policy is enforced: minimum length, mixed character classes, no dictionary words (ch3.2 p.13–14).

The user's password is used to derive a symmetric key on both the client and the KDC for the initial authentication step. The **raw password is never transmitted** — only a value derived from it.

### Part 4 — Protocol: Message Exchange

#### Phase 1 — Initial Login (Once, Using the Password)

**Step 1 — Client requests a Ticket-Granting Ticket (TGT):**

```
Client → KDC (Authentication Service):
  { username, timestamp }
```

The timestamp proves freshness (ch3.1 p.3) and prevents replay of old requests.

**Step 2 — KDC responds with TGT:**

```
KDC → Client:
  { session_key_TGS }  encrypted with client's password-derived key
  { TGT }              encrypted with KDC's master key K_KDC
```

TGT contents: `{ username, session_key_TGS, validity_period (1 hour), timestamp }`, encrypted with K_KDC. The client cannot read or modify the TGT — it is an opaque credential issued by the KDC.

The client decrypts the first part using its password-derived key, recovering `session_key_TGS`. This is the SSO credential — the user has authenticated **once** and now holds a TGT valid for 1 hour.

#### Phase 2 — Accessing a Website (No Password Re-entry)

**Step 3 — Client requests a service ticket for website W:**

```
Client → KDC (Ticket-Granting Service):
  { TGT }
  { authenticator: { username, timestamp } encrypted with session_key_TGS }
```

The authenticator proves the client possesses `session_key_TGS` without re-entering the password (ch3.2 p.21). The timestamp prevents replay (ch3.1 p.3).

**Step 4 — KDC issues service ticket for W:**

```
KDC → Client:
  { session_key_W }    encrypted with session_key_TGS
  { service_ticket }   encrypted with W's long-term key K_W
```

Service ticket contents: `{ username, session_key_W, validity_period, timestamp }`.

**Step 5 — Client authenticates to website W:**

```
Client → Website W:
  { service_ticket }
  { authenticator: { username, timestamp } encrypted with session_key_W }
```

**Step 6 — Mutual authentication (W authenticates back to client):**

```
Website W → Client:
  { timestamp + 1 }  encrypted with session_key_W
```

W decrypts the service ticket with K_W, recovers `session_key_W`, verifies the authenticator timestamp. W then responds by encrypting `(timestamp + 1)` with `session_key_W` — proving to the client that W genuinely holds K_W (mutual authentication, ch3.2 p.18, p.22). The session between client and W is subsequently protected with `session_key_W` (AES-256-GCM, ch2.2.3 p.70–75).

#### Summary

The user enters their password **once** (Step 1–2). For each subsequent website access during the 1-hour TGT validity, Steps 3–6 occur transparently without any password entry. This is the SSO effect.

### Part 5 — Securing Access to the KDC

**MFA required for TGT issuance** (ch3.7 p.20, p.46): a second factor (TOTP authenticator app) is required at Step 1 before the KDC issues a TGT. MFA is absent in 59% of incidents (ch3.7 p.20) — a stolen password alone must not be sufficient to obtain an SSO ticket.

**Brute-force protection** (ch3.7 p.85): the KDC applies threshold detection — account locked after repeated failed authentication attempts. This limits online password guessing against the SHA-512 hash.

**Packet filter** (ch3.7 p.51): the KDC is accessible only on the required Kerberos port from the internal network (or VPN). No direct internet exposure.

**Application-level gateway (proxy)** (ch3.7 p.62–65): a proxy in front of the KDC's HTTPS-facing login page inspects full HTTP requests, enforces TLS, and blocks malformed requests before they reach authentication logic. Unlike a packet filter, the proxy understands HTTP semantics and can block injection attacks.

**IDS** (ch3.7 p.77, p.85): detect anomalous patterns — mass failed login attempts (threshold detection, ch3.7 p.85), TGT requests from unusual IP ranges, abnormal ticket issuance volumes. Log all authentication events for retroactive forensic analysis (ch3.7 p.83, p.96).

**System hardening** (ch3.7 p.46): the KDC runs on a dedicated host — not shared with other services. Minimise attack surface: no unnecessary services, timely patching, EPP installed (ch3.7 p.43).

### Part 6 — Remaining Vulnerabilities

- **KDC compromise (single point of failure)**: if the KDC is breached, the attacker gains all long-term keys K_W for every website and the password hashes for all users. All active sessions and all future sessions are compromised. The KDC must be the most hardened and monitored system in the architecture. Redundancy (secondary KDC) addresses availability but creates two targets.
- **TGT theft (pass-the-ticket)**: malware on the client extracts the TGT from memory and impersonates the user for the remaining validity period. EPP (ch3.7 p.43) and short ticket validity (1 hour maximum) limit exposure.
- **Timestamp replay within the clock skew window**: Kerberos allows a small window for clock skew (typically 5 minutes). An authenticator captured and replayed within that window would be accepted. Nonces (ch3.1 p.7) complement timestamps; Kerberos uses both.
- **Weak password**: the entire chain rests on the password. A weak password is brute-forceable against the SHA-512 hash on the KDC (ch3.2 p.11). The password policy (ch3.2 p.13–14) and mandatory MFA are the primary mitigations.

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| SSO protocol | Kerberos with KDC as TTP | ch3.2 p.16–27; ch3.1 p.6 | Mutual authentication; time-limited tickets; built-in replay prevention |
| Password storage | SHA-512 + 96-bit salt per user | ch3.2 p.11 | Rainbow table attack defeated; no known practical collision |
| Session encryption | AES-256-GCM with session_key_W | ch2.2.3 p.70–75 | AEAD; unique per-service session key limits blast radius |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Forward secrecy; AEAD; application-independent |
| KDC access | MFA + packet filter + proxy + IDS | ch3.7 p.20, p.46, p.51, p.62–65 | Layered defence; stolen password alone insufficient |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.40, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.70–75)
- IS_UG_3_1_Appl_Basics (p.3, p.6–7)
- IS_UG_3_2_Appl_AuthMeth (p.10–11, p.13–14, p.16–27)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.51, p.62–65, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
