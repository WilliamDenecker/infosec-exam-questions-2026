# Case 29

Design the security architecture for a backup service in a public cloud environment (provided by some cloud service provider).

**What are the most essential security services? What security mechanisms would you use to implement those services (be sufficiently specific)? How would you secure the access to the service? What could be remaining vulnerabilities? Don't forget to consider system security and protection against malware.**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Backup data contains everything on the user's system. The cloud provider must never be able to read it. Unauthorised disclosure cannot be undone (ch1 p.5). | A cloud provider employee or attacker reads all backed-up files, photos, financial records, and documents. Disclosure is permanent — data cannot be "un-disclosed". |
| **Data integrity** | Yes — critical | ch1 p.34 | A backup is useless if silently corrupted or tampered with. Integrity must be verifiable before restoration. | The user restores from backup after a disaster and gets corrupted or maliciously modified data, compounding the loss. |
| **Availability** | Yes — critical | ch1 p.42 | Backups must be retrievable exactly when needed — after a ransomware attack, hardware failure, or disaster. Recovery is time-critical. | The service is down during the emergency. The user cannot restore. The backup defeats its own purpose. |
| **Authentication** | Yes — critical | ch1 p.22 | Only the legitimate owner may upload, download, or manage their backups. | An attacker who guesses or steals credentials downloads all backups and then deletes them. |
| **Access control / authorisation** | Yes | ch1 p.30 | Each user's backups are strictly private. The cloud provider must not access contents; other users must not access them. | A flaw in authorisation allows any authenticated user to download any other user's backup set. |

### Part 2 — Core Design Principle: Client-Side Encryption

The most fundamental architectural decision is **where encryption happens**.

**Option A — Server-side encryption:** The user uploads plaintext; the cloud provider encrypts it with a key they manage. The provider can decrypt at any time — on demand, under legal compulsion, or following a breach.

**Why server-side is insufficient**: the cloud provider holds the key. A subpoena forces disclosure. A malicious employee can decrypt. A provider breach exposes both data and key simultaneously. **Rejected**: server-side encryption does not remove the trust requirement from the provider — it just adds a step.

**Option B — Client-side encryption (chosen):** The user's device encrypts data before it ever leaves the device. The cloud provider receives only ciphertext. The encryption key never leaves the client. A complete breach of the cloud infrastructure — servers, databases, storage — exposes only useless ciphertext.

**The definitive property**: under legal compulsion, the cloud provider genuinely cannot produce plaintext — they have never had the key. This is not a policy position but a cryptographic guarantee.

### Part 3 — Backup Encryption: Why AES-256-GCM and Not Alternatives

#### Why Not AES-128-GCM?

AES-128-GCM provides 128-bit classical security. Against Grover's quantum algorithm (ch2 PQCrypto p.16), this reduces to **64-bit effective security** — considered obsolete. Backups may be stored for years or decades. An adversary who records today's ciphertext and waits for a quantum computer can decrypt it retrospectively. For long-lived data, this is an unacceptable risk.

AES-256-GCM provides 256-bit classical → **128-bit effective post-quantum security** (ch2 PQCrypto p.16). This is the appropriate choice for data with long storage lifetimes.

#### Why Not AES-256-CBC + HMAC?

AES-256-CBC provides confidentiality only. To also provide integrity, a separate HMAC must be added — two operations, two keys, two passes over the data. Moreover, CBC is vulnerable to **padding oracle attacks** if any error information leaks from decryption failures. AES-256-GCM (ch2.2.3 p.70–75) is an **AEAD** (Authenticated Encryption with Associated Data) mode: confidentiality and integrity in a single pass, no padding (stream cipher mode within GCM), no padding oracle attack surface. **Rejected**: CBC adds complexity and attack surface.

#### Why Not Plain AES-256-ECB?

ECB mode encrypts each block independently. Identical plaintext blocks produce identical ciphertext blocks — structural patterns in the data are preserved in the ciphertext. For backup data with repeated headers, file structure, and content, ECB is trivially analysable. **Categorically rejected**.

**Chosen: AES-256-GCM** — quantum-safe, AEAD in one pass, no padding, no structural leakage.

#### Per-Chunk Encryption, Not Whole-Backup

The backup is divided into chunks. Each chunk is encrypted independently:

```
ciphertext_i = AES-256-GCM encrypt(backup_key, plaintext_i, nonce_i)
                                                               ↑ fresh 96-bit random nonce per chunk
```

**Why per-chunk and not one AES-256-GCM operation over the entire backup?**

1. **Nonce reuse catastrophe**: AES-GCM is catastrophically broken if the same (key, nonce) pair is ever reused — an attacker can recover the key. A single nonce for the entire backup can never be reused. Per-chunk nonces are independent; reuse of one chunk's nonce does not compromise other chunks or the key.
2. **Parallel verification**: each chunk's GCM tag is verified independently before restoration — corruption is localised without decrypting the entire backup.
3. **Incremental backup efficiency**: unchanged chunks are identified by their hash without re-encrypting the entire backup.

### Part 4 — Backup Encryption Key: Derivation and Storage

The backup encryption key must be derived on the client and must never leave the client device.

#### Key Derivation: Why HMAC and Not Plain Hash

```
backup_key = HMAC(password, client_salt)
```

**Why not `SHA-256(password || client_salt)`?** Length-extension attacks (ch2.2.3 p.24–32): knowing `SHA-256(password || client_salt)` allows computing `SHA-256(password || client_salt || padding || X)` without knowing the password. HMAC's double-hashing construction prevents this.

**Why HMAC and not the raw password as AES key?** The raw password is a short, low-entropy string. AES-256 requires a uniformly random 256-bit key. Using a password directly as a key means the effective key space is the password space, not 2²âµ⁶. HMAC with a salt spreads the password over the full 256-bit key space for the given salt.

**The client_salt** is a random 96-bit value generated at account creation and stored on the server. It ensures two users with the same password derive different backup keys. It is downloaded by the client at login — it is not secret (knowing the salt without the password does not help derive the key).

**Key never transmitted**: `backup_key` is computed on the client from the password and the downloaded salt. It is used locally for all encryption and decryption. It is never sent to the server under any circumstances.

#### Consequence of Key Loss

If the password is permanently lost, `backup_key` cannot be reconstructed and all backups are permanently unrecoverable. This is the intended behaviour — it is the cryptographic basis of the confidentiality guarantee. Users must securely store a recovery passphrase offline.

### Part 5 — Backup Manifest: Detecting Structural Attacks

The server stores individual encrypted chunks. Without additional structure, a compromised server could:
- **Delete chunks**: restoration fails on missing data
- **Reorder chunks**: restoration produces incorrect data
- **Substitute old chunks**: rolls back specific files to older versions

**Solution — Encrypted backup manifest** authenticated with both AES-256-GCM and HMAC:

```
manifest = { chunk_1: { chunk_ID, SHA-256(plaintext_1), sequence_number, timestamp },
             chunk_2: { chunk_ID, SHA-256(plaintext_2), sequence_number, timestamp },
             ...
             total_chunks: N }
HMAC_manifest = HMAC-SHA256(backup_key, manifest_contents_including_total_chunks)
```

The manifest is encrypted with AES-256-GCM and stored on the server. Before restoration:

```
1. Download and decrypt manifest
   → AES-256-GCM tag verifies manifest ciphertext integrity
2. Verify HMAC_manifest over full manifest structure
   → detects any structural modification (deleted chunk entries, changed total)
3. Download all chunks listed in manifest
4. Verify GCM tag on each chunk ciphertext → detects per-chunk tampering
5. Verify SHA-256(decrypted_chunk) against manifest entry → confirms plaintext content
6. Verify sequence numbers cover complete expected range → detects missing chunks
```

**Why both GCM tag AND SHA-256 in manifest?**
- GCM tag (Step 4): detects tampering with the ciphertext of an individual chunk
- SHA-256 in manifest (Step 5): confirms the plaintext content after decryption is what was originally backed up

They serve different purposes. One is ciphertext integrity; one is plaintext content integrity.

**Why HMAC over the manifest in addition to GCM?**
The GCM tag on the encrypted manifest detects modification of the manifest ciphertext. But if the server silently deletes a chunk entry from the manifest (rather than modifying it), the remaining manifest is internally self-consistent — its GCM tag still verifies. The HMAC is computed over the full manifest including `total_chunks` — a deleted entry changes the HMAC and reveals the structural attack.

### Part 6 — User Authentication: Access to the Service

#### Password Storage

Server stores: `SHA-512(salt || password)` with 96-bit random salt per account (ch3.2 p.11).

**Why SHA-512 and not SHA-256?** Post-quantum (ch2 PQCrypto p.17): SHA-256 → 128-bit preimage resistance → 64-bit after Grover's. SHA-512 → 256-bit preimage resistance → 128-bit after Grover's. For a long-lived password hash, SHA-512 is correct.

#### Login: Challenge-Response to Defeat Pass-the-Hash (ch3.1 p.7)

**Why not send the raw password?** The raw password derives `backup_key` — it must never leave the client. It must also never be transmitted, even inside TLS.

**Why not send the stored hash directly?** A stolen `stored_hash` from the server database can be replayed directly against the server. The server computes `HMAC(stored_hash, nonce)` — so does the attacker using the stolen hash. This is the pass-the-hash attack.

**Challenge-response protocol**:

```
Step 1 — Server → Client:   { salt, nonce }    (fresh 128-bit random nonce per session)
Step 2 — Client computes:   response = HMAC(SHA-512(salt || password), nonce)
Step 3 — Client → Server:   { response }
Step 4 — Server verifies:   HMAC(stored_hash, nonce) == response → Factor 1 passed
```

A stolen `stored_hash` is useless without the fresh nonce — and each nonce is single-use. A captured response from session A cannot be replayed into session B.

#### Multi-Factor Authentication

**Why MFA is required** (ch3.7 p.20, p.46): backup data contains everything. A single factor is insufficient for this sensitivity level. MFA absent in 59% of incidents.

**TOTP** (ch2.2.3 p.63–66):

```
T    = floor(current_unix_timestamp / time_window)
code = truncate(HMAC-SHA256(K_totp, T), 6 digits)
```

**Why TOTP and not SMS 2FA?** SIM-swapping and SS7 attacks allow interception of SMS codes. TOTP generates codes locally — no SMS infrastructure; no interceptable transmission.

All connections over **TLS 1.3** (ch3.6 p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37). The server's X.509 certificate (ch3.2 p.28–29) is verified by the client — prevents connecting to an impostor backup server.

**Brute-force protection** (ch3.7 p.85): account locked after repeated failed attempts.

### Part 7 — Append-Only Storage: Ransomware Resistance

**The threat**: ransomware compromises the user's device. The backup client software runs under the ransomware's control. The ransomware uses the client's authenticated session to delete or overwrite all backup versions — destroying the only recovery option.

| Storage model | Ransomware can destroy backups? |
|---|---|
| Normal storage (read/write/delete) | Yes — authenticated delete via normal API |
| Versioning only | Yes — authenticated bulk delete of all versions |
| Append-only (chosen) | No — deletion requires separate MFA re-authentication |

**Append-only storage**: the backup client can upload new backup versions but cannot delete or overwrite existing ones via the normal API. Deletion requires a separate, explicitly authenticated request with TOTP MFA re-verification — a step ransomware cannot complete without the user's physical TOTP device.

**Why MFA re-verification for deletion specifically?** The ransomware has access to the user's authenticated session. It can perform any action the normal backup client can perform. Append-only means the client session simply does not have delete permission — deletion is a privileged operation requiring a fresh MFA challenge.

**Retention policy**: backup versions retained for a configurable period (e.g., 30 days). This protects against **slow ransomware** that encrypts files gradually over weeks before triggering. Short retention windows may have no pre-infection clean backup available.

### Part 8 — System Security (Server Side)

**Packet filter** (ch3.7 p.51): only port 443 (HTTPS) inbound from internet. All other ports blocked. Management interfaces accessible only from internal administrative networks.

**Application-level gateway (proxy)** (ch3.7 p.62–65): in front of the backup API. Enforces authentication before forwarding any request. Inspects request sizes, rates, and patterns. Blocks anomalous or malformed requests before they reach the application server.

**Separation** (ch3.7 p.46): the authentication service and backup storage service are separate systems on separate servers. A storage server breach yields only ciphertext — no keys. A breach of the authentication service does not directly expose backup data.

**IDS** (ch3.7 p.77, p.85): detect anomalous patterns — mass download of entire backup sets from a new IP, repeated failed logins, download immediately followed by deletion requests, access from unusual geographic locations. Threshold detection triggers alerts and temporary holds (ch3.7 p.85). Log retention for retroactive forensic analysis (ch3.7 p.83, p.96).

**EPP on servers** (ch3.7 p.43): behaviour-based malware detection on cloud infrastructure.

### Part 9 — Remaining Vulnerabilities

- **Keylogger on client device**: captures the master password before client-side hashing. The attacker derives `backup_key = HMAC(password, client_salt)` and decrypts all downloaded backups. EPP on the client device (ch3.7 p.43) reduces this risk but cannot eliminate it — a kernel-level keylogger may evade EPP.
- **Lost account password**: if the password is permanently lost (and no recovery passphrase is stored), all backups are unrecoverable. This is the price of client-side encryption. Users must securely store a recovery passphrase offline.
- **Slow ransomware**: ransomware encrypts files gradually over weeks. All backup versions from the infection period contain ransomware-encrypted data. Only pre-infection versions are clean. Requires long retention windows to guarantee a pre-infection backup exists.
- **Cloud provider legal compulsion**: a government authority may compel the provider to hand over backup data or deny the user access. Client-side encryption ensures the provider cannot decrypt what they hand over. However, they can delete backups or deny access — geographic distribution across providers in different jurisdictions mitigates the denial-of-access risk.

### Part 10 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Encryption location | Client-side before upload | ch1 p.15 | Provider never has plaintext or key; server breach yields useless ciphertext; server-side retains provider trust requirement |
| Backup encryption | AES-256-GCM per chunk, fresh 96-bit nonce | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | AEAD in one pass; 128-bit post-quantum security; no padding oracle; per-chunk nonce prevents reuse catastrophe |
| Backup key derivation | HMAC(password, client_salt) — never leaves client | ch2.2.3 p.63–66 | HMAC prevents length extension vs plain SHA; spreads password entropy over full key space; server never has key |
| Manifest integrity | SHA-256 per chunk + HMAC over full manifest + GCM on manifest ciphertext | ch2.2.3 p.63–66, p.70–75 | GCM detects per-chunk ciphertext tampering; SHA-256 confirms plaintext; HMAC detects structural attacks (deleted chunks, reordering) |
| Login | Challenge-response HMAC with nonce | ch3.1 p.7; ch3.2 p.11 | Pass-the-hash defeated; stolen stored hash useless without fresh nonce; raw password never transmitted |
| MFA | TOTP (HMAC-SHA256 of shared secret and time) | ch3.7 p.20, p.46; ch2.2.3 p.63–66 | TOTP over SMS (no SS7 attack surface); code generated locally; physical device required |
| Ransomware resistance | Append-only storage; MFA required for deletion | ch1 p.30; ch3.7 p.20 | Compromised client session cannot delete backups; ransomware cannot complete MFA re-verification |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Mandatory AEAD; forward secrecy; certificate validates server identity |

### Sources

- IS_UG_1_Introduction (p.5, p.10, p.15, p.22, p.30, p.34, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.63–66, p.70–75)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16–17)
- IS_UG_3_1_Appl_Basics (p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.28–29)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.51, p.62–65, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
