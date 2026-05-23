# Case 11

Consider a system allowing people to vote in elections using the Web.

**Which security services will be necessary for this purpose? Which security mechanisms and protocol would you use to achieve these security services? What are the possible disadvantages and limitations of this voting method with respect to the traditional voting process?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions. Be sufficiently specific in your implementation (algorithms, key lengths, modes, etc.). However, the system security (firewalls, IDS, etc.) are out-of-scope for this question.*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | Only registered, eligible voters may cast a ballot. The system must verify the voter's identity before issuing a voting token. | Unregistered persons or ineligible voters cast ballots. The same person can vote multiple times under different claimed identities. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | Each voter may vote exactly once. The system must enforce this constraint — a human admin cannot override it. | A voter submits multiple ballots. The system cannot distinguish legitimate from fraudulent submissions. |
| **Data integrity** | Yes — critical | ch1 p.34 | A ballot must arrive at the counting server exactly as cast — nothing added, modified, or deleted in transit. | A man-in-the-middle silently changes a voter's selection after submission. The published result does not reflect actual votes. |
| **Confidentiality** | Yes — critical | ch1 p.15 | The content of each individual ballot must remain secret — vote secrecy is a fundamental requirement of democratic elections. | An attacker or the election authority learns how each individual voted. Voters become vulnerable to coercion or retaliation. |
| **Availability** | Yes | ch1 p.42 | The voting system must remain operational throughout the entire election period. | A DDoS during the last hours disenfranchises all voters who have not yet voted. |

**Non-repudiation is deliberately excluded for individual ballots** (ch1 p.40): if a voter can prove how they voted, they can be coerced or sell their vote. Non-repudiation applies only at the aggregate level — the election authority signs the final result.

### Part 2 — Core Design Challenge: Authentication vs. Anonymity

Authentication and anonymity are in direct conflict. The system must verify the voter is legitimate (authentication) while ensuring no one can link a specific ballot to a specific voter (vote secrecy). The solution separates these concerns across two independent servers that share no data:

```
1. Authentication Server  — knows WHO voted, but never sees the ballot content
2. Vote Server            — receives ballots, but never knows whose ballot it is
```

### Part 3 — Protocol

All communications use **TLS 1.3** (ch3.6 p.5, p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).

#### Phase 1 — Voter Authentication and Token Issuance

```
Step 1 — Voter → Authentication Server:  presents national eID X.509 certificate
Step 2 — Authentication Server → Voter:  challenge nonce (ch3.1 p.7)
Step 3 — Voter → Authentication Server:  ECDSA_sign(private_key, nonce || timestamp)
Step 4 — Authentication Server:          verifies signature against X.509 public key;
                                          checks electoral register (eligible + not yet voted)
Step 5 — Authentication Server → Voter:  one-time token T (256-bit random);
                                          marks voter as "token issued" in register
```

The **nonce** (ch3.1 p.7) ensures each authentication exchange is fresh and prevents replay. The **X.509 certificate** (ch3.2 p.28–29) is government-issued and binds identity to a verified public key. The authentication server records that this voter was issued a token but **does not record the token value** — the link between token and voter is destroyed at issuance.

**Why X.509 eID over username/password?** Passwords authenticate a device claim; an X.509 certificate carries attributes (name, national ID number) certified by the government CA (ch3.2 p.28). A stolen password gives identity; a stolen certificate without the smart card PIN is useless.

#### Phase 2 — Anonymous Ballot Submission

```
Step 6 — Voter → Vote Server:   token T (over a new TLS connection, separate from Phase 1)
Step 7 — Vote Server:           verifies T is valid and unspent → marks T as spent
Step 8 — Voter → Vote Server:   ballot content (encrypted with AES-256-GCM, ch2.2.3 p.70–75)
Step 9 — Vote Server:           stores ballot with no voter identity attached — only the ballot
```

The vote server and authentication server **share no database**. The vote server never learns which voter corresponds to which token — the token value is not logged against any voter identity. Vote secrecy (ch1 p.15) is structurally enforced by separation.

#### Phase 3 — Counting

```
Step 10 — Election authority:   decrypts all ballots with AES-256-GCM key (ch2.2.3 p.70–75)
Step 11 — Election authority:   counts votes and signs result with ECDSA P-256
                                 (ch2.2.3 p.85–87) → non-repudiation at aggregate level
Step 12 — Published result:     signed result + signature verifiable by anyone
```

**Why ECDSA P-256 for the result signature?** ECDSA P-256 (ch2.2.3 p.85–87) provides 128-bit security with compact 64-byte signatures, efficient to verify. The signed published result means the election authority cannot deny having published a specific outcome (ch1 p.40 — non-repudiation at aggregate level).

### Part 4 — Disadvantages and Limitations vs. Traditional Voting

| Limitation | Explanation |
|---|---|
| **Coercion and vote buying** | In a physical polling booth the voter is alone; their choice is private. Online, a third party can stand over the voter, dictate their vote, or demand a screenshot as proof. Vote secrecy cannot be technically enforced on the client side. |
| **Malware on client device** | Malware on the voter's PC can change the ballot content after the voter's selection and before submission (ch3.7 p.43 — endpoint compromise). In traditional voting the physical ballot is in the voter's hand. |
| **Authentication failures** | A stolen eID card or compromised private key allows impersonation. In traditional voting, physical presence plus identity checking provides a baseline defence. |
| **Availability attacks** | A DDoS on the voting servers during the final hours disenfranchises voters who have not yet voted (ch1 p.42). A physical polling station cannot be DDoS'd. |
| **Trust and verifiability** | Traditional voting is observable by party representatives who can watch the count. An online vote requires trusting an opaque software stack. Verifiability for non-technical observers is much harder to provide. |
| **Digital divide** | Voters without internet access or adequate technical skills are excluded from the process. |

### Part 5 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Voter authentication | X.509 eID certificate + ECDSA challenge-response | ch3.2 p.28–29; ch3.1 p.7 | Government-certified identity; smart card PIN required; cannot be used without physical card |
| Anonymity | Two-server separation; token link destroyed at issuance | ch1 p.15 | Structural anonymity — no database links token to voter at the vote server |
| Ballot storage | AES-256-GCM per ballot | ch2.2.3 p.70–75 | AEAD; tamper-evident; cloud provider cannot read ballots |
| Result authentication | ECDSA P-256 signed result | ch2.2.3 p.85–87; ch1 p.40 | Non-repudiation at aggregate level; publicly verifiable |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Forward secrecy; AEAD; application-independent |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.40, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.70–75, p.85–87)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.28–29)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.43)

_Status: Complete_  
_Done by: William_
