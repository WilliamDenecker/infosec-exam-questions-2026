# Case 2

Email accounts may be very vulnerable. A few years ago (2013), it was the Prime Minister of Belgium who made the news because his private email account had been hacked.

Assume you are an ISP. Your clients have the possibility to access their email accounts using either a mail client or webmail.

**How would you secure your email service to minimise the risk of client accounts being hacked? Don't forget system security and don't forget to consider the usability of the system. How would you manage the password recovery process (yes, some customers will forget their passwords)?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions. Be sufficiently specific in your implementation (algorithms, key lengths, modes, etc.).*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Security Service | Required? | Reasoning | What breaks without it |
|---|---|---|---|
| **Confidentiality** | Yes — critical | Email content is private personal data. In transit it must be unreadable to any party other than sender and recipient. At rest on the mail server it must be unreadable to ISP staff/attackers (ch1 p.15). | Any attacker or malicious ISP employee can read all customer emails — both in transit and stored on the server. |
| **Authentication** | Yes — critical | The server must verify that the entity claiming to be user X is actually user X before granting access. This is the direct attack surface in the PM Belgium case — a weak password enabled account takeover (ch1 p.22). | Any attacker who knows (or guesses) a username+password can access the mailbox. The entire system falls without this. |
| **Access Control / Authorisation** | Yes — critical | After authentication, the system must enforce that user X can only read their own mailbox. ISP staff must not be able to read customer mail. Requires least privilege per role (ch1 p.30). | Authenticated users or staff can access each other's mailboxes — a breach even without password compromise. |
| **Data Integrity** | Yes | Emails must not be modified in transit or in storage. A malicious middlebox or storage attacker must not be able to alter message content silently (ch1 p.34). | Emails can be tampered with without the recipient detecting it — business fraud, modified instructions. |
| **Non-Repudiation** | Partially — optional | For ordinary consumer email, non-repudiation is not typically required at the ISP level. It would be relevant for legal disputes. For most consumer accounts this is out of scope — it requires end-to-end signing (e.g. S/MIME) which is a client-side concern (ch1 p.40). | A user can deny having sent or received an email. Relevant mainly in legal/business contexts. |
| **Availability** | Yes | Email is a critical communication service. The mail server, webmail interface, and authentication infrastructure must remain accessible under load and resist DoS attacks (ch1 p.42). | Users cannot access their email. An ISP that goes down regularly loses customers and trust. |

### Part 2 — Transport Security (ch3.6)

All client-to-server communication must be encrypted. This applies to both access paths:

- **Webmail**: served exclusively over HTTPS (HTTP over TLS). HTTP on port 80 must be disabled or immediately redirected to HTTPS on port 443.
- **Mail client**: the client protocols must use their TLS variants:
  - IMAPS (IMAP over TLS, port 993) for reading mail
  - SMTPS / SMTP+STARTTLS (port 465 or 587) for sending mail
  - Plain IMAP (port 143) and plain SMTP (port 25 for submission) must be disabled for client access

**TLS 1.3** (ch3.6 p.7–8) is chosen over TLS 1.2. TLS operates at the transport layer and is application-independent — the same security layer protects both mail clients and webmail without changes to the application (ch3.6 p.5). TLS 1.2 is still common but allows obsolete algorithms and does not mandate forward secrecy. TLS 1.3 removes all legacy algorithms (RC4, MD5, CBC-mode AES, static RSA key exchange) and mandates ephemeral key exchange, which provides forward secrecy (ch3.6 p.37): even if the server's private key is later compromised, past sessions cannot be decrypted.

**Cipher suite**: `TLS_AES_256_GCM_SHA384`:

- **AES-256-GCM** (ch3.6 p.13, ch2.2.1 p.55): AES with a 256-bit key in Galois/Counter Mode. GCM is an AEAD mode — it provides both confidentiality and integrity/authentication in a single pass, and is parallelisable. AES-256 is preferred over AES-128: both are secure today, but AES-256 provides a larger security margin against future attacks — including post-quantum attacks via Grover's algorithm which halves effective key length, making AES-256 equivalent to ~128-bit post-quantum security (ch2 PQCrypto p.16).
- **ECDHE** key exchange (ch3.6 p.18): both parties generate ephemeral EC keypairs per connection → fresh session key per connection → forward secrecy guaranteed (ch3.6 p.37). Why not CBC mode? CBC-mode AES requires careful IV management and is vulnerable to padding oracle attacks. GCM eliminates these concerns entirely (ch3.6 p.37 — TLS 1.3 drops CBC support).
- **SHA-384** for HKDF key derivation — collision resistant, no known practical attack.

**Server certificate**: an X.509v3 certificate (ch3.2 p.28–29) issued by a trusted CA, using ECDSA with P-256 or RSA-2048 minimum. The Subject Alternative Name must cover both the webmail domain and the mail server hostname.

### Part 3 — Authentication: Securing Login (ch3.2)

#### Password Storage: Salted Hashing

Passwords must never be stored in plaintext (ch3.2 p.6). The standard approach is a salted one-way hash (ch3.2 p.10–11).

**Chosen function**: a **memory-intensive hash function** (ch3.2 p.11 — "memory-intensive s-crypt functions make hardware-based attacks impractical"). The slides note that slow/memory-intensive one-way functions are good for single-user checks but slow down attackers trying many passwords. scrypt is the modern standard instantiation of this principle — it requires large amounts of memory (e.g. 64 MB) per evaluation, making parallel GPU/ASIC attacks economically infeasible since memory cannot be cheaply parallelised the way compute can.

Storage format per user: `user_id | salt (128-bit random) | scrypt(password ∥ salt)`

**Why a salt?** (ch3.2 p.10): without a salt, two users with the same password produce the same hash. An attacker who steals the database can precompute a single rainbow table and find all matching passwords at once. With a unique 128-bit random salt per user, the attacker must compute scrypt separately for every (salt, password) combination — making precomputed tables useless (ch3.2 p.10).

**Pepper**: in addition to the per-user salt, a system-wide pepper secret value is added before hashing: `scrypt(password ∥ salt ∥ pepper)`. The pepper is stored in application configuration, not in the database. An attacker who steals only the database cannot perform offline cracking without also knowing the pepper.

A password policy (ch3.2 p.13) is enforced: minimum length, mixed character classes, no dictionary words. Password-strength evaluation software serves as a first filter (ch3.2 p.14).

#### Password Transmission

Over the TLS 1.3 connection, the password is transmitted encrypted within the tunnel. For mail clients using IMAP/SMTP, the **SASL PLAIN mechanism** over TLS is acceptable. Where supported, **SCRAM-SHA-256** (a challenge-response protocol where the server never receives the plaintext password) is preferred — it provides an additional layer if TLS is somehow compromised at the transport layer, and aligns with the challenge-response principle (ch3.1 p.7).

#### Multi-Factor Authentication

MFA is identified as one of the single most effective preventive measures — still absent in 59% of incidents (ch3.7 p.20). Password alone is insufficient: a weak or reused password is all an attacker needs, as demonstrated by the PM Belgium case.

**TOTP-based MFA** is required as a second factor. During setup, the server generates a shared secret K and delivers it to the user's authenticator app via QR code. At login, both the server and app compute:

```
OTP = truncate(HMAC-SHA256(K, ⌊t/time_window⌋), 6 digits)   (ch2.2.3 p.63–66)
```

where t is the current Unix timestamp. The OTP is single-use and short-lived — the timestamp component (ch3.1 p.3) ensures freshness. Since both sides use the same shared secret and time window, no network round-trip is needed.

**Why TOTP over SMS OTP?** SMS is vulnerable to SIM-swapping attacks. TOTP requires physical possession of the enrolled device and does not depend on the mobile network (ch3.7 p.46 — phishing-resistant MFA).

**Usability balance** (ch3.2 p.14): MFA adds friction. To reduce burden: require MFA only on first login from a new device/IP, or allow "remember this device for 30 days". This is an explicit trade-off — more convenience reduces security, but periodic MFA is far better than none.

#### Brute-Force and Credential Stuffing Protection

From ch3.7 p.85 (threshold detection): after 5 failed login attempts within 10 minutes, lock the account temporarily and notify the user. Progressive delays after each failed attempt make online brute-forcing impractical. IP-based throttling detects and blocks IPs performing distributed credential stuffing.

### Part 4 — Password Recovery Process

Password recovery is a second authentication pathway that must be at least as strong as the primary one. A weak recovery mechanism entirely bypasses strong password requirements.

**Chosen approach**: identity verification via secondary email, then forced password reset. The process:

1. User requests a password reset by providing their registered email address.
2. The server generates a cryptographically random token (256-bit) and stores its hash (SHA-256, ch2.2.3 p.24–32) in the database with a **15-minute expiry timestamp** (ch3.1 p.3 — freshness).
3. The server sends a single-use reset link to the registered secondary contact address over TLS.
4. The user clicks the link. The server verifies the token hash, checks it has not expired, and marks it as **used** (preventing replay — ch3.1 p.7).
5. The user sets a new password satisfying the strength policy (ch3.2 p.13–14).
6. After successful reset, the server **invalidates all active sessions** for that account immediately.

**Why not security questions?** Security questions ("mother's maiden name", "first pet's name") are easily researched from social media and provide very low entropy — they are effectively a weaker password chosen from a tiny space.

**Why a time-limited, single-use token?** The token is a nonce that must be returned within a time window (ch3.1 p.7). The 15-minute expiry prevents an attacker who later gains access to the email account from using an old token. Single-use prevents replay (ch3.1 p.7). Session invalidation on reset ensures that even if an attacker has an active session at the time the legitimate user resets their password, that session is immediately terminated.

**If MFA device is also lost**: the user must authenticate in person or via government-issued eID. This is less convenient but unavoidable — there is no secure remote fallback when both factors are gone.

### Part 5 — System Security (ch3.7)

#### Network Architecture: DMZ

The mail infrastructure sits in a **DMZ** (ch3.7 p.47): public-facing servers (webmail, IMAPS, SMTPS) are in the DMZ; internal systems (user database, authentication backend) are in the protected internal network. Packet filter rules (ch3.7 p.51):

- **External → DMZ**: only HTTPS (443), IMAPS (993), SMTPS (587) permitted. All other ports blocked.
- **DMZ → internal**: only strictly necessary database and authentication calls. No direct internet access from internal systems.

**Application-level gateway (proxy)** in front of the webmail server (ch3.7 p.62–65): the proxy understands the full HTTP protocol and can inspect and block malicious requests before they reach the application. It can also enforce authentication before forwarding any request — unlike a packet filter which operates only at the IP/port level.

**Separation of frontend and storage** (ch3.7 p.46 — minimise attack surface): webmail and IMAP frontend servers face the internet; mail storage servers sit behind the internal packet filter and are never directly reachable from outside.

#### Principle of Least Privilege (ch3.7 p.46)

The mail server process runs as a dedicated, unprivileged user account. ISP staff accounts follow role separation: helpdesk staff can trigger account resets but cannot read mailbox contents. No single role has unrestricted access to both authentication data and mailbox content.

#### Endpoint Protection and Patching (ch3.7 p.38–43)

EPP with behaviour-based detection on all servers (ch3.7 p.43). Critical security patches must be deployed promptly — attackers monitor patch releases and actively target the window between release and deployment. Automated vulnerability scanning identifies unpatched components.

#### Intrusion Detection (ch3.7 p.77, p.83, p.85, p.96)

An IDS is deployed at the mail server level:

- **Threshold detection** (ch3.7 p.85): detect and alert on many failed login attempts from a single IP or across many accounts — indicating credential stuffing or brute-force. Feeds directly into the rate-limiting mechanism.
- **Profile-based detection** (ch3.7 p.77): detect anomalous behaviour post-login — e.g. a user who normally accesses email from Belgium suddenly downloads the entire mailbox from a foreign IP.
- **Audit logs** (ch3.7 p.83, p.96): all login attempts (successful and failed), IP addresses, timestamps, and session events are logged. Minimum 90-day retention. Logs stored on a separate, write-protected system to prevent tampering by an attacker who gains access to the mail server (ch3.7 p.96).

### Part 6 — Legal Considerations

- **GDPR**: as an ISP storing customer email, the company is a data controller. Technical measures above (TLS, salted hashing, access control, firewalls) implement the integrity and confidentiality principle (Art. 5(1)(f) GDPR). Deleted emails must actually be deleted — not retained indefinitely (storage limitation, Art. 5(1)(e)). A breach affecting customer data must be notified to the DPA within 72 hours.
- **Computer crime law**: the PM Belgium case involved unauthorised access to a computer system (art. 550bis Belgian Criminal Code). As an ISP, failure to implement reasonable security measures may expose the ISP to liability if a data breach results from clearly inadequate security (e.g. passwords stored in plaintext).
- **Audit logs** also serve a legal purpose: if a customer account is hacked and a complaint is filed, the ISP may need to provide evidence to law enforcement.

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Transport encryption | TLS 1.3 with AES-256-GCM-SHA384 | ch3.6 p.7–8, p.37 | Mandatory forward secrecy; removes all legacy/weak algorithms; AEAD eliminates padding oracle vulnerabilities present in TLS 1.2 CBC |
| Password storage | Memory-intensive hash (scrypt) with 128-bit salt + pepper | ch3.2 p.11 | Memory-hardness defeats GPU/ASIC brute force; salt defeats rainbow tables; pepper protects against DB-only theft |
| Login authentication | Password + TOTP MFA | ch3.7 p.20, p.46; ch2.2.3 p.63–66 | TOTP is phishing-resistant and does not depend on mobile network (unlike SMS OTP which is vulnerable to SIM-swapping) |
| Brute-force protection | Rate limiting + progressive lockout + IP throttling | ch3.7 p.85 | Prevents online dictionary attacks and credential stuffing — the dominant attack vector |
| Password recovery | Time-limited (15 min) single-use token to secondary email | ch3.1 p.3, p.7; ch3.2 p.13–14 | Single-use prevents replay; time limit prevents delayed use; no security questions (too guessable) |
| Network architecture | DMZ with external packet filter + application proxy + internal filter | ch3.7 p.47, p.51, p.62–65 | Limits blast radius: compromise of DMZ server does not grant access to internal authentication systems |
| Audit and IDS | Threshold + profile-based IDS; 90-day log retention | ch3.7 p.77, p.83, p.85, p.96 | Detects both brute-force (threshold) and post-compromise anomalies (profile); long retention enables forensic reconstruction |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.40, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.63–66)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.6, p.10–11, p.13–14, p.28–29)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.13, p.15, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.38–43, p.46–47, p.51, p.62–65, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
