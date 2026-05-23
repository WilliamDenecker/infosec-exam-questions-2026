# Case 7

A company has suffered ransomware attacks encrypting both production systems and online backups.

**Design a backup security architecture that is resilient against ransomware. Which security services are essential? Which cryptographic mechanisms, key-management practices, and system controls would you use? What risks remain?**

## Answer

### Part 1 — Essential Security Services (ch1 p.10)

The root failure was that ransomware reached both production and online backups. The backup architecture must guarantee:

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Availability** | Yes — critical | ch1 p.42 | The entire purpose of a backup is to restore operations after a disaster. If the backup is also destroyed, availability is permanently lost. | The company cannot recover; all data is gone. The ransomware attack succeeds completely. |
| **Integrity** | Yes — critical | ch1 p.34 | A silently corrupted or overwritten backup is useless at restore time — worse than no backup because it wastes recovery time. | The company restores corrupted data without realising it, compounding the disaster. |
| **Confidentiality** | Yes | ch1 p.15 | Backup media contains all company data. Physical theft or exfiltration of backup storage must not expose that data. | Physical theft of Tier 2/Tier 3 media reveals the company's entire dataset without any encryption barrier. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | The ransomware encrypted online backups because the backup repository was writable and deletable from production systems. Access must be reduced to append-only. | A compromised production server deletes or overwrites all online backup snapshots — the exact failure that occurred. |

### Part 2 — Architecture: Isolation and Append-Only Storage

The fundamental design principle is **separation**: the backup repository must never be reachable from any system that ransomware could compromise, and must enforce append-only semantics to existing data.

**Three backup tiers:**

**Tier 1 — Online incremental backup (separate network segment)**: the production server pushes encrypted snapshots to a backup server on an isolated network segment. The backup server accepts incoming writes (append) but production servers **cannot delete or overwrite** existing snapshots — this is enforced by access control at the backup server (ch1 p.30). A compromised production system therefore cannot destroy existing backups.

**Why append-only?** Ransomware operates by modifying or overwriting files. If the write credential used by the backup agent only allows appending new data — not modifying or deleting existing snapshots — even a fully compromised production server cannot destroy Tier 1 backups.

**Tier 2 — Offline air-gapped backup**: daily or weekly, snapshots are exported to removable media (tape or external disk) stored **physically disconnected** from all networks. Ransomware cannot reach storage with no network connection. This is the last line of defence for availability (ch1 p.42).

**Tier 3 — Geographic copy**: monthly offline backup stored at a separate physical location. Protects against fire, flood, or physical destruction of the primary site.

### Part 3 — Cryptographic Mechanisms

#### Part 3.1 — Backup Encryption

All backup data is encrypted with **AES-256-GCM** (ch2.2.3 p.70–75) before leaving the production system. GCM is an AEAD mode: it provides both confidentiality and integrity in a single operation. The 128-bit GCM authentication tag detects any corruption or tampering with ciphertext during storage or transport. A fresh random 96-bit nonce per backup chunk prevents nonce reuse.

**Why AES-256 over AES-128?** Against quantum adversaries (Grover's algorithm), AES-128 provides only 64-bit effective security — obsolete (ch2 PQCrypto p.16). AES-256 retains 128-bit effective security. For a backup system intended to protect data for years, AES-256 is the only defensible choice.

**Why GCM over CBC?** CBC-mode AES provides confidentiality only and requires a separate MAC for integrity. GCM provides authenticated encryption in one pass, is parallelisable, and has no padding oracle risk (ch2.2.3 p.70–75).

#### Part 3.2 — Dataset-Level Integrity with HMAC

In addition to per-chunk GCM tags, the backup agent computes **HMAC-SHA256** (ch2.2.3 p.63–66) over the entire backup manifest (the list of all chunks, their hashes, and timestamps).

**Why HMAC alongside GCM?** GCM authenticates each chunk *in isolation* — it detects tampering or corruption of a chunk's content, but it does not detect a **missing or deleted chunk**. If an attacker deletes an entire chunk from the manifest, each remaining chunk's GCM tag still passes verification. The HMAC over the complete manifest covers completeness: if any chunk is missing, the manifest HMAC fails, aborting the restore before any corrupted data is loaded.

Before restoration: (1) verify manifest HMAC first; (2) then verify each chunk's GCM tag on decryption.

### Part 4 — Key Management

The backup encryption key is **separate from all production system keys**. It is generated and stored on a dedicated key management host not reachable from production networks. The backup agent receives only a time-limited encryption capability — it can encrypt new backups but cannot export or access the raw key.

Critically: the backup encryption key is **not stored on production systems**. Ransomware that compromises a production server cannot reach the backup key. Offline backup media (Tier 2/3) is therefore useless to an attacker even if physically stolen.

The key itself is backed up separately on offline encrypted media under the control of at least two authorised personnel (split responsibility) — a single insider cannot access or destroy the key.

### Part 5 — System Security

**EPP and EDR on all endpoints** (ch3.7 p.38–43): ransomware has characteristic behaviour (mass rapid file encryption). Behaviour-based EDR (ch3.7 p.42) can detect and quarantine the ransomware process before it spreads to the backup tier.

**Packet filter** (ch3.7 p.51): the backup server is on an isolated network segment. The packet filter permits only the specific backup protocol from production servers inbound (write-only port). No outbound connections from the backup server to production. No direct internet access to the backup server.

**Principle of least privilege** (ch3.7 p.46 — minimise attack surface): the production server's backup agent runs with credentials permitting write-only append to the backup repository. It cannot list, read, or delete existing backups. This is the access control enforcement (ch1 p.30) that prevents ransomware from destroying existing snapshots.

**IDS** (ch3.7 p.77, p.85): monitor the backup server for anomalous deletion requests, unusually high write volumes outside scheduled windows (could indicate ransomware attempting to overwrite), and failed authentication attempts. Alert in real time (ch3.7 p.83) rather than logging silently.

### Part 6 — Remaining Risks

- **Slow-acting ransomware (APT-style)**: advanced ransomware may silently encrypt production files over weeks or months before triggering. All snapshots from that period contain encrypted data. The only defence is long retention of Tier 2 offline copies predating the infection — restoration requires accepting data loss proportional to dwell time.
- **Backup credential theft**: if an attacker gains write credentials for the Tier 1 backup server before append-only restrictions are enforced, they can submit garbage that overwrites nothing but fills the repository. HMAC verification on restore detects such corruption.
- **Physical theft of offline media**: Tier 2/3 media could be physically stolen. AES-256-GCM encryption renders the data unreadable; the risk to availability exists if the decryption key is also inaccessible, hence storing the key separately in a geographically different location.
- **Direct compromise of the backup server**: if the backup server's own OS or backup service is exploited, Tier 1 is lost. The air-gapped Tier 2 copies remain. This is why air-gapped offline backup is mandatory rather than optional.
- **Key management failure**: if the backup encryption key is lost or inaccessible (e.g. the key management host destroyed in the same incident as production), all encrypted backups become permanently unrecoverable. Secure offline key backup with geographic separation is therefore not optional.

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Backup encryption | AES-256-GCM per chunk | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | AEAD (integrity + confidentiality); quantum-safe at 256 bits; no padding oracle |
| Dataset completeness | HMAC-SHA256 over full manifest | ch2.2.3 p.63–66 | Detects deleted/missing chunks that per-chunk GCM tags cannot detect |
| Write access | Append-only from production | ch1 p.30 | Ransomware cannot delete or overwrite existing snapshots even with write credential |
| Offline backup | Air-gapped Tier 2/3 | ch1 p.42 | Ransomware cannot reach storage with no network connection — absolute last line |
| Ransomware detection | Behaviour-based EDR | ch3.7 p.42–43 | Detects mass encryption activity before it reaches the backup tier |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.30, p.34, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.63–66, p.70–75)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_7_Appl_System (p.38–43, p.46–47, p.51, p.77, p.83, p.85)

_Status: Complete_  
_Done by: William_
