# Case 31

Choosing good passwords and remembering them all may be a hard task for the human mind. Design the security architecture for a password manager running on a local device.

**What are the most essential security services? What security mechanisms would you use to implement those services (be sufficiently specific)? How would you implement the transfer of the password manager data to a new device? What could be remaining vulnerabilities? Don't forget to consider system security and protection against malware.**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | The vault file stored on disk must be unreadable without the master password. An attacker who copies the vault file must gain nothing. Disclosure cannot be undone (ch1 p.5). | Attacker copies vault file from disk or backup → reads all passwords for all accounts. |
| **Data integrity** | Yes — critical | ch1 p.34 | Tampering with the vault file must be detectable. A modified vault (corrupted credentials, substituted passwords) is worse than no vault — the user trusts and uses wrong passwords. | Malware silently alters stored passwords; user is unknowingly locked out of accounts or directed to attacker-controlled credentials. |
| **Authentication** | Yes — critical | ch1 p.22 | Only the legitimate master-password holder may unlock and read the vault. | Any user on the device — or any malware with filesystem access — opens the vault and reads all credentials. |
| **Availability** | Yes | ch1 p.42 | The vault must be accessible whenever the user needs to log in to a service. Vault corruption or key loss leaves the user locked out of all accounts simultaneously. | Master password forgotten or vault corrupted → all account credentials inaccessible; recovery requires resetting every account individually. |

### Part 2 — Core Design: Local-Only vs Cloud-Synced

This case specifies a **local device** password manager — no cloud server is involved. This differs from case 5 (cloud password vault) in a critical way: there is no server-side authentication, no server holds any key material, and no network communication is required for normal operation. The entire security architecture lives on the device.

**Consequence**: authentication is local — the master password is verified by attempting to decrypt the vault and checking the GCM authentication tag. There is no server to challenge or to block brute-force attempts beyond what the local device enforces.

### Part 3 — Vault Encryption: Why AES-256-GCM

**Why AES-256 and not AES-128?** The vault may be stored as a file for years or decades. Against Grover's quantum algorithm (ch2 PQCrypto p.16), AES-128 provides only 64-bit effective security — obsolete for long-lived stored data. AES-256 retains 128-bit effective security post-quantum.

**Why GCM and not CBC?**

| Mode | Confidentiality | Integrity | Padding oracle | Passes over data |
|---|---|---|---|---|
| AES-256-CBC + HMAC | Yes | Requires separate HMAC | Vulnerable | Two |
| AES-256-GCM | Yes | Built-in 128-bit tag | None | One |

AES-256-GCM (ch2.2.3 p.70–75) is AEAD — the 128-bit GCM authentication tag simultaneously proves confidentiality and integrity. If the wrong master password is entered, the wrong `vault_key` is derived, the GCM tag verification fails, and decryption is aborted. This tag failure is the primary local authentication mechanism — no separate password hash needed for verification.

**Why not store a password hash for verification separately?** Storing `SHA-512(master_password)` alongside the vault would allow offline brute-force attacks: an attacker copies both the hash and the vault file, cracks the hash offline (no attempt limits), then uses the recovered master password to decrypt the vault. Using the GCM tag as the sole authentication mechanism means the attacker must attempt full vault decryption with each candidate password — providing no additional information beyond success/failure.

The vault structure:

```
vault_file = {
    salt,                               // 96-bit random, unique per installation
    nonce,                              // 96-bit random, new on every save
    ciphertext = AES-256-GCM encrypt(vault_key, plaintext_entries, nonce),
    GCM_tag                             // 128-bit authentication tag
}
```

On unlock attempt:
```
vault_key = HMAC(master_password, salt)    // key derivation
plaintext = AES-256-GCM decrypt(vault_key, ciphertext, nonce, GCM_tag)
If GCM_tag verification fails → wrong password → refuse access
```

### Part 4 — Master Password Key Derivation: Why HMAC

```
vault_key = HMAC(master_password, salt)
```

**Why not `SHA-256(master_password || salt)`?** Length-extension attacks (ch2.2.3 p.24–32): knowing `SHA-256(master_password || salt)` allows computing `SHA-256(master_password || salt || padding || X)` without knowing the master password. HMAC's inner-outer double hash construction prevents this.

**Why not use the raw master password as AES key?** A master password is a human-chosen string of low entropy — perhaps 40–60 bits for a strong password. AES-256 requires 256 uniformly random bits. Using the password directly means the effective key space is the password space (2⁴â°–2⁶â°), not 2²âµ⁶. The HMAC with a unique salt maps the password to a full 256-bit output for the given salt.

**The salt** (96-bit random, stored in vault_file) ensures that two installations with the same master password produce different vault_keys. It also ensures that precomputed rainbow tables against the master password are useless.

**Why HMAC-SHA256 and not HMAC-SHA512?** HMAC-SHA256 produces 256 bits of output — exactly the key size needed for AES-256. HMAC-SHA512 produces 512 bits and would be truncated; the extra computation is unnecessary. HMAC-SHA256 is correct here.

### Part 5 — Master Password Verification and Brute-Force Protection

Because authentication is local (GCM tag failure), an attacker with the vault file can attempt brute-force offline with no server-imposed limits. Mitigations:

1. **Encourage a strong master password**: the master password is the single key to everything; it must be long and complex.
2. **Local attempt counter** (ch3.7 p.85): the application enforces a delay or lockout after N wrong attempts. A sophisticated attacker bypasses this by running their own decryption code — but it raises the cost for opportunistic attacks.
3. **The HMAC construction as a cost factor**: each decryption attempt requires one HMAC computation + one AES-GCM decryption. An attacker iterating through a wordlist pays this cost per attempt.

**Note on more expensive KDF**: a memory-hard KDF (e.g., making HMAC computation deliberately slow) would further raise the cost per attempt. The slides reference SHA-512 + salt (ch3.2 p.11) as the improved scheme; a slow KDF is an improvement beyond this baseline.

### Part 6 — Individual Entry Structure

Each vault entry stores:

```
entry = { service_name, username, password, notes, url }
```

All entries are encrypted together in one AES-256-GCM operation. The vault_key protects the entire vault.

**A fresh nonce is generated every time the vault is saved** — not reused. Reusing a (key, nonce) pair in AES-GCM catastrophically breaks confidentiality; the nonce is therefore generated randomly (96-bit) on each write and stored in the vault_file header.

### Part 7 — Transfer to a New Device

When the user acquires a new device, the vault must be transferred securely. Several options exist:

**Option A — Physical media (USB drive, encrypted archive):** The vault file is already AES-256-GCM encrypted — it can be safely copied to any medium. The vault file alone, without the master password, is useless. Copy the file; enter the master password on the new device; the application derives the same vault_key from the same master password and salt → successful decryption.

**Why this is safe**: the vault_key is never transmitted — only the encrypted vault is. The master password is entered locally on each device. No key material is ever on the USB drive.

**Option B — Direct device-to-device transfer over TLS 1.3** (ch3.6 p.7–8): both devices connect to a local network or direct link. A fresh TLS 1.3 session is established between source and destination devices. The vault file (already encrypted) is transferred inside the TLS channel. ECDHE (ch3.6 p.18) ensures forward secrecy (ch3.6 p.37).

**Why TLS is still useful if the vault is already encrypted?** TLS prevents a network observer on the local network from capturing the vault file. The vault is protected by the master password alone — a strong password provides good protection, but belt-and-suspenders confidentiality is appropriate. TLS adds essentially zero operational cost.

**Authentication for the TLS transfer**: the source device generates a short pairing code (HMAC-SHA256 of a random nonce, truncated to 6 decimal digits). The user reads this code from the source device's screen and types it on the destination device's import screen. This out-of-band verification ensures the TLS connection is between the right two devices and not a man-in-the-middle (ch3.1 p.7 — challenge-response principle applied to device pairing).

**Option C — Export as QR code chain**: for small vaults. Not scalable; impractical for large vaults. Rejected as primary method.

**Chosen: physical media (Option A) as primary**, with Option B as an alternative for users who prefer network transfer. The security property is identical — the vault file is always protected by the master password regardless of how it is transported.

### Part 8 — System Security

**Memory protection**: the vault_key and plaintext entries must be zeroed from memory when the vault is locked. If the OS allows memory dumps (e.g., crash reports, swap files), the vault_key could be recovered. The application should use OS APIs for secure memory zeroing and mark sensitive buffers as non-swappable where available.

**Screen protection**: displayed passwords should not appear in screenshot captures or accessibility APIs that could be read by other applications.

**Clipboard security**: passwords copied to clipboard should auto-clear after a short timeout (e.g., 30 seconds). Clipboard contents are accessible to all applications on the device.

**EPP on the device** (ch3.7 p.43): behaviour-based malware detection to detect keyloggers, screen scrapers, and memory dumpers that target the master password or vault contents.

**Vault file access control** (ch1 p.30): the vault file should be stored with OS-level file permissions restricting read access to the owner account only. Other user accounts on the same device cannot read the vault file.

**Auto-lock**: the vault should automatically lock after a configurable inactivity period — the vault_key is zeroed from memory, requiring master password re-entry. Limits exposure if the device is left unattended.

### Part 9 — Remaining Vulnerabilities

- **Keylogger on device**: captures the master password at unlock time. The attacker now has vault_key = HMAC(master_password, salt) and can decrypt the vault. EPP (ch3.7 p.43) reduces but does not eliminate this risk. The master password is always the ultimate secret — if it is captured, the vault is compromised.
- **Memory scraping**: sophisticated malware reads the process memory of the password manager while the vault is unlocked, extracting plaintext credentials or the vault_key. OS-level memory isolation mitigates this; privileged malware can bypass it.
- **Shoulder surfing**: an observer watches the user type the master password or reads credentials from the screen. Physical security of the device and screen privacy matters.
- **Vault file backup exposure**: if the device backup (e.g., to cloud storage) includes the vault file, the backup must also be secured. The vault file is protected by the master password — but if the same master password is weak, both the vault and the backup are vulnerable.
- **Master password loss**: if the master password is forgotten and no recovery mechanism exists, all stored credentials are permanently inaccessible. The user must then reset every account individually. This is the price of strong local-only encryption — there is no recovery path by design.

### Part 10 — Comparison with Case 5 (Cloud Password Vault)

| | Case 31 — Local password manager | Case 5 — Cloud password vault |
|---|---|---|
| **Vault storage** | Local device only | Cloud server (encrypted) |
| **Authentication** | GCM tag failure = wrong password | Challenge-response against server |
| **Brute-force limits** | Local application-enforced delay | Server-enforced lockout |
| **Device transfer** | Copy vault file or local TLS | Automatic via cloud sync |
| **Server breach risk** | None — no server | Server holds ciphertext; encryption limits exposure |
| **Key on server** | Never | Never (client-side encryption) |
| **Multi-device sync** | Manual transfer | Automatic |

### Part 11 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Vault encryption | AES-256-GCM with fresh 96-bit nonce per save | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | AEAD in one pass; 128-bit post-quantum security; GCM tag = authentication mechanism; CBC adds padding oracle risk |
| Master password key derivation | HMAC(master_password, salt) | ch2.2.3 p.63–66 | HMAC prevents length extension; spreads password entropy over 256-bit key space; raw password as key has low entropy |
| Local authentication | GCM tag verification on decryption | ch2.2.3 p.70–75 | No separate password hash stored = no hash to brute-force independently; correct password → successful decryption |
| Vault transfer | Copy encrypted vault file (protected by master password alone) | ch1 p.15; ch3.6 p.7–8 | Vault ciphertext is safe to copy to any medium; master password never transmitted; TLS optional belt-and-suspenders for network transfer |
| Malware protection | EPP + memory zeroing on lock + auto-lock timeout | ch3.7 p.43, p.46 | Keylogger/memory scraper defence; reduces exposure window |

### Sources

- IS_UG_1_Introduction (p.5, p.10, p.15, p.22, p.34, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.63–66, p.70–75)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16–17)
- IS_UG_3_1_Appl_Basics (p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11)
- IS_UG_3_6_Appl_TLS (p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.85)

_Status: Complete_  
_Done by: William_
