# Question 45

TLS is often used as a building block for VPN solutions.

**Compare TLS-based VPNs with IPsec-based VPNs.**

**Discuss differences in:**

- **security layer (network vs transport),**
- **compatibility with NAT and firewalls,**
- **operational complexity.**

**Explain why tunnelling TCP over TCP can be problematic.**

## Answer

### 1. Security Layer: Network vs Transport (ch3.4 p.2–3, p.26–27; ch3.6 p.5)

**IPsec operates at the network layer** (Layer 3). The ESP (Encapsulating Security Payload) header is inserted between the IP header and the transport layer payload. In tunnel mode, the entire original IP packet (including its header) is encrypted and encapsulated in a new IP packet. This means:
- **All IP traffic** between the protected networks is secured, regardless of application or protocol (TCP, UDP, ICMP, etc.)
- The protection is **transparent to applications** — applications do not need to be modified or even aware of IPsec
- A gateway at the perimeter handles IPsec for the entire local network (ch3.4 p.8)
- Limited **traffic flow confidentiality** in tunnel mode: an observer can see only which gateways communicate, not which hosts or applications (ch3.4 p.27)

**TLS operates at the transport/application layer** (between Layer 4 and Layer 7). TLS protects a specific TCP stream between two endpoints. When used as a VPN:
- Only traffic **explicitly routed through the VPN client** is protected
- Other traffic (e.g., direct internet browsing with split tunnelling) bypasses the VPN
- Applications must establish individual TLS sessions, or the VPN client tunnels all traffic inside one TLS TCP connection
- An attacker can see the outer IP headers (source, destination, port 443) — less traffic flow confidentiality than IPsec tunnel mode

**Key difference**: IPsec provides network-level protection for all traffic; TLS-VPN is application-layer and requires explicit routing of traffic through the tunnel.

---

### 2. Compatibility with NAT and Firewalls (ch3.4 p.11; ch3.6 p.18)

**IPsec and NAT (ch3.4 p.11)**:
- AH (Authentication Header) authenticates the outer IP header including the source IP address. NAT modifies the source IP address, destroying the AH integrity check — **AH is incompatible with NAT** (ch3.4 p.11, p.18)
- ESP (protocol 50 in the IP header): standard NAT devices translate TCP/UDP port numbers but cannot translate arbitrary IP protocols. Many NAT devices drop ESP packets
- **NAT-T** (NAT Traversal): encapsulates ESP inside UDP datagrams (port 4500) to traverse NAT — requires detection and negotiation during IKEv2 (ch3.4 p.39)
- Corporate firewalls frequently block UDP 500 (IKE) and UDP 4500 (NAT-T) as non-standard traffic

**TLS-VPN and NAT**:
- TLS runs over standard **TCP port 443** (HTTPS) — NAT translates TCP ports trivially
- All firewalls permit outbound TCP 443; no special rules required
- Deep Packet Inspection may detect non-browser TLS traffic, but standard port 443 is almost never blocked
- **Conclusion**: TLS-VPN traverses NAT and firewalls transparently; IPsec requires special handling

---

### 3. Operational Complexity (ch3.4 p.9, p.34; ch3.6 p.5)

**IPsec operational complexity (ch3.4 p.34)**:
- IKEv2 specification: 142 pages; complex parameter negotiation (DH groups, SA parameters, traffic selectors)
- Must configure: SPD (Security Policy Database), SAD (Security Association Database), IKE parameters, certificate infrastructure for authentication
- Requires kernel-level configuration on endpoints; not feasible on devices that cannot be reconfigured (printers, IoT, legacy systems)
- Multi-site deployments require managing hundreds of Security Associations
- Interoperability testing between different vendors is often required

**TLS-VPN operational complexity (ch3.6 p.5, p.7–8)**:
- Client software: browser-based (zero installation) or lightweight downloadable agent
- Configuration: standard TLS parameters (cipher suites, certificates); familiar to administrators who manage HTTPS
- PKI reuses the same certificates and CA infrastructure as HTTPS deployments
- Gateway is essentially a web server (e.g., nginx with OpenVPN) — no kernel-level policy management
- **Conclusion**: TLS-VPN is significantly simpler to deploy and manage

---

### 4. Why Tunnelling TCP over TCP Is Problematic (ch3.4 p.26–27; ch3.6 p.5)

When a TLS-VPN tunnel uses **TCP** as transport, and the traffic inside the tunnel also uses **TCP** (e.g., web browsing, SSH, file transfer), the result is **TCP-in-TCP** ("TCP meltdown"):

**The mechanism**:
1. The outer TCP connection (TLS tunnel) is responsible for reliable delivery between the two VPN gateways
2. The inner TCP connections are responsible for reliable delivery between the actual endpoints
3. When a packet is lost on the physical link between gateways:
   - The outer TCP's retransmission timer fires → it retransmits (with exponential backoff)
   - **Simultaneously**, the inner TCP's retransmission timer also fires → it also retransmits
   - Both retransmissions travel on the same (congested) link → more congestion → more losses
4. Both TCP layers reduce their congestion windows simultaneously → throughput collapses
5. On a link with even 1–2% packet loss, throughput can drop to near zero because the two congestion control mechanisms amplify each other

**Why IPsec avoids this**: IPsec uses ESP (directly over IP, not TCP) or encapsulates ESP in UDP. Neither outer protocol has a retransmission mechanism. Only the inner TCP connections manage reliability end-to-end.

**Why DTLS (TLS over UDP) avoids this** (ch3.6 p.18): UDP is connectionless and unreliable — it has no retransmission mechanism. Lost packets in the DTLS tunnel are simply dropped; inner TCP retransmits exactly once, as designed.

---

### Summary Comparison Table

| Aspect | IPsec (tunnel mode) | TLS-VPN (TCP) | TLS-VPN (DTLS/UDP) |
|---|---|---|---|
| **Layer** | Network (Layer 3) | Application/transport | Application/transport |
| **Traffic coverage** | All IP traffic | Routed traffic only | Routed traffic only |
| **NAT compatibility** | Poor — requires NAT-T | Excellent — TCP 443 | Good — UDP 443 |
| **Firewall traversal** | Moderate — UDP 500/4500 | Excellent — port 443 | Good — UDP 443 |
| **Operational complexity** | High | Low | Low |
| **TCP-over-TCP problem** | None — ESP is not TCP | Yes — TCP meltdown | None — UDP has no retransmission |
| **Forward secrecy** | IKEv2 with ECDHE | TLS 1.3 mandatory | TLS 1.3 mandatory |
| **Traffic flow confidentiality** | High (tunnel mode encrypts inner IP headers) | Low (outer IP visible) | Low (outer IP visible) |

### Sources

- IS_UG_3_4_Appl_IPSec (p.2–3: network layer security; p.8–9: IPsec advantages and drawbacks; p.11: AH and ESP protocols; p.18: AH and NAT incompatibility; p.26–27: ESP transport and tunnel mode; p.34–35: IKEv2 key management complexity)
- IS_UG_3_6_Appl_TLS (p.5: TLS over TCP and VPN; p.7–8: TLS 1.3 handshake and security properties; p.18: DTLS — TLS over UDP)

_Status: Complete_  
_Done by: William_
