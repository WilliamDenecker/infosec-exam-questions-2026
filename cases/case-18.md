# Case 18

An online exam platform combines webcam monitoring, microphone input, screen capture, and AI-based behaviour analysis. The platform stores recordings for later review.

**Design the security architecture for this platform. What security services are essential? How would you protect confidentiality, integrity, access control, and privacy of students? Which cryptographic mechanisms would you choose? What legal and ethical constraints apply?**

## Answer

### Part 1 — Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Recordings contain biometric data (face, voice) and exam answers — among the most sensitive personal data. Only authorised reviewers may access them. | A storage breach exposes the biometric data and answers of every student who sat an exam. GDPR Art. 9 violation; personal and reputational harm. |
| **Data integrity** | Yes — critical | ch1 p.34 | Recordings serve as evidence of exam conduct. Tampering must be detectable — a modified recording could falsely accuse or falsely exonerate a student. | An attacker modifies a recording to fabricate cheating behaviour; the student is penalised on falsified evidence. |
| **Authentication** | Yes — critical | ch1 p.22 | The identity of the student starting the session must be cryptographically verified. Reviewers accessing recordings must also be authenticated. | A student sits the exam under another student's identity. An unauthorised person accesses stored recordings. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | Students may not access any recordings (including their own). Only the responsible professor and authorised proctors may access a student's session recording. | A student accesses their own recording before an integrity investigation, compromising the evidence. |
| **Non-repudiation** | Yes | ch1 p.40 | A student must not be able to deny submitting specific answers or exhibiting specific behaviour if the recording proves it. The authenticity of recordings must be provable in academic proceedings. | A student denies the recording is authentic. Without a cryptographic signature, there is no proof the recording was not fabricated. |
| **Availability** | Yes | ch1 p.42 | The platform must remain fully operational during the exam period. Downtime during an exam prevents students from completing it — a serious academic consequence. | Platform failure mid-exam invalidates all ongoing exam sessions. Students cannot submit answers. |

### Part 2 — Student Authentication

The student authenticates before the exam starts using their **X.509 certificate** from a university-issued smart card (ch3.2 p.28–29). This binds their verified identity to the session cryptographically.

```
Step 1 — Server → Student:  { challenge nonce, exam_id, timestamp }   (ch3.1 p.7)
Step 2 — Student:           ECDSA_sign(private_key, { nonce || student_id || exam_id || timestamp })
Step 3 — Student → Server:  signature
Step 4 — Server:            verifies signature against student's registered X.509 public key
                            if valid + nonce fresh → session started
```

**Why X.509 certificates over username/password?** Passwords can be shared or phished — a different student could log in with someone else's credentials. An X.509 certificate is bound to a smart card; the private key is protected by a PIN. The server verifies that the holder of the registered key signed the challenge — not just that someone knew a password (ch3.2 p.28).

All connections over **TLS 1.3** (ch3.6 p.5, p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37). Forward secrecy ensures recording streams transmitted during the exam cannot be decrypted retroactively even if the server's long-term key is later compromised.

### Part 3 — Recording: Confidentiality and Integrity

All streams (webcam, microphone, screen capture) are encrypted in transit by TLS 1.3. At the storage server, each recording segment is encrypted at rest with **AES-256-GCM** (ch2.2.3 p.70–75) under a session-specific key. GCM provides both confidentiality and integrity — the 128-bit authentication tag on each segment detects any tampering.

**Why AES-256 over AES-128?** Against Grover's quantum algorithm, AES-128 provides only 64-bit effective security — obsolete (ch2 PQCrypto p.16). Biometric recordings stored for years until academic proceedings conclude must resist future decryption.

**Non-repudiation of recordings** (ch1 p.40): at the end of the exam session, the server computes a **SHA-256** hash (ch2.2.3 p.24–32) over the complete recording archive for that student and produces a **digital signature** using the platform's **ECDSA P-256** key (ch2.2.3 p.85–87):

```
signature = ECDSA_sign(platform_key, SHA-256(recording_archive || student_id || exam_id || timestamp))
```

This signed digest is stored immutably alongside the recording. It proves:
1. The recording existed in this exact form at the time of signing.
2. The platform — not a third party — authenticated it.

Any modification to the recording after signing invalidates the SHA-256 hash and thus the signature — detectable by any verifier during an academic investigation.

### Part 4 — Access Control

Role-based access control (ch1 p.30):
- **Students**: zero access to any recording (including their own, to prevent coaching based on prior session behaviour).
- **Professors**: read-only access to recordings of their own exam course only.
- **Proctors**: read-only access to recordings flagged for review by the AI analysis system.
- **Platform administrator**: manages accounts and access roles but cannot access recording content — separation of duties.

All access to recordings is logged with timestamp, reviewer identity, and which recording was accessed. Logs are append-only and stored on a separate system.

### Part 5 — System Security

**Packet filter** (ch3.7 p.51): the recording storage server is not directly reachable from the internet. Only the exam front-end server (in a DMZ) can write recordings; only the reviewer interface can read them, both over authenticated internal connections.

**Separation** (ch3.7 p.46): exam front-end, AI behaviour analysis engine, and recording storage are separate systems. A compromise of the AI engine does not expose recordings.

**IDS** (ch3.7 p.77, p.85): detect anomalous access — a reviewer downloading many recordings outside scheduled review periods triggers a threshold alert (ch3.7 p.85). Retain logs for retroactive investigation (ch3.7 p.83, p.96).

**EPP on all servers** (ch3.7 p.43): behaviour-based malware detection to prevent platform compromise.

### Part 6 — Legal and Ethical Constraints

- **GDPR Art. 9**: facial images and voice recordings constitute **biometric data** — special category personal data requiring explicit informed consent or a specific legal basis before collection.
- **Purpose limitation**: recordings may only be used for academic integrity review — not for AI model training, research, or any other purpose.
- **Retention limits**: recordings must be deleted after the academic appeal period expires. Automated deletion must be enforced by the platform.
- **Right of access**: under GDPR, a student may request a copy of their own recording. The platform must implement this, subject to safeguards preventing pre-investigation access.
- **Breach notification**: a breach of a recording database involving biometric data must be notified to the data protection authority within 72 hours (GDPR Art. 33) and to affected students if the risk is high (Art. 34).
- **Proportionality**: continuous webcam + microphone + screen capture is highly intrusive. This level of surveillance must be proportionate to the academic integrity risk — lighter measures (randomised question pools, in-person exams) must be considered first.
- **Transparency**: students must be clearly informed before the exam what data is collected, how it is used, who has access, and how long it is retained.

### Part 7 — Remaining Vulnerabilities

- **Endpoint malware on student device**: malware captures exam answers directly from the screen before encryption, bypasses the webcam, or displays answers from a hidden window outside the capture area. Cryptography at the transport and storage layers cannot prevent this — it requires operating system integrity checking on the student's device.
- **False positives from AI analysis**: AI flags innocent movements as suspicious behaviour. A wrongly accused student suffers significant harm. Human review of all AI flags is mandatory before any academic consequence.
- **Recording key management breach**: despite AES-256-GCM encryption of recordings, a breach of the key management system exposes all stored recordings. The key management service must be the most hardened component in the architecture.

### Part 8 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Student authentication | X.509 smart card + ECDSA challenge-response | ch3.2 p.28–29; ch3.1 p.7 | Cryptographic identity binding; smart card PIN required; cannot be shared |
| Recording encryption | AES-256-GCM per session | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | AEAD; quantum-safe; tamper-evident per segment |
| Non-repudiation | ECDSA P-256 signature over session hash + timestamp | ch2.2.3 p.85–87; ch1 p.40 | Platform-signed proof; modification detectable; provable in proceedings |
| Access control | Role-based; append-only audit log | ch1 p.30; ch3.7 p.83 | Structural enforcement; no student can access any recording |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Forward secrecy; biometric streams confidential even against future key compromise |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.40, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.70–75, p.85–87)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.28–29)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.43, p.46–47, p.51, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
