# Case 13

You have a mail server within a corporate network. You want employees also to be able to read their emails and send emails when they are at home or on the road.

**Suggest an appropriate security solution (don't forget system security). Which security protocols, cryptographic algorithms, etc. would you use?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions. Be sufficiently specific in your implementation (algorithms, key lengths, modes, etc.).*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Email content and credentials transmitted over the public internet must be unreadable to any eavesdropper. Home and mobile connections traverse untrusted networks. | An ISP employee or attacker intercepts all email traffic and login credentials from the employee's home connection. |
| **Authentication** | Yes — critical | ch1 p.22 | The VPN gateway and mail server must verify that the connecting employee is legitimate, not an attacker or impersonator. The employee must also verify the server's identity. | An attacker impersonates an employee and accesses their mailbox. A rogue VPN endpoint intercepts credentials. |
| **Data integrity** | Yes | ch1 p.34 | Emails must not be modified in transit between the employee's device and the mail server. | A man-in-the-middle modifies email instructions silently — the employee sends a transaction and the amount is changed. |
| **Access control / authorisation** | Yes | ch1 p.30 | Each employee accesses only their own mailbox. The mail server enforces this; no employee can read another's email. | An authenticated employee accesses colleagues' mailboxes by exploiting missing authorisation checks. |
| **Availability** | Yes | ch1 p.42 | Remote employees must be able to access their email when needed — business continuity depends on it. | VPN or mail server downtime prevents remote employees from receiving time-sensitive communications. |

### Part 2 — Design Choice: IPSec VPN Tunnel

The remote device connects to the corporate network via an **IPSec VPN tunnel** (ch3.4). Inside the tunnel, the employee accesses the mail server using the same protocols as in the office. This approach:
- Keeps all mail traffic within the corporate network perimeter
- Requires no changes to the internal mail server configuration
- Provides network-level protection for all corporate traffic, not just email

**Why IPSec VPN over direct TLS access?** Direct IMAPS/SMTPS from the internet exposes the mail server's ports to the public internet. An IPSec VPN concentrates the attack surface at a single hardened VPN gateway; the mail server is entirely invisible from the internet (ch3.7 p.46 — minimise attack surface). TLS still runs inside the tunnel as a second layer (defence in depth).

### Part 3 — IPSec Configuration

**Tunnel mode** (ch3.4 p.26–28) between the employee's device and the corporate VPN gateway. In tunnel mode the entire original IP packet (source: employee home IP, destination: mail server) is encrypted and encapsulated in a new IP packet (source: home IP, destination: VPN gateway). This hides the internal network topology — an eavesdropper on the internet cannot see which internal server the employee is communicating with.

**ESP** (Encapsulating Security Payload, ch3.4 p.11) provides both confidentiality and integrity:
- Encryption: **AES-256-GCM** (ch3.4 p.24) — AEAD mode, single pass for encryption and authentication
- GCM's built-in integrity eliminates the need for a separate AH header (ch3.4 p.11), which only provides integrity without confidentiality

**Why AES-256 over AES-128?** AES-128 provides only 64-bit effective security against Grover's quantum algorithm (ch2 PQCrypto p.16). AES-256 retains 128-bit effective security. Corporate communications stored long-term should resist future decryption.

**IKEv2** (ch3.4 p.34–41) handles key exchange and tunnel endpoint authentication:
- **Ephemeral Diffie-Hellman** (ch2.2.4 p.10) for session key establishment → forward secrecy: compromise of the long-term key does not expose past sessions
- Authentication: both VPN gateway and employee device authenticate using **X.509 certificates** (ch3.2 p.28–29) issued by the company CA

**Why X.509 certificates over pre-shared keys for IKEv2?** Pre-shared keys are a shared secret — if compromised, all users are affected. X.509 certificates can be individually revoked (ch3.2 p.50–53) when an employee leaves; a single compromised credential does not affect others.

### Part 4 — Mail Access Inside the Tunnel

Once the VPN tunnel is established, the employee's mail client connects to the internal mail server using:
- **IMAPS** (IMAP over TLS, port 993) for reading email
- **SMTPS** (SMTP over TLS, port 465/587) for sending email

Both use **TLS 1.3** (ch3.6 p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).

**Why TLS on top of IPSec?** Defence in depth (ch3.7 p.46): if the VPN session is somehow compromised, TLS still protects mail credentials and content independently. The two layers are cryptographically independent.

Mail credentials are stored on the server as `SHA-512(salt || password)` (ch3.2 p.11). Login uses **challenge-response** (ch3.1 p.7): the server sends a nonce, the client responds with `HMAC(hash, nonce)`. The raw password never travels the network and a stolen stored hash cannot be replayed (pass-the-hash defeated).

### Part 5 — System Security

**Packet filter** (ch3.7 p.51) on the corporate network perimeter:
- Inbound from internet: permit only UDP 500 and UDP 4500 (IKEv2/IPSec) to the VPN gateway. All other inbound traffic: drop.
- The mail server is not directly reachable from the internet — only from inside the corporate network or through the authenticated VPN tunnel.
- VPN clients (inside authenticated tunnel): permit IMAPS (993) and SMTPS (587) to mail server only.

**Application-level gateway (proxy)** (ch3.7 p.62–65): deployed inside the network in front of the mail server. Inspects IMAP/SMTP requests for anomalous commands before forwarding. Unlike a packet filter, the proxy understands protocol semantics.

**IDS** (ch3.7 p.77, p.85): monitor VPN authentication attempts for brute-force patterns (threshold detection, ch3.7 p.85). Alert on repeated failed IKEv2 certificate authentication. Log all VPN connections with timestamps, employee identity, and source IP for retroactive analysis (ch3.7 p.83, p.96).

**EPP on endpoint devices** (ch3.7 p.43): the employee's remote device is outside the corporate perimeter. Malware on a home PC can capture credentials or intercept decrypted mail after the VPN and TLS layers have processed it. EPP is the primary mitigation. Company-managed devices with enforceable EPP policy are strongly preferred over personal devices for VPN access.

### Part 6 — Remaining Vulnerabilities

- **Endpoint compromise**: malware on the employee's home device captures decrypted emails and credentials after all cryptographic protections have been applied. This is the primary residual risk for remote access. EPP (ch3.7 p.43) reduces but does not eliminate it.
- **Certificate theft**: if the employee's device certificate (used for IKEv2 authentication) is stolen, an attacker can establish a VPN tunnel. Revocation via CRL (ch3.2 p.50–53) limits the exposure window; compromised devices must be reported immediately.
- **Split tunnelling**: if the VPN routes only corporate traffic through the tunnel (not all internet traffic), malware can exfiltrate data via the direct internet path while the VPN is active. The VPN must force **all traffic** through the tunnel.
- **MFA absence**: IKEv2 certificate authentication is strong but if the device certificate is compromised without the employee knowing, the attacker has full access. Adding a TOTP second factor to VPN authentication (ch3.7 p.20, p.46) provides an additional barrier.

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Remote access | IPSec tunnel mode + ESP AES-256-GCM | ch3.4 p.11, p.24, p.26–28 | Mail server hidden from internet; AEAD in one pass; topology hidden |
| Key exchange | IKEv2 with ephemeral DH | ch3.4 p.34–41; ch2.2.4 p.10 | Forward secrecy; fresh session key per tunnel; no long-term key exposure |
| VPN authentication | X.509 certificates from company CA | ch3.2 p.28–29 | Per-employee identity; revocable individually; no shared secret |
| Mail protocol | IMAPS/SMTPS over TLS 1.3 | ch3.6 p.7–8, p.18, p.37 | Defence in depth; AEAD; forward secrecy |
| Password storage | SHA-512 + 96-bit salt + challenge-response | ch3.2 p.11; ch3.1 p.7 | Pass-the-hash defeated; stolen hash cannot be replayed |
| Network perimeter | Packet filter: only UDP 500/4500 inbound | ch3.7 p.51 | Mail server not internet-visible; single hardened entry point |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.42)
- IS_UG_2_2_4_SecM_KeyExch (p.10)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.28–29, p.50–53)
- IS_UG_3_4_Appl_IPSec (p.11, p.24, p.26–28, p.34–41)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.51, p.62–65, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
