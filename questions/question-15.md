# Question 15

There is a modified version of TLS (DTLS aka Datagram TLS) allowing to offer some kind of TLS over UDP.

**Which elements of the TLS Handshake and of the TLS Record Protocol should be modified to allow operation on top of UDP (instead of TCP)?**

*Reminder: for those who might have forgotten what UDP is, this is a very basic transport layer protocol, running on top of IP; it is connectionless, unreliable, but fast. Packets may arrive out of order or not at all.*

*Note: don't attempt to include all TCP features in TLS!*

## Answer

### Why TLS Over TCP Cannot Directly Run Over UDP

TCP provides three properties that TLS relies on (ch3.6 p.5):
1. **Reliable delivery**: TCP guarantees that all bytes arrive and retransmits lost packets
2. **In-order delivery**: TCP presents data to the application in the correct sequence
3. **Connection-oriented**: TCP maintains a connection state; bytes form a continuous stream

UDP provides none of these: packets may be lost, reordered, or duplicated; there is no connection state and no retransmission. TLS assumes reliable in-order delivery at the transport layer — several of its mechanisms break when this assumption is violated.

The modifications needed for DTLS cover two areas: the **handshake protocol** and the **record protocol**.

---

### Modifications to the TLS Handshake Protocol (ch3.6 p.7–8)

**Problem 1 — Lost handshake messages**

The TLS handshake is a sequential exchange: each message depends on the previous. If a UDP handshake message is lost, the entire handshake stalls. TCP retransmits automatically; UDP does not.

**DTLS modification**: add **handshake retransmission with a timeout**. Each party starts a timer after sending a handshake message. If no response arrives within the timeout, the message is retransmitted. The other party must be idempotent — receiving the same message twice produces the same response. Timer values use exponential back-off to handle congested networks.

**Problem 2 — Reordered handshake messages**

UDP packets may arrive out of order. In TLS, receiving the third handshake message before the second is impossible (TCP orders them). Over UDP, this is possible.

**DTLS modification**: add a **message_seq** field (sequence number) to each handshake message header. Receivers buffer out-of-order messages and process them in the correct sequence order. Messages with unexpected sequence numbers are held in a reorder buffer until missing messages arrive (or are retransmitted).

**Problem 3 — Large handshake messages (certificate fragmentation)**

TLS certificates can be larger than the network's MTU (Maximum Transmission Unit, typically 1500 bytes for Ethernet). In TLS over TCP, this is transparent — TCP handles fragmentation and reassembly. Over UDP, packets larger than the MTU are fragmented at the IP layer (or dropped if the "Don't Fragment" bit is set), and IP fragmentation is unreliable.

**DTLS modification**: add **handshake message fragmentation** within DTLS itself. Each handshake message includes:
- `fragment_offset`: byte offset of this fragment within the full handshake message
- `fragment_length`: length of this fragment
- `length`: total length of the full handshake message

This allows DTLS to split large messages (e.g., a certificate) into MTU-sized fragments and reassemble them at the receiver. Lost fragments can be individually retransmitted.

**Problem 4 — Replay attacks (UDP has no sequence number)**

TCP's connection state implicitly prevents replay: a replayed packet would have an out-of-sequence TCP sequence number and would be rejected. UDP has no such protection — a replayed DTLS handshake packet could be accepted as a new handshake attempt.

**DTLS modification**: add a **cookie mechanism** to prevent Denial-of-Service amplification attacks and replay. When the server receives a ClientHello, it does not immediately allocate state. Instead, it sends a HelloVerifyRequest containing a cookie (a keyed hash of the client's IP address and port, computed with a secret known only to the server). The client must include this cookie in a retransmitted ClientHello. The server verifies the cookie before allocating any handshake state. This ensures the client's claimed IP address is reachable (preventing source-IP spoofing DDoS), and the cookie's MAC prevents forgery.

---

### Modifications to the TLS Record Protocol (ch3.6 p.7–8)

The TLS record protocol handles the encryption and MAC of application data after the handshake.

**Problem 5 — Stateful sequence number (implicit in TLS)**

Standard TLS uses an implicit 64-bit sequence number that increments with each record. This sequence number is included in the MAC computation (and as additional authenticated data in GCM) to prevent record reordering and replay. In TLS, this is implicit — both sides maintain the same counter and it never needs to be transmitted because TCP guarantees in-order delivery.

Over UDP, records may arrive out of order or be lost. The receiver cannot maintain an implicit sequence number because the counter would diverge from the sender's whenever a record is dropped.

**DTLS modification**: include an **explicit sequence number** in every DTLS record header. The receiver uses this sequence number for two purposes:
- **Reorder handling**: accept records out of order (within a configurable window)
- **Replay detection**: maintain a sliding window of recently received sequence numbers. A record with a sequence number already seen in the window is rejected as a replay. A record with a sequence number below the window's lower bound is rejected as too old.

**Problem 6 — Record fragmentation across UDP packets**

TLS records may span multiple TCP segments (TCP provides a byte stream). Over UDP, each record must fit in a single UDP datagram (or be fragmented at the DTLS layer). DTLS restricts record size to fit within the network MTU. Application data that exceeds one record must be split into multiple records, each transmitted as a separate UDP datagram.

**Problem 7 — Epoch field for key changes**

In TLS, a cipher suite change (ChangeCipherSpec) is delivered in order by TCP. Over UDP, records encrypted under the old key might arrive after the ChangeCipherSpec. The receiver needs to know which key to use for each record.

**DTLS modification**: add an **epoch** field to the record header. The epoch increments with each key change (after ChangeCipherSpec / after a DTLS 1.3 KeyUpdate). Records from different epochs use different keys. The receiver can process records from two adjacent epochs simultaneously, handling out-of-order delivery across a key change.

---

### What Does NOT Change

- **Cryptographic algorithms**: the same cipher suites (AES-256-GCM, ECDHE, RSA/ECDSA) are used in DTLS as in TLS. The security properties of the encryption and authentication are identical.
- **Certificate-based authentication**: the server's certificate and CertificateVerify are unchanged in content — only their transport (fragmentation, retransmission) changes.
- **Forward secrecy**: ephemeral ECDHE is used identically in DTLS.
- **Key derivation**: the master secret and session key derivation are identical.

---

### Summary of DTLS Modifications

| TLS Assumption (TCP-based) | Problem Without TCP | DTLS Solution |
|---|---|---|
| Reliable message delivery | Lost handshake messages stall | Retransmission with timer and exponential back-off |
| In-order message delivery | Handshake messages arrive out of order | message_seq field; reorder buffer |
| No fragmentation needed | Large certificates exceed MTU | Handshake-level fragmentation with offset/length fields |
| No replay of connection setup | UDP allows source-IP-spoofed ClientHello floods | Cookie-based stateless HelloVerifyRequest |
| Implicit sequence numbers | Out-of-order records break counter | Explicit sequence number in every DTLS record |
| Single in-order key stream | Records from old/new key intermix across ChangeCipherSpec | Epoch field in record header |

### Sources

- IS_UG_3_6_Appl_TLS (p.5: TLS and TCP dependency; p.7–8: TLS handshake and record protocol structure; p.18: cipher suites)

_Status: Complete_  
_Done by: William_
