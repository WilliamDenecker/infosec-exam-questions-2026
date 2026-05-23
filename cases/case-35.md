# Case 35

Design a secure end-to-end encrypted group messaging system similar to Signal or WhatsApp group chats. Members may join and leave over time.

**Which security services are essential? How do you achieve forward secrecy and post-compromise security? Which cryptographic protocols and primitives would you use? What are the remaining vulnerabilities? What legal aspects should you consider?**

## Answer

### Part 1 — Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Messages must be readable only by current group members — not by the service provider, past members, or eavesdroppers. | The messaging provider reads all group messages. An eavesdropper captures all traffic. A past member reads all new messages after they left. |
| **Authentication / data-origin authentication** | Yes — critical | ch1 p.22 | Each message must be verifiably from the stated sender within the group. | Attacker impersonates a group member and injects false messages ("Alice says: meet me at X") without detection. |
| **Data integrity** | Yes — critical | ch1 p.34 | Messages must arrive unmodified. A changed message may cause real-world harm if acted upon. | Man-in-the-middle changes "don't come" to "come now" — recipient acts on a false instruction. |
| **Forward secrecy** | Yes — critical | ch3.6 p.37 | Compromise of a current key must not allow decryption of past messages. | Attacker archives all ciphertext; later compromises a device; retroactively reads all past group history. |
| **Post-compromise security (future secrecy)** | Yes | ch3.6 p.37 | After a key compromise, future messages must become secure again after re-keying without requiring device replacement. | Attacker who temporarily compromised a key continues reading all future messages indefinitely. |
| **Availability** | Yes | ch1 p.42 | Messages must be deliverable even when some members are temporarily offline. The server must store and forward messages. | Member is offline for an hour; returns and has missed all group messages. |

**Note on non-repudiation**: in private messaging, **deniability** is often preferred over non-repudiation — a member should not be able to prove to a third party that another member said something specific. This is the opposite of non-repudiation (ch1 p.40). Group messaging systems typically aim for message authentication within the group (members trust the source) without external provability (no court-admissible proof). HMAC provides this property — any group member could have produced the MAC, so no individual can be cryptographically proven as the author to an outsider.

### Part 2 — End-to-End Encryption Design Principle

The server must never have access to message plaintext. The server stores and forwards only ciphertext. This is the same principle as case 32 (location sharing) and case 29 (cloud backup).

**Why E2E and not server-side encryption?** A server that decrypts messages to "re-encrypt" them provides no meaningful confidentiality — it sees all plaintext, can be compelled by legal authorities, and is a single point of compromise. E2E encryption means the server can hand over only ciphertext under legal compulsion.

### Part 3 — Key Establishment: ECDHE for Forward Secrecy

**Forward secrecy** requires that session keys are ephemeral — derived from ephemeral Diffie-Hellman exchanges that are discarded after use. If a device's long-term key is later compromised, the ephemeral session keys are gone and past sessions cannot be decrypted.

**ECDHE** (ch2.2.4 p.10, ch3.6 p.37): Elliptic Curve Diffie-Hellman Ephemeral. Each user has a long-term ECDSA P-256 identity keypair. At message time, ephemeral ECDHE key material is exchanged to derive per-session (or per-message-batch) symmetric keys.

**Why ECDHE and not static DH?** Static DH shares the same key across all messages — compromise of that key later compromises all past messages. ECDHE generates fresh ephemeral keypairs per session; the private half is discarded immediately after key derivation. An adversary who later compromises the long-term identity key cannot recompute past ephemeral DH values (ch3.6 p.37 — forward secrecy property).

**Why ECDHE and not RSA key encapsulation?** RSA key encapsulation (encrypting a session key with recipient's RSA public key) does not provide forward secrecy — if the recipient's RSA private key is later compromised, all past session keys (encrypted to that key) can be decrypted. ECDHE is specifically designed for forward secrecy.

### Part 4 — Group Key Management

For group messaging with N members, two approaches exist:

**Option A — Pairwise keys (each member pair has an independent key):** Member A encrypts each message N-1 times, once for each other member. Each recipient decrypts with their pairwise key. Provides perfect forward secrecy per pair.
- Problem: for a group of 50 members, each message requires 49 encryption operations and 49 ciphertexts sent to the server. Computationally and bandwidth-inefficient.

**Option B — Symmetric group key K_group (chosen):** All current members share K_group. Each message is encrypted once with K_group. The server stores and delivers one ciphertext to all members.

```
message_ciphertext = AES-256-GCM encrypt(K_group, { sender_ID, message, timestamp }, nonce)
                                         ↑ fresh 96-bit nonce per message
```

**Why AES-256-GCM?** Messages may be stored on devices for years. Against Grover's quantum algorithm (ch2 PQCrypto p.16), AES-128 → 64-bit effective — insufficient for long-retained messages. AES-256 → 128-bit post-quantum. GCM is AEAD (ch2.2.3 p.70–75) — integrity is built-in; the GCM tag detects any message tampering.

**Why a fresh nonce per message?** AES-GCM nonce reuse is catastrophic — two messages encrypted with the same (K_group, nonce) pair leak the XOR of their plaintexts and allow key recovery. Each message must have a unique 96-bit random nonce.

### Part 5 — Forward Secrecy for the Group Key: Key Ratcheting

A static K_group does not provide forward secrecy — compromise of K_group allows decryption of all past group messages. The solution is **key ratcheting**: K_group evolves with every message or batch of messages.

**Ratchet mechanism using SHA-512** (ch2.2.3 p.24–32):

```
After each message batch:
K_group_new = SHA-512(K_group_current || message_counter)
              ↑ the old key is replaced; it cannot be reversed (SHA-512 is one-way)
```

Once K_group advances to K_group_new, K_group_old is securely deleted from all devices. An attacker who compromises K_group_new cannot compute K_group_old (SHA-512 is preimage-resistant). Past messages encrypted under K_group_old are therefore safe.

**Why SHA-512 and not SHA-256?** The ratchet output is used as a 256-bit AES key. SHA-512 provides 256-bit output directly without truncation, and its 256-bit preimage resistance exceeds SHA-256's 128-bit (ch2.2.3 p.24–32). SHA-256 truncated to 256 bits has 128-bit preimage resistance — weaker.

**Post-compromise security**: after a key compromise, the ratchet continues advancing. Once the compromised key has been ratcheted past, new keys are derived from material the attacker does not have. The group becomes secure again after ratcheting — without requiring anyone to leave and rejoin.

### Part 6 — Message Authentication: ECDSA Per Sender

Each message is **digitally signed by the sender** using their ECDSA P-256 private key (ch2.2.3 p.85–87):

```
signature = ECDSA_sign(sender_private_key, SHA-256(message || group_ID || timestamp))
```

The signature is included in the message payload (encrypted with K_group). Recipients decrypt the message, then verify the signature against the sender's registered public key.

**Why ECDSA and not HMAC for message authentication?**

- **HMAC** (ch2.2.3 p.63–66): all members would share a group MAC key. Any member could forge a message appearing to come from any other member — "Alice" could send a message that looks like it came from "Bob". Within-group sender authentication is broken. HMAC cannot distinguish between members.
- **ECDSA**: each member has a unique private key known only to them. The signature is verifiable by all members using the sender's public key, but only the sender can produce it. Each member's authorship within the group is cryptographically attributable.

**Why not RSA-PSS?** ECDSA P-256 produces 64-byte signatures; RSA-PSS 2048 produces 256-byte signatures. Every group message carries a signature — smaller signatures reduce message size and processing time. ECDSA P-256 provides equivalent 128-bit security (ch2.2.3 p.85–87 vs p.88–93).

**Note — deniability to outsiders**: ECDSA provides authentication within the group. However, unlike HMAC, ECDSA also provides external provability — a signature can be shown to a third party to prove authorship. For messaging systems that prioritise deniability (like Signal), a combination of ECDSA (for intra-group authentication) and an additional HMAC (for deniability layer) could be used. For the slide-based approach, ECDSA is chosen for its clear sender-attribution property within the group.

### Part 7 — Member Join and Leave: Forward and Post-Compromise Security

#### Member Joins

When a new member (E) joins the group:

```
1. Group admin generates new K_group_new via ECDHE with E:
   K_distribution = ECDHE(admin_ephemeral, E_public)
2. K_group_new is encrypted for E: AES-256-GCM encrypt(K_distribution, K_group_new)
3. K_group_new is distributed to all existing members (each receives it encrypted to them)
4. All members switch to K_group_new for new messages
```

**Forward secrecy for new member**: E cannot read messages encrypted under K_group_old (before they joined). They only receive K_group_new onwards. Old messages under K_group_old were never provided to E.

#### Member Leaves

When member (C) leaves the group:

```
1. Group admin generates a new K_group_post via ECDHE with remaining members
2. New K_group_post is distributed to all remaining members (not C)
3. All remaining members switch to K_group_post immediately
```

**Post-compromise security for leave**: all future messages use K_group_post, which C was never given. Even if C retained K_group_old, they cannot derive K_group_post (ECDHE-derived; C's private key was not used). C can no longer read any future messages.

**Why a complete key rotation on leave and not just removing C from the ACL?** If the group continues using K_group_old after C leaves, C (who still knows K_group_old) can decrypt all future messages. A new key must be established from which C is excluded.

### Part 8 — Transport Layer

All client-to-server communication over **TLS 1.3** (ch3.6 p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).

**Why TLS on top of E2E encryption?** TLS protects metadata — who is sending messages to whom, when, how frequently. An eavesdropper on the network cannot see even that a message is being sent, let alone its content. TLS also prevents traffic injection by a network-level attacker.

User authentication to the messaging server: SHA-512 + challenge-response (ch3.1 p.7, ch3.2 p.11), MFA with TOTP (ch3.7 p.20).

### Part 9 — Legal Aspects

**GDPR**: messages are personal communications — personal data under GDPR Art. 4(1). E2E encryption technically enforces purpose limitation (Art. 5(1)(b)) — the provider cannot use data they cannot read.

**Right to erasure** (GDPR Art. 17): users must be able to delete their message history. Server-side deletion of ciphertext, plus key deletion on the device, achieves effective erasure.

**Law enforcement access**: a government order to hand over message content cannot be fulfilled when E2E encryption is in place — the provider has only ciphertext. They can comply with court orders to preserve ciphertext; they cannot provide plaintext. This creates legal tension in jurisdictions that require messaging providers to enable lawful interception. The design prioritises user privacy; this is a known trade-off.

**Data retention**: the server retains messages only until delivered. Long-term server-side retention of ciphertext creates metadata exposure and legal risk. Short retention windows (e.g., delete after delivery or after 30 days if undelivered) minimise this.

**Consent for group membership**: adding a user to a group shares some information about them (their presence in the group, their public key) with other members. Group membership should require explicit acceptance to satisfy GDPR consent requirements.

### Part 10 — Remaining Vulnerabilities

- **Endpoint compromise**: if a member's device is compromised, all messages visible on that device are exposed — regardless of how strong the cryptography is. E2E encryption protects messages in transit and on the server; it cannot protect messages that are already decrypted and displayed on a compromised device. EPP on member devices (ch3.7 p.43) is the mitigation.
- **Metadata exposure**: the server knows who is in which group, how frequently messages are exchanged, and the sizes of messages. Even with E2E content encryption, metadata reveals significant information (active relationships, communication patterns). TLS conceals metadata from network observers but not from the server itself.
- **Key distribution server trust**: members must trust that the server distributes the correct public keys for each member. If the server substitutes an attacker's public key for a member's key, the attacker can participate in ECDHE key exchanges. Key verification out-of-band (e.g., comparing key fingerprints via another channel) defeats this attack.
- **Group admin power**: the group admin controls membership and key distribution. A compromised admin can add an attacker as a group member, giving them K_group. Group admin capabilities should be minimal and auditable.

### Part 11 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Message encryption | AES-256-GCM with K_group, fresh nonce per message | ch2.2.3 p.70–75; ch2 PQCrypto p.16 | AEAD; 128-bit post-quantum security; single encryption for entire group (efficient vs pairwise) |
| Forward secrecy | ECDHE for key establishment; SHA-512 key ratchet | ch2.2.4 p.10; ch3.6 p.37; ch2.2.3 p.24–32 | ECDHE ephemeral keys cannot be recomputed after discarded; ratchet prevents retrospective decryption |
| Message authentication | ECDSA P-256 per message per sender | ch2.2.3 p.85–87 | Asymmetric: each member has unique private key; HMAC cannot distinguish senders in a group |
| Key ratchet primitive | SHA-512 one-way chain | ch2.2.3 p.24–32 | Preimage resistance: old keys unrecoverable from new key; SHA-512 provides 256-bit preimage resistance |
| Member leave | Full group key rotation via ECDHE among remaining members | ch2.2.4 p.10 | ACL-only approach fails: departed member still knows K_group; only new ECDHE-derived key excludes them |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Metadata protection; prevents network-level injection; independent of E2E layer |
| Legal | GDPR compliance; E2E encryption enforces purpose limitation | GDPR Art. 5, 6, 17 | Provider cannot misuse data they cannot read; right to erasure satisfied by key deletion |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.40, p.42)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.63–66, p.70–75, p.85–87, p.88–93)
- IS_UG_2_2_4_SecM_KeyExch (p.10)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16–17)
- IS_UG_3_1_Appl_Basics (p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.28–29)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.77, p.85)

_Status: Complete_  
_Done by: William_

---

### Clarification — Post-Compromise Security and the SHA-512 Ratchet

**Question raised**: if an attacker compromises K_group_current, can they keep computing K_group_new, K_group_new+1, ... and read all future messages indefinitely?

**Answer: yes — with a pure SHA-512 ratchet, they can.**

SHA-512 is a public algorithm. The ratchet formula:

```
K_group_new = SHA-512(K_group_current || counter)
```

is deterministic and known to everyone. An attacker who captures K_group_current can compute every future key in the chain indefinitely. The SHA-512 ratchet alone therefore provides **forward secrecy** (cannot go backwards — SHA-512 is preimage-resistant) but **not post-compromise security** (can go forwards — SHA-512 is not secret).

**What is actually required for post-compromise security?**

True post-compromise security requires periodically injecting **fresh secret material** that the attacker does not have. This is what Signal's Double Ratchet protocol achieves by combining two ratchets:

1. **Symmetric ratchet** (SHA-512 chain) — efficient, provides forward secrecy between messages.
2. **DH ratchet** (periodic ECDHE exchange) — each party generates a new ephemeral keypair; the new chain key is derived from both the old chain and the fresh ECDHE output. Since the attacker does not have the new ephemeral private key, they cannot follow the ratchet forward after this point.

```
After a DH ratchet step:
K_group_new = SHA-512(K_group_current || ECDHE(my_new_ephemeral, their_new_ephemeral))
                                         ↑ fresh secret the attacker cannot compute
```

**Implication for this design**: Part 5 of this answer oversimplifies the post-compromise security claim. The SHA-512 ratchet alone heals against going *backwards* (forward secrecy) but not against going *forwards* after a compromise. To fully satisfy post-compromise security, periodic ECDHE ratchet steps must be incorporated — triggered, for example, on each member's first message after receiving a new message from another member.