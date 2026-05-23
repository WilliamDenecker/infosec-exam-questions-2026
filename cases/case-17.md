# Case 17

A large company operates on two different locations (one in Ghent, the other one in Brussels), each with a local network. The company wants to allow employees securely to access resources from both local networks and also requires both local networks to be secure.

**Suggest a suitable security solution (don't forget system security). Which security protocols, cryptographic algorithms, etc. would you use?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions. Be sufficiently specific in your implementation (algorithms, key lengths, modes, etc.).*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | All inter-site traffic traverses the public internet between Ghent and Brussels. Business-confidential data must be unreadable to any ISP, eavesdropper, or attacker on the path. | An attacker on the internet reads all internal business communications, database queries, and file transfers between the two sites. |
| **Authentication** | Yes — critical | ch1 p.22 | Each VPN gateway must verify the identity of the remote gateway and connecting employees. A rogue gateway could intercept all inter-site traffic. | An attacker sets up a rogue VPN endpoint; both sites connect to it instead of each other. All inter-site traffic is relayed through the attacker. |
| **Data integrity** | Yes | ch1 p.34 | Traffic between sites must arrive unmodified. A man-in-the-middle must not be able to silently alter packets. | An attacker on the path modifies database replication packets or financial transactions between sites. |
| **Access control / authorisation** | Yes | ch1 p.30 | Being on either local network or VPN does not automatically grant access to all resources. Role-based access must be enforced within each site. | Any employee connected to either network accesses all resources on both sites regardless of role. |
| **Availability** | Yes | ch1 p.42 | Employees in Ghent rely on resources in Brussels and vice versa. VPN downtime breaks cross-site workflows. | A VPN gateway failure or IKE failure leaves employees at one site unable to access the other site's resources. |

### Part 2 — Site-to-Site VPN: IPSec Tunnel Mode

**Tunnel mode** (ch3.4 p.26–28) is used between the VPN gateway in Ghent and the VPN gateway in Brussels. In tunnel mode, the entire original IP packet (source: internal Ghent address, destination: internal Brussels address) is encrypted and encapsulated in a new outer IP packet (source: Ghent gateway, destination: Brussels gateway). This:
- Hides the internal IP addressing of both networks from internet eavesdroppers
- Protects all application traffic between the two sites regardless of protocol

**Why tunnel mode over transport mode?** Transport mode (ch3.4 p.27) only encrypts the payload but leaves the original IP headers visible — the source and destination IP addresses (internal hosts) are exposed to the internet. Tunnel mode hides all original addressing inside the encrypted encapsulation.

**ESP** (Encapsulating Security Payload, ch3.4 p.11) with **AES-256-GCM** (ch3.4 p.24): AEAD mode providing confidentiality and integrity in one operation. No separate AH header is needed — GCM's authentication tag covers the inner packet and provides integrity.

**Why AES-256 over AES-128?** Against Grover's quantum algorithm, AES-128 provides only 64-bit effective security — obsolete (ch2 PQCrypto p.16). AES-256 retains 128-bit effective security. Corporate communications may include data that must remain confidential for years.

**IKEv2** (ch3.4 p.34–41) manages Security Associations and key exchange:
- **Ephemeral Diffie-Hellman** (ch2.2.4 p.10) → forward secrecy: compromise of the gateway's long-term certificate key does not expose past traffic
- The two gateways mutually authenticate using **X.509 certificates** (ch3.2 p.28–29) issued by the company's internal CA

**Why X.509 certificates over pre-shared keys for IKEv2?** A pre-shared key is a shared secret — if compromised, both gateways are affected. X.509 certificates can be individually revoked (ch3.2 p.50–53). If one gateway is compromised, its certificate is revoked without affecting the other.

### Part 3 — Individual Employee Remote Access

Employees working from home connect to their nearest site's VPN gateway using a separate IPSec tunnel from their device. Same configuration: IKEv2 + AES-256-GCM + ephemeral DH. Employee devices authenticate with individual X.509 certificates — per-employee revocation is possible when an employee leaves.

Once connected to either site gateway, the employee reaches resources on both networks — Ghent-Brussels traffic flows through the site-to-site tunnel transparently.

### Part 4 — Internal Authentication: Kerberos

Internal services on both networks authenticate users via **Kerberos** (ch3.2 p.16–27) — centralised mutual authentication (ch3.2 p.18) issuing time-limited service tickets. Users log in once; the TGT is valid for the session. Passwords stored as `SHA-512(salt || password)` (ch3.2 p.11). The KDC serves both sites (or each site has a KDC that cross-authenticates via the VPN).

### Part 5 — System Security — Each Local Network

**Packet filter at the internet perimeter** (ch3.7 p.51): each site has a packet filter between its internal network and the internet:
- Inbound from internet: permit only UDP 500 and UDP 4500 (IKEv2/IPSec) to the VPN gateway. All other inbound: drop.
- Inter-site traffic: ESP-encapsulated packets to/from the remote gateway permitted.

**Network zone separation** (ch3.7 p.46 — minimise attack surface): within each local network, segment into zones with packet filters between them:
- DMZ (internet-facing services, if any)
- Internal user network
- Server/data network (not reachable from DMZ)

**Application-level gateway (proxy)** (ch3.7 p.62–65): in front of any internet-facing services. The proxy understands the full application protocol and blocks malicious requests before they reach internal servers.

**IDS** (ch3.7 p.77, p.83, p.85): deployed at each site's perimeter and on internal segments. Threshold detection for anomalous patterns (ch3.7 p.85), lateral movement, and unexpected IKEv2 renegotiation. Logs retained for retroactive forensic analysis (ch3.7 p.96).

**EPP/EDR on all endpoints** (ch3.7 p.38–43): behaviour-based malware detection on all workstations and servers at both locations. Malware on an endpoint inside one site could use the VPN tunnel to attack the other site — endpoint protection limits lateral movement.

### Part 6 — Remaining Vulnerabilities

- **Gateway compromise**: if a VPN gateway is compromised, the attacker can intercept all inter-site traffic passing through it. The gateway must be the most hardened host on each network — dedicated hardware, minimal services, timely patching, EPP installed.
- **Insider threat**: an employee on the internal network can reach resources on both sites via the VPN. Lateral movement is not prevented by the inter-site tunnel. Network segmentation (ch3.7 p.46) and IDS (ch3.7 p.77) reduce this.
- **Certificate lifecycle management**: if a gateway certificate expires or CRL distribution fails, IKEv2 authentication fails and the tunnel drops. Certificate monitoring and timely renewal are operationally critical for availability (ch1 p.42).

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Inter-site tunnel | IPSec tunnel mode + ESP AES-256-GCM | ch3.4 p.11, p.24, p.26–28 | Full packet encryption; hides internal topology; AEAD |
| Key exchange | IKEv2 with ephemeral DH | ch3.4 p.34–41; ch2.2.4 p.10 | Forward secrecy; automatic key renegotiation |
| Gateway authentication | X.509 certificates from company CA | ch3.2 p.28–29 | Individual revocation; no shared secret between sites |
| Internal auth | Kerberos with time-limited tickets | ch3.2 p.16–27 | Mutual authentication; SSO for internal resources |
| Network perimeter | Packet filter: only IKEv2/IPSec inbound | ch3.7 p.51 | All other inbound blocked; single hardened entry point |
| Endpoint protection | Behaviour-based EDR | ch3.7 p.42–43 | Prevents malware from using VPN as lateral movement path |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.42)
- IS_UG_2_2_4_SecM_KeyExch (p.10)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.16–27, p.28–29, p.50–53)
- IS_UG_3_4_Appl_IPSec (p.11, p.24, p.26–28, p.34–41)
- IS_UG_3_7_Appl_System (p.38–43, p.46–47, p.51, p.62–65, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
