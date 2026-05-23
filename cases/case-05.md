# Case 5

Choosing good passwords and remembering them all may be a hard task for the human mind. Design the security architecture for a password vault service in a public cloud environment (provided by some cloud service provider).

**What are the most essential security services? What security mechanisms would you use to implement those services (be sufficiently specific)? How would you secure the access to the service? What could be remaining vulnerabilities? Don't forget to consider system security and protection against malware.**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Vault contents (all user passwords) must never be readable by the cloud provider, other users, or attackers. Unauthorised disclosure cannot be undone (ch1 p.5). | A cloud provider employee or attacker reads every password the user has ever stored. |
| **Authentication** | Yes — critical | ch1 p.22 | The service must verify that only the legitimate owner accesses their vault. | Any attacker who can guess or steal the master password gains all stored credentials. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | Each vault is strictly private; no user may access another user's vault. | A flaw in authorisation logic allows any authenticated user to download any vault. |
| **Data integrity** | Yes | ch1 p.34 | A silently corrupted or tampered vault entry is worse than a missing one — the user might unknowingly use a modified password. | Attacker changes a stored password for an important account; user is locked out without realising the vault was tampered. |
| **Availability** | Yes | ch1 p.42 | The vault must be accessible when the user needs a password — a login page is waiting. | Service downtime leaves users unable to access any of their accounts. |

### Part 2 — Design Principle: Client-Side Encryption

The cloud provider must not be able to read vault contents. The vault is encrypted on the client before upload; the server stores only ciphertext and never has the plaintext or the encryption key. Even a full breach of the cloud provider's servers exposes only useless ciphertext.

**Why client-side encryption?** Server-side encryption (cloud encrypts after receiving plaintext) means the cloud provider holds the key — a subpoena, breach, or malicious employee can access all data. Client-side removes this trust requirement entirely.

### Part 3 — Master Password Storage and Login Protocol

The server stores `SHA-512(salt || master_password)` with a random per-user salt of at least 96 bits (ch3.2 p.11). The raw password is never stored.

#### Why Not Send the Raw Password Over TLS?

A naive approach sends the plaintext password inside the TLS tunnel; the server recomputes the hash and compares. If the database is breached, the attacker steals `SHA-512(salt || password)` and can send it directly to authenticate without knowing the real password — a **pass-the-hash attack**. The stolen hash becomes the effective password.

#### Solution: Challenge-Response (ch3.1 p.7)

```
Step 1 — Client → Server:  username
Step 2 — Server → Client:  { salt, nonce }
Step 3 — Client computes:
          H = SHA-512(salt || master_password)
          response = HMAC(H, nonce)       ← ch2.2.3 p.63–66
Step 4 — Client → Server:  response
Step 5 — Server computes:
          expected = HMAC(stored_hash, nonce)
          if response == expected → authenticated
```

The **nonce** (ch3.1 p.7) is fresh and random per login. A stolen `stored_hash` cannot be replayed — the server generates a new nonce for each session, so `HMAC(stored_hash, old_nonce)` never matches `HMAC(stored_hash, new_nonce)`.

**TLS 1.3** (ch3.6 p.5, p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) is still required. TLS prevents a real-time MitM attacker from intercepting and relaying the nonce/response, because the server's X.509 certificate (ch3.2 p.28) is verified by the client. Challenge-response defeats pass-the-hash; TLS defeats real-time MitM — both are needed.

#### Vault Encryption Key

The master password also derives the vault decryption key on the client using HMAC-based key derivation (ch2.2.3 p.63–66): `vault_key = HMAC(master_password, client_salt)`. This key **never leaves the device**. The server sends back the encrypted vault blob after authentication; the client decrypts it locally.

A strong password policy is enforced (ch3.2 p.13–14): minimum length, character classes, no dictionary words.

### Part 4 — Vault Encryption

Each vault entry is encrypted with **AES-256-GCM** (ch2.2.3 p.70–75). GCM is an AEAD mode: single operation for confidentiality + integrity. The 128-bit GCM authentication tag detects any tampering with the ciphertext before decryption. A fresh random 96-bit nonce per encryption operation.

**Why AES-256-GCM over AES-128-GCM?** Against quantum adversaries (Grover's), AES-128 provides only 64-bit effective security — obsolete (ch2 PQCrypto p.16). AES-256 retains 128-bit effective security post-quantum (ch2 PQCrypto p.16).

**Why GCM over CBC?** CBC requires a separate MAC for integrity and is vulnerable to padding oracle attacks. GCM provides authenticated encryption in one pass with no padding oracle risk (ch3.6 p.37, ch2.2.3 p.70–75).

### Part 5 — Access Security

**MFA** required (ch3.7 p.20, p.46): TOTP authenticator app preferred over SMS to avoid SIM-swap attacks. MFA still absent in 59% of incidents (ch3.7 p.20) — its absence is the leading cause of account compromise. Even if the master password is stolen, MFA blocks access.

**Brute-force protection** via threshold detection (ch3.7 p.85): account locked after repeated failed login attempts.

**Device trust** (ch3.2 p.14): after successful MFA, a device can be registered as trusted for a limited period (e.g. 30 days), reducing friction for daily use. Explicit usability vs. security trade-off — longer validity window improves convenience but extends exposure if device is stolen.

### Part 6 — System Security (Server Side)

**Packet filter** (ch3.7 p.51): permit only port 443 inbound. All other ports blocked.

**Application-level gateway (proxy)** (ch3.7 p.62–65): in front of vault API. Enforces authentication before forwarding; blocks malformed requests before they reach application logic. Unlike a packet filter, the proxy understands HTTP semantics and can block injection attacks.

**Separation** (ch3.7 p.46): authentication service, vault storage server, and key derivation service are separate systems. A compromise of the storage server yields only ciphertext — the keys are elsewhere.

**IDS** (ch3.7 p.77, p.85): detect anomalous access — repeated failed logins, mass vault downloads from unusual IPs (threshold detection, ch3.7 p.85). Retain logs for retroactive analysis (ch3.7 p.83, p.96).

**EPP on servers** (ch3.7 p.43): behaviour-based malware detection.

### Part 7 — Remaining Vulnerabilities

- **Keylogger on client**: captures master password before client-side hashing and key derivation — bypasses all server-side security. EPP on client device (ch3.7 p.43) reduces this.
- **Weak master password**: user ignores the strength policy. A brute-force attack against the SHA-512 hash (ch3.2 p.11) becomes feasible for weak passwords despite challenge-response.
- **Lost master password**: the vault key is derived from the master password and never stored on the server — no server-side recovery possible. The user must maintain a secure offline backup. This is a deliberate usability trade-off.
- **Session hijacking**: a stolen session token allows vault download. Attacker still cannot decrypt without the master password, but can attempt offline brute-force.

### Part 8 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Master password storage | SHA-512 + 96-bit salt | ch3.2 p.11 | Salt defeats rainbow tables; SHA-512 no known practical attack |
| Login | Challenge-response HMAC with nonce | ch3.1 p.7; ch2.2.3 p.63–66 | Pass-the-hash defeated; stolen DB hash cannot be replayed |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Real-time MitM defeated; forward secrecy |
| Vault encryption | AES-256-GCM per vault | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | AEAD (integrity + confidentiality); quantum-safe at 256 bits |
| Vault key | Client-derived from master password; never on server | ch2.2.3 p.63–66 | Cloud provider breach exposes only useless ciphertext |
| MFA | TOTP authenticator app | ch3.7 p.20, p.46 | Phishing-resistant vs. SMS; no SIM-swap risk |

### Sources

- IS_UG_1_Introduction (p.5, p.10, p.15, p.22, p.30, p.34, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.63–66, p.70–75)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.13–14, p.28)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.51, p.62–65, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
