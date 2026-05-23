# Case 14

Design the security architecture for a file service in a public cloud environment (provided by some cloud service provider).

**What are the most essential security services? What security mechanisms would you use to implement those services (be sufficiently specific)? How would you secure the access to the service? What could be remaining vulnerabilities? Don't forget to consider system security and protection against malware.**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Files may contain sensitive personal or business data. The cloud provider must never be able to read them; unauthorised disclosure cannot be undone (ch1 p.5). | A cloud provider employee or attacker reads all files. A breach of the provider's servers exposes every user's data. |
| **Data integrity** | Yes — critical | ch1 p.34 | A silently corrupted or tampered file is worse than a missing one — the user may unknowingly work with modified data. | An attacker or malicious cloud provider silently modifies a contract, spreadsheet, or medical record. The user trusts and acts on incorrect data. |
| **Authentication** | Yes — critical | ch1 p.22 | The service must verify that only the legitimate owner accesses their files. | Any attacker who can guess or steal login credentials gains full access to all stored files. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | Each user's files are private; shared files accessible only to explicitly authorised users; the cloud provider itself must be excluded from file contents. | A flaw in authorisation allows any authenticated user to access any other user's files. |
| **Availability** | Yes | ch1 p.42 | Files must be accessible when the user needs them — a file service that is unavailable is useless. | Service downtime prevents users from accessing business-critical documents. |

### Part 2 — Design Principle: Client-Side Encryption

The cloud provider must not be able to read file contents. Files are encrypted on the client before upload; the server stores only ciphertext and never has access to the plaintext or the encryption key.

**Why client-side over server-side encryption?** Server-side encryption means the cloud provider holds the key — a subpoena, breach, or malicious employee can access all files. Client-side encryption removes this trust requirement entirely: even a full breach of the cloud provider exposes only useless ciphertext.

### Part 3 — Authentication and Access Security

The user's account password is stored on the server as `SHA-512(salt || password)` with a random per-user salt of at least 96 bits (ch3.2 p.11).

Login uses **challenge-response** (ch3.1 p.7) to prevent pass-the-hash attacks:

```
Step 1 — Client → Server:  username
Step 2 — Server → Client:  { salt, nonce }
Step 3 — Client computes:
          H = SHA-512(salt || password)
          response = HMAC(H, nonce)
Step 4 — Client → Server:  response
Step 5 — Server verifies:  HMAC(stored_hash, nonce) == response → authenticated
```

A stolen `stored_hash` cannot be replayed — the server generates a fresh nonce per session, so `HMAC(stolen_hash, old_nonce)` never matches `HMAC(stored_hash, new_nonce)`.

All communications use **TLS 1.3** (ch3.6 p.5, p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37). TLS prevents real-time MitM during the challenge-response exchange; the server's X.509 certificate (ch3.2 p.28) is verified by the client.

**MFA required** (ch3.7 p.20, p.46): TOTP authenticator app as a second factor. MFA is absent in 59% of incidents (ch3.7 p.20) — a stolen password alone must not be sufficient.

**Brute-force protection** via threshold detection (ch3.7 p.85): account locked after repeated failed login attempts.

### Part 4 — File Encryption (Client Side)

Each file is encrypted on the client with **AES-256-GCM** (ch2.2.3 p.70–75) before upload. GCM is an AEAD mode: one operation provides both confidentiality and integrity. The 128-bit GCM authentication tag detects any tampering with the ciphertext after storage. A fresh random 96-bit nonce is generated per file encryption.

**Why AES-256 over AES-128?** Against Grover's quantum algorithm, AES-128 provides only 64-bit effective security — obsolete (ch2 PQCrypto p.16). AES-256 retains 128-bit effective security. Files stored in the cloud may persist for years.

**Why GCM over CBC?** CBC-mode AES provides confidentiality only and requires a separate MAC. GCM provides authenticated encryption in one pass, has no padding oracle risk, and is parallelisable (ch2.2.3 p.70–75).

**File encryption key management**: each file has a unique randomly generated 256-bit AES key (the file key). The file key is encrypted with the user's master key and stored alongside the ciphertext on the server. The master key is derived on the client from the account password using HMAC key derivation (ch2.2.3 p.63–66): `master_key = HMAC(password, client_salt)`. The **master key never leaves the client device**.

**File sharing**: when sharing a file with another user, the file key is encrypted with the recipient's public key (RSA-2048 is "really secure" per ch2.2.2 p.14) and stored on the server. The recipient decrypts the file key with their private key, then decrypts the file. The server never holds any file key in plaintext.

### Part 5 — Data Integrity

The AES-256-GCM tag per file detects per-file tampering. For detecting server-side deletion, substitution, or rollback of files, the client maintains a **signed manifest** of all file metadata (name, size, last-modified timestamp, SHA-256 hash of ciphertext). The manifest is stored locally and verified on each sync. A mismatch indicates silent deletion, rollback, or substitution by the server.

### Part 6 — System Security (Server Side)

**Packet filter** (ch3.7 p.51): permit only port 443 inbound. All other ports blocked.

**Application-level gateway (proxy)** (ch3.7 p.62–65): in front of the file API. Enforces authentication before forwarding any request; inspects request sizes and rates to block malicious uploads. Unlike a packet filter, the proxy understands HTTP semantics and can block injection attacks.

**Separation** (ch3.7 p.46 — minimise attack surface): authentication service, file storage, and encrypted-key storage are separate systems. A breach of the storage server yields only ciphertext — the keys are stored separately.

**IDS** (ch3.7 p.77, p.85): detect anomalous patterns — mass downloads from unusual IPs, repeated failed logins (threshold detection, ch3.7 p.85), access from unexpected geographic regions. Retain logs for retroactive forensic analysis (ch3.7 p.83, p.96).

**EPP on servers** (ch3.7 p.43): behaviour-based malware detection on all server-side systems.

### Part 7 — Remaining Vulnerabilities

- **Keylogger / malware on client device**: captures the account password before client-side hashing and key derivation. The attacker derives the master key and decrypts all files. EPP on client devices (ch3.7 p.43) reduces this risk but cannot guarantee protection.
- **Lost master password**: the master key is derived entirely from the password — if the password is permanently lost, all encrypted files are unrecoverable. No server-side recovery is possible (by design: the server never had the key). Users must maintain a secure offline backup of the master key or recovery passphrase.
- **Metadata leakage**: even with encrypted content, the server observes file sizes, access times, and access patterns. This may reveal sensitive information about usage patterns even without decrypting any content.
- **Cloud provider serves a modified client**: a malicious provider could push a compromised web or desktop application that exfiltrates the password before client-side encryption. Native apps with code signing and open-source clients mitigate this.

### Part 8 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| File encryption | AES-256-GCM per file, client-side | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | AEAD; quantum-safe at 256 bits; cloud provider never has plaintext |
| Master key derivation | HMAC(password, client_salt) — never sent to server | ch2.2.3 p.63–66 | Key stays on client; server breach cannot expose key material |
| Login | Challenge-response HMAC with nonce | ch3.1 p.7; ch3.2 p.11 | Pass-the-hash defeated; stolen stored hash cannot be replayed |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Forward secrecy; AEAD; real-time MitM defeated |
| MFA | TOTP authenticator app | ch3.7 p.20, p.46 | Password alone insufficient; SIM-swap resistant vs. SMS |
| File sharing | File key encrypted with recipient's RSA-2048 public key | ch2.2.2 p.14 | Per-recipient, per-file access; server never holds plaintext file keys |

### Sources

- IS_UG_1_Introduction (p.5, p.10, p.15, p.22, p.30, p.34, p.42)
- IS_UG_2_2_2_SecM_AsymmEncr (p.14)
- IS_UG_2_2_3_SecM_HashMac (p.63–66, p.70–75)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.28)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.51, p.62–65, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
