# Case 24

Consider a system to input exam scores in a university. The goal is that only lecturers can input scores for their own courses (i.e. no access to students or external actors, and no access to another lecturer's course). Furthermore lecturers should also be able to input scores from home.

**Suggest an appropriate security solution (don't forget system security). Which security protocols, cryptographic algorithms, etc. would you use?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions. Be sufficiently specific in your implementation (algorithms, key lengths, modes, etc.).*

## Answer

### Part 1 — Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | The system must verify that the person submitting scores is a legitimate lecturer — not a student, external party, or credential-stuffing attacker. | Any authenticated user can submit scores for any student. Students modify their own grades. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | A lecturer may only read and write scores for their own courses. Students have zero write access. This must be enforced by the system, not by trust. | Lecturer A overwrites Lecturer B's scores. A student directly sets their own grade to a pass. |
| **Data integrity** | Yes — critical | ch1 p.34 | A submitted score must arrive unchanged. Tampering in transit (10 → 20) causes direct harm to students and academic integrity. | A man-in-the-middle changes a failing grade to a passing grade; the tampered record is stored without detection. |
| **Non-repudiation** | Yes | ch1 p.40 | A lecturer must not be able to deny having submitted or modified a specific score. Cryptographic evidence must be retained for dispute resolution. | A lecturer submits a wrong score, then claims someone else modified it. No cryptographic evidence exists to contradict them. |
| **Confidentiality** | Yes | ch1 p.15 | Scores in transit must not be readable by third parties. Scores are personal data under GDPR — interception causes both privacy harm and legal liability. | An eavesdropper reads the scores of all students in a course submitted over an unencrypted connection. |

### Part 2 — Remote Access: Why IPSec VPN and Not Just TLS

The core requirement is that lecturers must be able to submit scores from home. Several approaches exist:

**Option A — Expose the score server directly on the internet (TLS only):** The score submission server gets a public IP and runs HTTPS. Lecturers connect directly. The server is permanently exposed to port scans, automated exploitation attempts, and denial-of-service. Any vulnerability in the web application or its stack is exploitable from anywhere on the internet. **Rejected**: violates the principle of minimising attack surface (ch3.7 p.46).

**Option B — SSH tunnelling:** Lecturers SSH into a university jump host and then reach internal systems. SSH provides a secure channel but is not designed for general-purpose application traffic and requires lecturers to manage SSH keys and tunnel configuration. **Rejected**: operationally complex; not the right tool for this use case.

**Option C — IPSec VPN (chosen):** Lecturers connect to the university VPN gateway. The VPN gateway is the only internet-facing component. Once the tunnel is established, the lecturer's device appears to be inside the university network. The score server has no public IP and is completely unreachable from the internet without an authenticated VPN session.

**IPSec tunnel mode** (ch3.4 p.26–28) between the lecturer's device and the university VPN gateway. Tunnel mode encapsulates the entire original IP packet — the internal topology (score server address, internal network structure) is hidden from any internet-level observer (ch3.4 p.27).

**ESP with AES-256-GCM** (ch3.4 p.24): Encapsulating Security Payload provides both confidentiality (AES-256) and integrity (GCM authentication tag) in a single pass. **Why AES-256 and not AES-128?** Against Grover's quantum algorithm, AES-128 provides only 64-bit effective security — insufficient for data that may be retained for years (ch2 PQCrypto p.16). AES-256 retains 128-bit effective security post-quantum.

**IKEv2** (ch3.4 p.34–41) for key exchange: establishes the IPSec security associations, negotiates algorithms, and provides mutual authentication. IKEv2 supports **ephemeral Diffie-Hellman** (ch2.2.4 p.10) for **forward secrecy** (ch3.6 p.37) — even if the VPN gateway's long-term key is later compromised, past recorded traffic cannot be decrypted.

#### VPN Authentication: Why X.509 Certificates and Not Pre-Shared Keys

**Option A — Pre-shared key (PSK):** All lecturers share the same secret. One compromised or leaving lecturer requires a key rotation affecting all users simultaneously. PSK cannot be individually revoked. **Rejected**: does not scale and cannot be revoked per user.

**Option B — Username/password for VPN authentication:** Possible but passwords can be phished or brute-forced at the VPN gateway, which is internet-facing. **Rejected**: weaker than certificate-based authentication for machine-level VPN auth.

**Option C — X.509 certificates (chosen)** (ch3.2 p.28–29): one certificate per lecturer device, issued by the university's internal CA. The certificate is bound to the lecturer's identity and device. When a lecturer leaves, their certificate is added to the Certificate Revocation List (ch3.2 p.50–53) — other lecturers are unaffected. IKEv2 uses the certificate for mutual authentication: the lecturer's device authenticates to the gateway, and the gateway authenticates to the device.

### Part 3 — Score Submission Application: TLS and Two-Factor Authentication

Inside the VPN, the score submission application runs over **TLS 1.3** (ch3.6 p.7–8) with cipher suite `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).

**Why TLS on top of IPSec (defence-in-depth)?** IPSec protects the network layer. TLS protects the application layer independently. If the VPN layer is misconfigured or compromised, TLS still protects the score data. The two layers use different keys and different authentication — compromising one does not compromise the other (ch3.7 p.46 — layered defence).

#### Factor 1: X.509 Client Certificate

The same certificate used for VPN authentication also authenticates the lecturer to the web application (mutual TLS). The certificate contains the lecturer's staff ID as a subject attribute. The server verifies the certificate against the university CA.

**Why not rely on the VPN session alone?** The VPN proves the device is authorised; it does not prove which individual is at the keyboard. The application-level certificate check is a separate step.

#### Factor 2: Challenge-Response for Account Password

**Why not send the password directly?** If the password travels over the network — even inside TLS — a stolen stored hash could be replayed in a pass-the-hash attack. Challenge-response defeats this: the server sends a fresh nonce, and the client proves knowledge of the hash without transmitting it.

**Why not just accept the hash directly?** A stolen `stored_hash` from the server database could be used directly to compute the correct response for any nonce — this is the pass-the-hash attack. The server must verify the response against `stored_hash`, not accept `stored_hash` itself as the response. The protocol is:

```
Step 1 — Server → Lecturer:   { salt, nonce }
Step 2 — Lecturer computes:   response = HMAC(SHA-512(salt || password), nonce)
Step 3 — Lecturer → Server:   { response }
Step 4 — Server verifies:     HMAC(stored_hash, nonce) == response
                               match → Factor 2 passed
```

Password stored as `SHA-512(salt || password)` with 96-bit random salt per account (ch3.2 p.11).

**Why SHA-512 and not SHA-256?** SHA-512 provides 256-bit preimage resistance; against Grover's algorithm that reduces to 128-bit — still secure (ch2 PQCrypto p.17). SHA-256 provides only 128-bit preimage resistance, reducing to 64-bit post-quantum — insufficient for a long-lived password hash.

**Why SHA-512 and not a memory-hard function?** The slides cover SHA-512 + salt as the improved scheme (ch3.2 p.11); memory-hard functions are referenced as an improvement direction but not specified in detail in the slides. SHA-512 + 96-bit salt is the chosen baseline from slide material.

The combination of X.509 certificate (something-you-have) and password (something-you-know) constitutes MFA (ch3.7 p.20, p.46). MFA is absent in 59% of security incidents (ch3.7 p.20).

### Part 4 — Access Control

**Role-based access control** (ch1 p.30) enforced at both application and database level:

| Role | Permissions |
|---|---|
| Lecturer | Read and write scores for own courses only |
| Student | No access to score submission system whatsoever |
| Administrator | Manage accounts; cannot modify scores directly (separation of duties) |

**Enforcement**: before any read or write operation, the server checks `course_owner(course_id) == authenticated_lecturer_id`. This is enforced in the application layer, not just the UI. A lecturer cannot bypass the check by crafting a direct API request.

**Why not rely on the UI alone?** An attacker who obtains a valid session token can craft HTTP requests directly to the API, bypassing the web UI entirely. Server-side enforcement is mandatory.

**Separation of systems** (ch3.7 p.46): the score submission server and the student-facing results portal are separate applications on separate servers. A vulnerability in the student-facing portal cannot reach the score database.

### Part 5 — Non-Repudiation and Timestamp Integrity

#### Chosen Signing Approach

Every score submission is **digitally signed** by the lecturer using their ECDSA P-256 private key (ch2.2.3 p.85–87):

```
signature = ECDSA_sign(lecturer_private_key,
            SHA-256(student_id || course_id || score))
```

**Why ECDSA P-256 and not RSA-PSS?** RSA-PSS (ch2.2.3 p.88–93) achieves equivalent security with a 2048-bit key but produces 256-byte signatures and requires significantly more computation. ECDSA P-256 produces 64-byte signatures at 128-bit security, using the same keypair already present in the client certificate — no additional infrastructure required.

**Why ECDSA and not HMAC for non-repudiation?** HMAC uses a shared secret key (ch2.2.3 p.63–66). If the lecturer and the server share K, either party can produce a valid HMAC — the server itself could forge a submission and the lecturer would have no way to prove they did not submit it. Non-repudiation (ch1 p.40) requires an asymmetric scheme: only the lecturer holds the private key, so only the lecturer could have produced the signature. HMAC cannot provide non-repudiation.

**Why not a plain hash (SHA-256)?** A plain hash has no secret key — anyone who knows the inputs can compute the same hash. It proves nothing about the identity of the submitter.

#### Timestamp Security: The Backdating Problem

**Why the timestamp is NOT in the lecturer's signature:** If the lecturer's device provided the timestamp and it was covered by the signature, a lecturer could backdate a submission by setting their system clock to before the submission deadline. The ECDSA signature would verify as valid, but the timestamp inside it would be false. The lecturer controls their own clock, so a lecturer-provided timestamp provides no integrity guarantee.

**Solution — Server-side timestamp on receipt:**

```
received_at = server_timestamp   (applied by server on arrival; not provided by client)
stored_record = { student_id, course_id, score, signature, received_at }
```

The server records when the submission arrived. The server is trusted infrastructure — its clock cannot be manipulated by the lecturer.

**Stronger alternative — TTP timestamp server** (ch3.1 p.6): after receiving the submission, the server requests a countersignature from a Trusted Third Party timestamp authority:

```
TTP_token = TTP_sign(SHA-256(signature || received_at))
```

The TTP's countersignature cryptographically proves the submission existed at the stated time. Even the university server cannot retroactively change the timestamp — the TTP's signature would no longer verify.

**Why is the TTP approach stronger than server-side only?** With server-side timestamp alone, a dishonest server administrator could in theory alter the recorded `received_at` field. The TTP's countersignature is an immutable external proof — the TTP has no incentive to cooperate with falsification.

Audit logs are append-only. The ECDSA signature proves authorship (lecturer cannot deny); the server timestamp and TTP token prove time (lecturer cannot backdate).

### Part 6 — System Security

**Packet filter** (ch3.7 p.51): only the VPN gateway is reachable from the internet (UDP 500 and 4500 for IKEv2). The score database and submission web server have no internet-facing ports whatsoever.

**Application-level gateway (proxy)** (ch3.7 p.62–65): in front of the score submission web server inside the internal network. Inspects HTTPS requests for malformed inputs, SQL injection attempts, and oversized payloads before forwarding to the application server.

**Separation** (ch3.7 p.46): the score submission server, the student results portal, and the database are separate systems on separate network segments. A compromise of the student-facing portal cannot reach the score database directly.

**IDS** (ch3.7 p.77, p.85): detect anomalous access patterns — a lecturer submitting hundreds of scores in seconds, logins outside working hours, VPN connections from unexpected geographic locations. Threshold detection (ch3.7 p.85) triggers alerts. Retain logs for retroactive forensic analysis (ch3.7 p.83, p.96).

**EPP on servers** (ch3.7 p.43): behaviour-based malware detection on server infrastructure.

### Part 7 — Remaining Vulnerabilities

- **Compromised lecturer device**: malware on the lecturer's home PC can capture credentials and the private key (if the key is software-stored on the device). The ECDSA signature on a fraudulently submitted score is traceable to the lecturer's certificate, but EPP on the endpoint (ch3.7 p.43) is needed for prevention. A hardware security token holding the private key (smartcard) would prevent key extraction.
- **Insider threat — intra-course fraud**: a legitimate lecturer can submit wrong scores for their own course. Access control prevents cross-course fraud but not intra-course manipulation. The ECDSA signature and TTP timestamp provide deterrence and retroactive detection — any fraudulent submission is cryptographically attributed to the responsible lecturer.
- **Score correction race condition**: without a controlled workflow, a lecturer could submit a score, wait for publication, then submit a correction with a backdated timestamp if the TTP mechanism is absent. Server-side timestamp enforcement and a formal correction workflow (e.g. administrator approval required for any change after the submission window closes) prevent this.
- **VPN gateway as single point of failure**: all remote access routes through the VPN gateway. If it is unavailable, no remote submissions are possible. Redundant gateways mitigate this.

### Part 8 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Remote access | IPSec tunnel mode + ESP AES-256-GCM | ch3.4 p.24, p.26–28 | Score server not internet-facing; internal topology hidden; AEAD |
| VPN key exchange | IKEv2 with ephemeral DH | ch3.4 p.34–41; ch2.2.4 p.10 | Forward secrecy; session keys not tied to long-term credential |
| VPN authentication | X.509 certificates per lecturer | ch3.2 p.28–29, p.50–53 | Individually revocable; PSK would require global rotation on compromise |
| Application MFA | X.509 client cert + challenge-response password | ch3.2 p.28; ch3.1 p.7; ch3.7 p.20 | Two independent factors; pass-the-hash defeated; cert proves device identity |
| Password storage | SHA-512 + 96-bit salt | ch3.2 p.11 | Post-quantum preimage resistance (256-bit → 128-bit post Grover's) |
| Non-repudiation | ECDSA P-256 signature over submission content | ch2.2.3 p.85–87; ch1 p.40 | Only lecturer holds private key; HMAC cannot provide non-repudiation |
| Timestamp | Server-side on receipt + TTP countersignature | ch3.1 p.6 | Client-provided timestamp backdatable; TTP provides immutable external proof |
| Access control | Role-based, course-ownership enforced server-side | ch1 p.30 | UI bypass via direct API calls still blocked; structural impossibility |

### Sources

- IS_UG_1_Introduction (p.5, p.10, p.15, p.22, p.30, p.34, p.40, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.63–66, p.85–87, p.88–93)
- IS_UG_2_2_4_SecM_KeyExch (p.10)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16–17)
- IS_UG_3_1_Appl_Basics (p.3, p.6, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.28–29, p.50–53)
- IS_UG_3_4_Appl_IPSec (p.11, p.24, p.26–28, p.34–41)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.51, p.62–65, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
