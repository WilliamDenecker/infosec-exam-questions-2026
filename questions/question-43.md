# Question 43

Problems may arise with some VPN tunnels when a TCP connection is tunneled over another TCP connection when the transmission quality of the link between both VPN gateways is low.

**Which solutions could you use to achieve the desired VPN security (a secure connection between two local networks) without suffering the destructive interference between both TCP connections?**

*Note: Don't forget to take into account the feasibility of your solutions.*

## Answer

### The TCP-over-TCP Problem (ch3.4 p.26–27; ch3.6 p.5)

When a TLS-based VPN tunnel (which uses TCP as transport) carries TCP traffic between the two local networks, the result is **TCP-in-TCP**: an outer TCP connection (the VPN tunnel) encapsulates inner TCP connections (the actual data flows).

**Why this is destructive when the link is lossy**:

TCP's reliability mechanism uses retransmission with exponential backoff and flow control (sliding window). With TCP-in-TCP on a lossy link:

1. A packet is lost on the link between the VPN gateways
2. The **outer TCP** detects the loss and initiates retransmission — this takes time (retransmission timeout, possibly multiple attempts with backoff)
3. Meanwhile, the **inner TCP** also detects the loss (its ACK has not arrived) and starts its own retransmission timer, which also expires
4. The inner TCP sends a retransmission → now there are two copies of the data in flight on the outer TCP
5. The outer TCP may experience congestion collapse: inner TCP congestion control windows shrink, throughput collapses far below the link's actual capacity
6. Both TCP layers fight each other: every loss causes both retransmission timers to expire simultaneously, doubling the congestion penalty

This is known as **"TCP meltdown"** — throughput can drop to near zero on links with even a small packet loss rate (1–2%) because the two retransmission mechanisms reinforce rather than compensate each other.

---

### Solutions (ch3.4 p.26–35; ch3.6 p.5, p.18)

The fundamental fix is to avoid having a TCP-based outer tunnel carry TCP inner traffic.

**Solution 1 — Use IPsec in tunnel mode (ch3.4 p.27, p.34)**

IPsec ESP operates directly over IP (protocol number 50 for ESP), not over TCP. In tunnel mode, IPsec encapsulates the original IP packet in a new IP packet with an ESP header — no outer TCP at all. The VPN gateways exchange ESP packets directly, and retransmission responsibility remains solely with the inner TCP connections end-to-end.

*Feasibility*: Requires IPsec support on both VPN gateways (software or hardware). Widely supported and standard. ESP may be blocked by firewalls → use NAT-T (UDP encapsulation of ESP, port 4500) when needed.

**Solution 2 — Use DTLS (TLS over UDP) as the VPN tunnel (ch3.6 p.18)**

DTLS replaces the outer TCP connection with UDP. Since UDP is connectionless and provides no reliability guarantee itself, there is no outer retransmission mechanism. The inner TCP connections remain the sole managers of reliability. Lost packets in the VPN tunnel are simply dropped at the UDP level — inner TCP retransmits exactly once (no double retransmission).

*Feasibility*: Implemented in OpenVPN (UDP mode), Cisco AnyConnect (DTLS mode). Well-supported in practice. UDP port 443 traverses most firewalls. DTLS adds its own handshake complexity (retransmission timers, cookie exchange) but this is limited to handshake and does not affect data transfer.

**Solution 3 — L2TP/IPsec (ch3.4 p.34)**

L2TP (Layer 2 Tunneling Protocol) uses UDP (port 1701) as transport. Combined with IPsec ESP for confidentiality (L2TP/IPsec), the outer UDP layer eliminates TCP-in-TCP. Inner TCP connections manage their own reliability without outer-layer interference.

*Feasibility*: Natively supported on most operating systems (Windows, macOS, Linux). Widely deployed for remote access. Requires IPsec support and correct configuration of L2TP and IPsec policies.

**Solution 4 — IPsec transport mode with end-to-end support (ch3.4 p.26)**

If all end-user devices on both networks support IPsec, transport mode can be used end-to-end between pairs of communicating hosts. Transport mode adds IPsec headers to existing IP packets without an additional IP header, and again uses no outer TCP.

*Feasibility*: Requires IPsec implementation on every endpoint — not practical if the networks include devices that cannot be configured for IPsec (printers, IoT devices, etc.). More suitable for server-to-server or host-to-host scenarios than general LAN-to-LAN VPN.

---

### Non-Feasible Workaround (not recommended)

**Reducing TCP window sizes and MTU**: reducing the inner TCP window and MTU limits the amount of data in flight and reduces the probability of multiple simultaneous losses. This can reduce (but not eliminate) TCP meltdown and introduces significant throughput degradation. Not a real fix.

---

### Summary

| Solution | Outer transport | Avoids TCP-in-TCP? | Feasibility |
|---|---|---|---|
| IPsec tunnel mode (ESP) | IP (no TCP) | ✓ | High — standard, hardware-accelerated |
| DTLS-based VPN | UDP | ✓ | High — OpenVPN, AnyConnect |
| L2TP/IPsec | UDP (L2TP) + IPsec | ✓ | High — native OS support |
| IPsec transport mode | IP (no TCP) | ✓ | Limited — requires IPsec on all endpoints |
| Reduce TCP window/MTU | TCP (unchanged) | ✗ | Low — workaround only, not a fix |

The most practical solutions for connecting two local networks over a lossy link without TCP meltdown are **IPsec in tunnel mode** (Solution 1) and **DTLS-based VPN** (Solution 2).

### Sources

- IS_UG_3_4_Appl_IPSec (p.26–27: ESP transport and tunnel mode; p.34–35: key management and tunnel configurations; p.6–9: VPN scenarios)
- IS_UG_3_6_Appl_TLS (p.5: TLS over TCP; p.18: DTLS as TLS over UDP)

_Status: Complete_  
_Done by: William_
