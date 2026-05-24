# Question 50

There is a modified version of TLS (DTLS aka Datagram TLS) allowing to offer some kind of TLS over UDP.

**What might be the advantages and drawbacks in using DTLS instead of TLS in a Layer 3 VPN tunnel connecting two different local networks over the Internet.**

*Reminder: for those who might have forgotten what UDP is, this is a very basic transport layer protocol, running on top of IP; it is connectionless, unreliable, but fast. Packets may arrive out of order or not at all.*

## Answer

### Context (ch3.6 p.18; ch3.4 p.26–27)

DTLS (Datagram TLS) is a version of TLS adapted for use over UDP — a connectionless, unreliable, unordered transport protocol. TLS normally requires TCP (reliable, ordered, connection-oriented). DTLS adds mechanisms to handle the unreliability of UDP (retransmission during handshake, explicit sequence numbers, epoch field, fragmentation support, cookie exchange — as discussed in Question 15).

For a **Layer 3 VPN** connecting two local networks, the VPN tunnel carries arbitrary IP packets. The inner traffic may include TCP, UDP, ICMP, and other protocols.

---

### Advantages of DTLS over TLS for a Layer 3 VPN (ch3.6 p.18; ch3.4 p.26–27)

**1. Eliminates the TCP-over-TCP problem (ch3.4 p.26–27)**

With standard TLS (over TCP), the VPN tunnel uses an outer TCP connection. Any inner TCP traffic (file transfers, web browsing, SSH) is encapsulated in TCP inside TCP. On a lossy link between the VPN gateways:
- Both the outer TCP and each inner TCP detect the loss and initiate retransmission
- Both TCP layers reduce their congestion windows simultaneously → throughput collapses ("TCP meltdown")

With DTLS (over UDP), the outer tunnel provides **no reliability guarantee**. A lost UDP packet is simply dropped. The inner TCP connections are the sole managers of reliability, exactly as designed. No feedback loop, no TCP meltdown. Throughput is proportional to actual link capacity.

**2. Lower latency for real-time and UDP-based applications**

DTLS over UDP does not wait for TCP acknowledgments before delivering data to the inner network stack. Packets that arrive out of order are processed independently (or discarded if old). This is essential for:
- VoIP and video conferencing: even one TCP retransmission causes a large delay spike (the entire stream waits for the retransmitted packet); with UDP/DTLS, late packets are discarded by the application without stalling the stream
- DNS queries, NTP, gaming: short UDP transactions complete in one round trip without connection setup or retransmission overhead

**3. No inner-outer TCP interaction on packet loss**

UDP is connectionless: no per-packet acknowledgement from gateway to gateway. Each IP packet is processed independently. If the VPN gateway crashes and restarts, the DTLS tunnel can be re-established without waiting for TCP connection timeouts (typically 2–4 minutes for TCP's TIME_WAIT state).

**4. Robustness to network disruptions**

UDP-based tunnels are stateless at the outer layer. A brief link outage (seconds) causes packet loss, which DTLS handles gracefully — no TCP connection teardown and re-establishment needed. The inner TCP connections survive brief outages as long as the outage is shorter than their retransmission timeout.

---

### Drawbacks of DTLS over TLS for a Layer 3 VPN (ch3.6 p.18)

**1. Out-of-order delivery and reordering overhead**

UDP does not guarantee packet order. Packets traversing different routes may arrive out of order. DTLS has explicit sequence numbers to handle this (required for replay protection), but:
- The VPN must decide whether to buffer out-of-order packets (adding latency and memory) or discard them (causing inner TCP to retransmit)
- For inner TCP traffic, out-of-order delivery at the VPN layer forces inner TCP to buffer and wait for gaps, reducing throughput

**2. No delivery guarantee for UDP inner traffic**

Lost UDP/DTLS packets carry lost inner UDP packets (DNS, VoIP RTP) that are permanently dropped — no recovery at the tunnel level. For reliable inner protocols (TCP), this is fine (inner TCP retransmits). For inner UDP traffic (VoIP, DNS), dropped packets are unrecoverable, which may degrade quality.

**3. DTLS handshake complexity (ch3.6 p.18)**

DTLS adds significant complexity to the handshake compared to TLS:
- Retransmission timers must be implemented (packets may be lost during handshake)
- Handshake fragmentation: DTLS handshake messages may need to be fragmented to fit within the MTU
- Cookie exchange (HelloVerifyRequest) adds one round trip before DH computation to prevent DoS
- Message_seq fields must be tracked to detect and discard duplicate handshake messages

**4. MTU and fragmentation issues**

DTLS records must fit within the network MTU (typically 1500 bytes for Ethernet). The DTLS header + AES-GCM overhead consumes ~50–100 bytes, reducing the effective payload. If inner IP packets are large, they must be fragmented by the inner IP stack or the VPN must perform Path MTU Discovery and signal inner hosts to reduce their MTU (PMTUD). Misconfigured MTU causes silent packet loss for large frames.

**5. Some firewalls block UDP**

UDP on non-standard ports may be blocked by restrictive firewalls. While DTLS on UDP port 443 is increasingly accepted (QUIC has normalised this), some enterprise firewalls still block all UDP. TLS over TCP 443 is universally allowed.

---

### Comparison Table

| Criterion | TLS (over TCP) | DTLS (over UDP) |
|---|---|---|
| TCP-over-TCP problem | Yes — TCP meltdown on lossy links | No — no outer retransmission |
| Real-time application latency | High — TCP retransmission delays | Low — no outer buffering |
| Packet ordering | Guaranteed by outer TCP | Not guaranteed — must handle reordering |
| Handshake complexity | Standard TLS | Higher — retransmission, cookie, fragmentation |
| MTU/fragmentation | Handled by TCP (segmentation) | Must be managed manually |
| Firewall compatibility | Excellent (TCP 443) | Good (UDP 443) but some firewalls block UDP |
| Robustness to brief outages | TCP reconnection needed | Stateless — resumes automatically |

### Sources

- IS_UG_3_6_Appl_TLS (p.5: TLS over TCP; p.18: DTLS — modifications to TLS for UDP, retransmission timers, message_seq, fragmentation, cookie exchange, explicit sequence numbers, epoch)
- IS_UG_3_4_Appl_IPSec (p.26–27: ESP tunnel mode and VPN; p.6–8: VPN scenarios and advantages)

_Status: Complete_  
_Done by: William_
