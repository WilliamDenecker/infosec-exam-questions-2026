# Question 36

IPsec provides security services through the Authentication Header (AH) and the Encapsulating Security Payload (ESP).

**Compare AH and ESP in terms of:**

- **security services offered,**
- **protection of IP header fields,**
- **deployment practicality.**

**Why is AH often omitted in modern IPsec deployments?**

## Answer

### Authentication Header (AH) (ch3.4 p.11)

**Security services provided by AH** (ch3.4 p.11):
- **Data integrity**: AH computes a MAC (HMAC-SHA256 or similar) over the packet and appends it in the AH header. Any modification to the covered fields is detected.
- **Data-origin authentication**: the MAC is computed using a shared key, proving the packet originates from the authenticated IPsec peer.
- **Anti-replay**: AH includes a sequence number; the receiver maintains a replay window and rejects duplicate sequence numbers.
- **No confidentiality**: AH provides **no encryption**. The payload is transmitted in plaintext. An eavesdropper can read all packet contents; only modification is detected.

**IP header fields protected by AH**:
AH protects all IP header fields that do not change in transit (the "immutable" fields):
- Source and destination IP addresses
- Protocol field
- Other fields that are set at source and not modified by routers

AH explicitly excludes "mutable" fields that are legitimately changed by routers in transit (e.g., TTL/Hop Limit decrements, TOS/DSCP fields, header checksum). Mutable fields are zeroed out before MAC computation and not authenticated.

---

### Encapsulating Security Payload (ESP) (ch3.4 p.11)

**Security services provided by ESP** (ch3.4 p.11):
- **Confidentiality**: ESP encrypts the payload (in tunnel mode: the entire original IP packet; in transport mode: the transport layer payload). The encryption algorithm (e.g., AES-256-GCM) protects content from eavesdroppers.
- **Data integrity**: ESP includes an authentication tag (computed over the ESP header, the encrypted payload, and the ESP trailer). Any modification of the ciphertext is detected.
- **Data-origin authentication**: via the authentication tag using the shared key.
- **Anti-replay**: ESP includes a sequence number with the same sliding-window replay protection as AH.
- **Limited traffic flow confidentiality**: in tunnel mode, ESP hides the original source and destination IP addresses (they are encrypted inside the tunnel; only the outer IP header — VPN gateway endpoints — is visible).

**IP header fields protected by ESP**:
In **transport mode**: ESP protects only the transport layer payload and above. The IP header is NOT authenticated by ESP — only the ESP header onwards is covered. The outer IP header is in plaintext and unauthenticated.

In **tunnel mode** (ch3.4 p.26–28): the entire original IP packet (including original IP header) is encrypted and encapsulated within a new outer IP packet. ESP's authentication covers the ESP header + encrypted inner packet + ESP trailer. The **outer** IP header is NOT authenticated by ESP — same limitation as transport mode.

---

### Comparison: AH vs. ESP

| Feature | AH | ESP |
|---|---|---|
| Confidentiality (encryption) | **No** | **Yes** |
| Data integrity | Yes | Yes |
| Data-origin authentication | Yes | Yes |
| Anti-replay | Yes | Yes |
| IP header authentication | **Yes** (immutable fields) | **No** (outer header unauthenticated) |
| Payload visibility | Plaintext (eavesdropper can read) | Encrypted |
| Traffic flow confidentiality (tunnel) | No | Partial (inner header hidden) |
| Protocol number | 51 | 50 |

**Key difference**: AH authenticates the outer IP header (immutable fields); ESP does not. ESP provides confidentiality; AH does not.

---

### Deployment Practicality

**AH — major deployment problem — NAT incompatibility** (ch3.4 p.11):

Network Address Translation (NAT) is ubiquitous in modern networks (home routers, corporate NAT devices, cloud load balancers). NAT **changes the source IP address** of packets passing through it. But AH authenticates the source IP address field. After NAT modifies the source IP, AH's MAC verification fails — the receiver's MAC over the modified packet will not match the transmitted MAC.

**Consequence**: AH and NAT are fundamentally incompatible. Any NAT device in the path between IPsec peers breaks AH authentication. Since NAT is almost universally present in modern network deployments (especially for remote access VPNs where home users are behind NAT), AH is non-functional in most practical scenarios.

**ESP — works through NAT** (ch3.4 p.11):

ESP does not authenticate the outer IP header. When NAT modifies the source IP address, ESP's authentication covers only the ESP header and payload — which are not modified by NAT. The MAC passes verification.

**NAT-T (NAT Traversal)**: for ESP, IKEv2 includes NAT traversal support — the ESP packet is encapsulated in UDP port 4500, which NAT devices can handle without breaking the ESP authentication. AH has no equivalent NAT-T mechanism — there is no way to make AH work through NAT without fundamentally changing what it authenticates.

---

### Why AH Is Often Omitted in Modern IPsec Deployments (ch3.4 p.11)

**1. NAT incompatibility (primary reason)**: as described above, AH breaks when any NAT device exists in the path. Modern network deployments almost universally involve NAT (home routers, cloud environments, carrier-grade NAT). AH is therefore non-functional in the vast majority of real-world deployment scenarios.

**2. ESP with authentication provides AH's security services minus the IP header coverage**: modern ESP deployments use an AEAD mode (AES-256-GCM, ch3.4 p.24) that provides both confidentiality and integrity in a single pass. The only security service that AH provides and ESP does not is authentication of the **outer IP header**. In practice, this is rarely critical:
- In tunnel mode, the outer IP header contains only the VPN gateway endpoints — not the original source and destination. Spoofing these would at most cause misdirection between VPN gateways, not between final communicating parties.
- The inner IP header (original source/destination) is protected by ESP encryption in tunnel mode.

**3. Using AH + ESP simultaneously is possible but redundant**: some deployments use both AH (for IP header authentication) and ESP (for confidentiality). But this doubles the overhead, and the combined scheme still breaks through NAT due to AH. The added security from authenticating the outer IP header in a VPN scenario is marginal.

**4. IKEv2 + ESP covers the practical requirements**: the IKEv2 handshake authenticates the VPN peers (ch3.4 p.34–41). After authentication, all traffic is protected by ESP (confidentiality + integrity). The outer IP header modification risk (e.g., source IP spoofing by a router) is mitigated by the fact that the IKEv2 security association is established between specific, authenticated endpoints. Adding AH for outer-header integrity provides minimal additional practical security.

**Conclusion**: in modern deployments, IPsec is almost always implemented as **IKEv2 + ESP with AES-256-GCM** (ch3.4 p.24). AH is specified in the standard but rarely deployed due to its NAT incompatibility. The security it provides (outer IP header authentication) is marginal in VPN contexts and does not justify the deployment obstacles.

### Sources

- IS_UG_3_4_Appl_IPSec (p.11: AH and ESP security services, protocol numbers, coverage comparison; p.24: AES-256-GCM for ESP; p.26–28: tunnel vs. transport mode; p.34–41: IKEv2 authentication)

_Status: Complete_  
_Done by: William_
