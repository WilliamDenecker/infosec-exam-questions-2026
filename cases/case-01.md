# Case 1

In March 2025 a security incident occurred within the U.S. administration. Several high-ranking U.S. officials discussed detailed imminent military operations, including specific information about airstrikes (e.g. timing) over regular personal communication devices in a group chat on the Signal messaging app. Furthermore, a journalist was invited by mistake into the group chat.

The information discussed was classified information, that might have endangered the security of the aircraft pilots if it had been revealed to hostile forces.

**Explain which security services would be essential for such a discussion about classified information. Which security services might have been breached in this incident? What improvements would you suggest to the approach that had been chosen during this incident?**

## Answer

### Part 1 — Essential Security Services (ch1 p.10)

All six security services are essential here. Discussing classified military operations demands the strongest possible guarantees across every dimension.

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Classified data (timing, targets, pilot identities) may only reach authorised personnel. Unauthorised release of this information cannot be undone (ch1 p.5). | A journalist or hostile intelligence agency reads the entire operation plan. Aircraft pilots are exposed to lethal risk. |
| **Authentication** | Yes — critical | ch1 p.22 | Every participant must be verified — entity authentication (is this really General X, not an impersonator?) and data-origin authentication (did this order actually come from the Secretary of Defense?). | An attacker who impersonates a senior official can inject false orders, cause friendly fire, or redirect forces. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | Only people with the correct security clearance may join the channel. The system must enforce this, not rely on a human admin doing the right thing. | Any group admin can add any phone contact — as demonstrated, an uncleared journalist was added. The system allowed it. |
| **Data integrity** | Yes — critical | ch1 p.34 | Military orders must arrive exactly as sent — nothing modified, replayed, or injected. A tampered timing instruction could cause friendly fire or abort a successful operation. | A man-in-the-middle modifies a strike time by one hour; pilots arrive to unexpected resistance. |
| **Non-repudiation** | Yes | ch1 p.40 | Every order and every message must be attributable to its sender. Sender cannot deny having sent it; receiver cannot deny having received it. This supports accountability and command chain integrity. | A participant denies having authorised an order. With disappearing messages and no signatures, there is no evidence. |
| **Availability** | Yes | ch1 p.42 | The channel must remain operational throughout the operation. A communication blackout while aircraft are airborne is operationally catastrophic. | Pilots cannot receive updated orders, abort instructions, or emergency changes — lives and aircraft are lost. |

### Part 2 — Which Services Were Breached?

**Confidentiality — breached (primary breach)** (ch1 p.15, p.5): A journalist — an entirely unauthorised party — was added to the group chat and read classified operational details including airstrike timing. The consequences of this unauthorised disclosure cannot be undone (ch1 p.5).

**Access control / authorisation — breached (root cause)** (ch1 p.30): Signal performs no clearance-based access control. Any group admin can add any phone contact. The system did not prevent an uncleared civilian from joining a classified discussion — it delegated this responsibility entirely to human judgment, which failed.

**Authentication — structurally insufficient** (ch1 p.22, ch3.2 p.28): Signal authenticates phone numbers and devices, not government identities or clearance levels. There is no way to verify that the person behind a Signal account holds a specific security clearance or government role. Proper entity authentication requires verifying the entity's relevant attributes — here, their clearance level and official identity.

**Non-repudiation — structurally absent** (ch1 p.40, ch3.1 p.6): Signal supports disappearing messages, which permanently destroys the evidence trail. There are no digital signatures, no trusted timestamps, and no append-only audit log. Nobody can be held cryptographically accountable for any specific message.

**Data integrity** (ch1 p.34): Not breached in this specific incident, but the architecture provides no protection — Signal's encryption protects against outsiders but does not digitally sign individual messages to their sender.

**Availability** (ch1 p.42): Not breached in this incident.

### Part 3 — Improvements

#### Part 3.1 — PKI with X.509 Certificates for Proper Authentication (ch3.2 p.28–29, ch3.1 p.19)

Each official should hold a government-issued **X.509 certificate** (from a government CA hierarchy) that cryptographically binds their identity, role, and clearance level to their public key. Authentication becomes cryptographic, not based on a phone number. A participant's certificate is verified against the CA before admission to any channel.

**Why X.509 over Signal's device-based authentication?** Signal maps a phone number to a device key — it says nothing about the holder's identity, role, or clearance. An X.509 certificate carries attributes (name, clearance level, role) certified by an authoritative CA (ch3.2 p.28). The system, not the human admin, verifies these attributes.

#### Part 3.2 — System-Enforced Clearance-Based Access Control (ch1 p.30)

Before any participant is admitted to a classified channel, the system automatically verifies their certificate's clearance attribute against the channel's required clearance level. This is not overridable by a human admin — the system enforces it at the cryptographic layer. This directly addresses the root cause of the incident: a human admin accidentally adding the wrong contact is no longer possible.

#### Part 3.3 — Digital Signatures + TTP Timestamp Server for Non-Repudiation (ch3.1 p.6, ch3.2 p.85–87)

Every message must be digitally signed by the sender using **ECDSA P-256** (ch2.2.3 p.85–87). A trusted third-party timestamp server ("may be offered as a service by a trusted third party", ch3.1 p.6) adds a certified timestamp to each signed message, binding the content to a specific moment in time. This creates an append-only, cryptographically attributable audit trail. **Disappearing messages must be disabled** — they are incompatible with the non-repudiation requirement (ch1 p.40).

#### Part 3.4 — Centralised Authentication via Kerberos (ch3.2 p.16–21)

For a distributed multi-user environment with many participants across different roles, **Kerberos** (ch3.2 p.16–27) provides:
- Centralised mutual authentication — users authenticate to servers and vice versa (ch3.2 p.18)
- Time-limited session tickets (ch3.2 p.21) — automatic expiry limits exposure
- Built-in replay prevention via nonces and timestamps (ch3.1 p.3, p.7)

The KDC (Key Distribution Center) acts as the trusted third party (ch3.1 p.6) and can enforce clearance-level access control at ticket issuance — if a user's certificate does not carry sufficient clearance, no ticket is issued.

#### Part 3.5 — Government-Controlled Hardened Devices, Not Personal Smartphones (ch3.7 p.46)

Personal devices running consumer applications are outside government control, unpatched, and potentially infected with malware. Officials discussing classified operations must use **dedicated, government-managed devices** (ch3.7 p.46 — minimise attack surface). Even perfect channel encryption is defeated if the endpoint is compromised: malware on the device reads plaintext before encryption or after decryption. Government devices can enforce OS hardening, EPP (ch3.7 p.43), and prevent installation of arbitrary applications.

### Part 4 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Participant identity | X.509 certificates from government CA | ch3.2 p.28–29; ch3.1 p.19 | Carries clearance attributes cryptographically certified; phone-number auth has no attribute binding |
| Access control | Clearance-based, certificate-attribute-enforced | ch1 p.30 | System-enforced; no human override; root cause of incident eliminated |
| Non-repudiation | ECDSA P-256 per message + TTP timestamp | ch1 p.40; ch3.1 p.6; ch2.2.3 p.85–87 | Disappearing messages eliminated; every message attributable with certified time |
| Multi-user auth / SSO | Kerberos KDC | ch3.2 p.16–27 | Mutual authentication; time-limited tickets; built-in replay protection |
| Devices | Dedicated government-controlled hardware | ch3.7 p.46 | Endpoint compromise defeated at source; consumer devices outside security policy |

### Sources

- IS_UG_1_Introduction (p.5, p.10, p.15, p.22, p.30, p.34, p.40, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.85–87)
- IS_UG_3_1_Appl_Basics (p.3, p.6–7)
- IS_UG_3_2_Appl_AuthMeth (p.16–27, p.28–29)
- IS_UG_3_7_Appl_System (p.43, p.46)

_Status: Complete_  
_Done by: William_
