# Case 3

Design the architecture for a regional/national healthcare data exchange system. This system must enable the exchange of patient records among hospitals, laboratories, and doctors. It should allow *authorized* healthcare professionals to access patient data. It should also allow patients to have a (possibly limited) view of their own personal medical data.

**What are the most essential security services? What security mechanisms would you use to implement those services (be sufficiently specific)? How would you secure the access to the service? What could be remaining vulnerabilities? Don't forget to consider system security and protection against malware. What are the legal aspects you need to take into account.**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Patient records are among the most sensitive personal data. Unauthorised disclosure cannot be undone (ch1 p.5). An ISP employee or attacker must not be able to read medical records. | Any breach of the system exposes irreversibly sensitive data. GDPR violations, patient harm from disclosed conditions. |
| **Authentication** | Yes — critical | ch1 p.22 | Every user (patient or professional) must be verified before any data is revealed. Both entity authentication (who is this person?) and data-origin authentication (who created this record?) apply. | A GP could impersonate a cardiologist; an attacker could impersonate any professional to access all records. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | A GP must not read psychiatric records; a lab must not read surgical notes. Clearance must be enforced by the system, not by trust. | Professionals access records they have no legitimate reason to see — insider threat, privacy violation, GDPR Art. 9 breach. |
| **Data integrity** | Yes — critical | ch1 p.34 | A tampered lab result or modified dosage can harm or kill a patient. Nothing may be added, deleted, or modified without detection. | A malicious or compromised system silently alters a blood type, allergy note, or medication dosage — patient harm. |
| **Non-repudiation** | Yes | ch1 p.40 | Every data access and every record modification must be attributable to a specific professional. They cannot deny having accessed or issued data. | A professional can deny falsifying a record; audit trail is worthless without cryptographic attribution. |
| **Availability** | Yes | ch1 p.42 | Medical data may be needed in emergencies. Downtime directly endangers lives — a treating physician without access to allergy information may administer a fatal dose. | Emergency treatment delayed or incorrectly administered due to inaccessible records. |

### Part 2 — Transport Security

Use **TLS 1.3** (ch3.6 p.7–8) for all connections between clients (portals, EHR systems) and the central platform. TLS is application-independent — the same layer secures both the professional portal and the patient portal without changes to the application (ch3.6 p.5).

**Cipher suite**: `TLS_AES_256_GCM_SHA384` with **ECDHE** key exchange (ch3.6 p.18) → forward secrecy: compromise of the server's long-term key does not expose past sessions (ch3.6 p.37). AES-256-GCM is an AEAD mode (ch3.6 p.13) providing confidentiality and integrity in one pass.

**Why TLS 1.3 over 1.2?** TLS 1.2 allows CBC-mode AES (vulnerable to padding oracle attacks) and static RSA key exchange (no forward secrecy). TLS 1.3 removes all legacy algorithms and mandates forward secrecy for every session (ch3.6 p.37).

### Part 3 — Data Confidentiality at Rest

All stored records are encrypted with **AES-256-GCM** (ch2.2.3 p.70–75). Record-level encryption (each record encrypted separately) enables role-based access: a user receives only the decryption key for records they are authorised to read, not a single database-wide key.

**Why AES-256 over AES-128?** Against quantum adversaries (Grover's algorithm halves key strength), AES-256 retains 128-bit effective security; AES-128 drops to 64 bits — obsolete (ch2 PQCrypto p.16). For a healthcare system storing data for decades, 256-bit is the only defensible choice.

### Part 4 — Data Integrity and Non-Repudiation

Every record created or modified by a healthcare professional is **digitally signed**. The signing algorithm is **ECDSA with P-256** (ch2.2.3 p.85–87), providing 128-bit security with compact 64-byte signatures. The professional's signing key is bound to their identity via an **X.509 certificate** issued by a national healthcare CA (ch3.2 p.28–29, ch3.1 p.19).

**Process**: `SHA-256(record)` → sign with professional's ECDSA private key → store signature alongside record. Any modification invalidates the signature (integrity, ch1 p.34). The signature ties the record to its issuer (non-repudiation, ch1 p.40).

**Why ECDSA P-256 over Ed25519?** Ed25519 is not covered in the slides. The slides explicitly cover ECDSA with ECC curves (ch2.2.3 p.85–87). ECDSA P-256 provides equivalent security and is widely standardised in existing healthcare PKI.

**Why ECDSA over RSA signatures?** RSA-2048 achieves equivalent security but with 256-byte signatures vs 64 bytes for ECDSA P-256. On a system signing every record access and modification, the size difference is significant (ch2.2.3 p.85–87).

Modifications do not overwrite the original: each change creates a new signed version. This provides a tamper-evident audit history.

### Part 5 — Authentication and Access Control

**Healthcare professionals** authenticate with a two-factor mechanism:
- Their government-issued X.509 certificate (ch3.2 p.28) — linked to their professional licence number (RIZIV). Proves identity.
- A PIN or biometric unlocks the private key stored on their smart card.

Before any login is accepted, the system checks the certificate against the CA's **Certificate Revocation List (CRL)** (ch3.2 p.50–53). Nightly synchronisation with the professional registry immediately revokes certificates of professionals whose licence has lapsed.

**Why X.509 certificates over passwords?** Passwords authenticate a device claim; X.509 certificates carry attributes (professional role, licence number, clearance) verifiable by the CA (ch3.2 p.28). A stolen password gives full access; a stolen certificate without the smart card PIN is useless.

**Patients** authenticate with their national eID card (also X.509 certificate, ch3.2 p.28). Strong identity assurance without bespoke credential infrastructure.

**Access control** (ch1 p.30) is role-based: a GP reads general records; a cardiologist accesses cardiology data; a patient reads their own records read-only. The patient can grant or revoke specific professionals' access to specific record categories (GDPR requirement). Role assignment is managed by the CA/directory, not editable by individual users.

### Part 6 — System Security

**Packet filter firewall** (ch3.7 p.47, p.51): permit only port 443 (HTTPS) inbound. Separate into three network zones — patient/professional portal (internet-facing), application layer, and database layer — with packet filters between each. The database is never directly reachable from the internet (ch3.7 p.46 — minimise attack surface).

**Why a DMZ architecture?** A single flat network means a compromise of the web portal immediately grants database access. With zone separation, an attacker who compromises the portal server still faces a second packet filter before reaching patient data (ch3.7 p.47).

**Application-level gateway (proxy)** in front of portals (ch3.7 p.62–65): inspects full HTTP requests, enforces authentication before forwarding, blocks malformed requests before they reach the application. Unlike a packet filter, the proxy understands HTTP semantics and can block attacks like SQL injection in request parameters.

**EPP and EDR** on all servers (ch3.7 p.38–43): signature-based and behaviour-based malware detection. Timely patching of all OS and middleware.

**IDS** (ch3.7 p.77, p.83, p.85, p.96): threshold detection for anomalous access (a professional downloading thousands of records outside working hours triggers alert, ch3.7 p.85). Retain audit logs long enough for retroactive investigation (ch3.7 p.96). All record accesses logged with timestamp, user identity, and record identifier — append-only logs.

### Part 7 — Legal Aspects

- **GDPR Art. 9**: medical records are special category data (health data). Processing requires explicit informed consent or a legal basis (treatment). Patients have the right to view (implemented via patient portal), rectify, object, and request erasure of data not required for ongoing treatment.
- **Purpose limitation**: records collected for treatment may not be used for research without separate informed consent. The access control system enforces this.
- **Breach notification**: a breach affecting patient data must be notified to the DPA within 72 hours (GDPR Art. 33) and to affected patients if the risk is high (Art. 34).
- **Data minimisation**: role-based access control and record-level encryption ensure professionals see only the data needed for the specific treatment context.

### Part 8 — Remaining Vulnerabilities

- **Insider threat**: a legitimate professional can access authorised records for illegitimate purposes (curiosity, insurance fraud). IDS (ch3.7 p.77) and audit logs (ch3.7 p.83) provide deterrence and retroactive detection, but cannot prevent access in real time.
- **Endpoint compromise**: malware on a professional's workstation captures decrypted data after the TLS and application layers have processed it. EPP (ch3.7 p.43) reduces but does not eliminate this risk.
- **CRL staleness**: if revocation checking fails open (network error), a revoked certificate may still be accepted. The system must fail closed — deny access if CRL is unreachable.
- **Availability / DDoS**: a national healthcare platform is a high-value target for disruption. Redundancy and traffic scrubbing are required; this is a residual architectural risk.

### Part 9 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Transport | TLS 1.3, AES-256-GCM-SHA384, ECDHE | ch3.6 p.7–8, p.18, p.37 | Mandatory forward secrecy; AEAD; no legacy algorithms |
| Data at rest | AES-256-GCM per record | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | Record-level enables role-based key access; AES-256 quantum-safe |
| Integrity + non-repudiation | ECDSA P-256 signature per record | ch2.2.3 p.85–87 | Compact signatures; provable attribution; any modification detectable |
| Identity | X.509 certificates from healthcare CA | ch3.2 p.28–29, ch3.1 p.19 | Carries professional attributes (role, licence); cryptographically verified |
| Access control | Role-based, certificate attribute driven | ch1 p.30 | System-enforced; cannot accidentally grant wrong access |
| Network | DMZ + packet filter + application proxy | ch3.7 p.47, p.51, p.62–65 | Layered defence; proxy catches semantic attacks packet filter misses |

### Sources

- IS_UG_1_Introduction (p.5, p.10, p.15, p.22, p.30, p.34, p.40, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.70–75, p.85–87)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.16–22)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.28–29, p.50–53)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.13, p.18, p.37)
- IS_UG_3_7_Appl_System (p.38–43, p.46–47, p.51, p.62–65, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
