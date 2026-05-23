# Case 33

A factory deploys thousands of IoT sensors and actuators that communicate with a central control system. Devices have no user interface and limited resources.

**Design an authentication and key-management solution. Which security mechanisms and protocols would you use? How are devices enrolled and revoked? What threats remain?**

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication (mutual)** | Yes — critical | ch1 p.22 | The control system must verify that data comes from a legitimate enrolled sensor, not a rogue device. Sensors and actuators must verify that commands come from the legitimate control system, not an attacker. | A rogue device injects false sensor readings, causing the control system to make wrong decisions. An attacker sends fabricated commands to actuators, causing physical damage to factory equipment. |
| **Data integrity** | Yes — critical | ch1 p.34 | Sensor readings must arrive unmodified. Actuator commands must not be tampered with in transit. | Man-in-the-middle changes temperature reading from 80°C to 30°C → cooling system never activates → equipment overheats. Attacker flips a valve command → chemical process fails. |
| **Access control / authorisation** | Yes | ch1 p.30 | Only the legitimate control system may send commands to actuators. Only enrolled sensors may submit readings. Rogue devices must be excludable. | Any device on the factory network can submit fabricated sensor data or send actuator commands. |
| **Confidentiality** | Yes | ch1 p.15 | Sensor readings may reveal proprietary manufacturing process parameters — trade secrets. Actuator commands reveal process details. | Industrial espionage: competitor intercepts sensor data and reverse-engineers the manufacturing process. |
| **Availability** | Yes — critical | ch1 p.42 | Factory control systems run continuously. Authentication and key management must not introduce latency that disrupts real-time control loops. A DoS must not stop the factory. | Authentication bottleneck causes command delays; actuators miss timing windows; production line stops or produces defects. |

### Part 2 — Design Constraint: Thousands of Constrained Devices, No UI

Key constraints:
- **No user interface**: enrollment and revocation cannot involve a human typing credentials on each device. Must be automated.
- **Limited resources**: constrained microcontrollers. Asymmetric operations (RSA, ECDSA) are expensive. AES is available in hardware.
- **Thousands of devices**: key management must scale. Per-device keys are required (a single shared key means one compromised device compromises all).
- **Industrial timing requirements**: authentication must complete in milliseconds, not seconds.

### Part 3 — Authentication Mechanism: Per-Device X.509 Certificates vs Per-Device Symmetric Keys

Two viable approaches exist. Both are analysed; a hybrid is chosen.

#### Option A — X.509 Certificate per Device (PKI-based)

Each device receives a unique X.509 certificate (ch3.2 p.28–29) at manufacturing time, signed by the factory's internal CA. The control system trusts the factory CA. Mutual TLS (ch3.6 p.7–8) authentication uses these certificates — the device proves identity by presenting its certificate and signing the TLS handshake with its private key (ECDSA P-256, ch2.2.3 p.85–87).

**Advantages**:
- Individually revocable: add compromised device's certificate to CRL (ch3.2 p.50–53)
- Strong forward secrecy via ECDHE (ch3.6 p.37; ch2.2.4 p.10)
- Standard protocol (TLS 1.3 with mutual auth)

**Disadvantages**:
- ECDSA signature computation on very constrained hardware may be too slow for high-frequency sensor reporting. A sensor reporting 100 readings/second cannot compute an ECDSA signature per reading.
- Certificate + ECDSA requires more flash storage than a symmetric key.

#### Option B — Per-Device Pre-Shared Symmetric Key (derived from master key)

```
K_device = AES(K_master, device_ID)
```

Same approach as case 28 (transit card) and case 27 (smart plug). K_master held in tamper-resistant hardware in the control system server. Any device's key derivable on-demand without a database lookup.

Each message is authenticated with AES-128-GCM (ch2.2.3 p.70–75):

```
message_packet = AES-128-GCM encrypt(K_device, { device_ID, reading, sequence_counter, timestamp })
```

**Advantages**: fast (AES hardware); no asymmetric computation per message; scales to thousands of devices.

**Disadvantages**: to revoke a device, the control system must add it to a revocation list and reject its messages — but K_master cannot change (would invalidate all devices). Revocation is application-level, not cryptographic.

#### Chosen: Hybrid Approach

- **Enrollment and session key establishment**: X.509 certificate + TLS 1.3 mutual authentication (one-time or per-session setup). ECDHE establishes a session key with forward secrecy. This gives strong initial authentication and individual revocability.
- **Per-message data authentication**: AES-128-GCM with the TLS session key for all sensor readings and actuator commands within the session. Fast symmetric operations after the initial handshake.

This mirrors how TLS itself works: asymmetric for authentication and key exchange; symmetric for bulk data.

### Part 4 — Device Enrollment

**Manufacturing-time provisioning (chosen)**:

At manufacturing, each device is provisioned with:
1. A unique **device_ID** (serial number)
2. A unique **ECDSA P-256 private key** and corresponding certificate, signed by the factory internal CA
3. The **factory CA's public certificate** (to verify the control system's certificate)

This provisioning happens in a controlled manufacturing environment — physically secure, offline from the internet. No provisioning data travels over the network during enrollment.

**Why manufacturing-time provisioning and not network-based enrollment?**

**Option A — Network enrollment (e.g., device sends certificate signing request over the network)**: a new device connects to the factory network and requests a certificate. An attacker who connects a rogue device to the network before enrollment can also request a certificate — the control system has no way to distinguish a genuine factory device from an attacker's device at this stage. The enrollment channel becomes an attack vector.

**Option B — Manufacturing-time provisioning (chosen)**: the device key and certificate are loaded in a physically secure factory environment. An attacker cannot access this environment to inject rogue devices. The device arrives at the factory already authenticated — its identity is established before it ever touches the factory network.

**Device registration in control system**: the factory ships a manifest of all device IDs and their certificate fingerprints to the factory operator. The control system imports this manifest and trusts only listed device certificates.

### Part 5 — Session Establishment Protocol

```
Step 1 — Device initiates TLS 1.3 connection to control system
         (TLS 1.3, ch3.6 p.7–8, with mutual authentication)

Step 2 — TLS handshake: mutual authentication
         Device:          presents certificate; signs handshake with ECDSA P-256 private key
         Control system:  presents certificate; signs handshake with its ECDSA P-256 private key
         Both:            verify certificate against factory CA
         ECDHE (ch2.2.4 p.10) establishes session key → forward secrecy (ch3.6 p.37)

Step 3 — Session established: AES-256-GCM session key for all data exchange

Step 4 — Sensor readings sent as:
         { device_ID, reading, sequence_counter, timestamp }
         encrypted and authenticated under the TLS session (AES-256-GCM)

Step 5 — Actuator commands received as:
         { command, target_device_ID, sequence_counter, timestamp }
         encrypted and authenticated under TLS session
```

**Why AES-256-GCM for data (not AES-128)?** Unlike case 27 (smart plug), factory IoT devices are typically powered from mains — they are not battery-constrained. The control system server is also unconstrained. AES-256 is appropriate for long-running industrial sessions where session keys may be used for hours before renegotiation.

**Sequence counter** (ch3.1 p.7): each device maintains a sequence counter per session. The control system rejects any message with `sequence_counter ≤ last_accepted`. This prevents replay attacks within a compromised network segment.

**Timestamp** (ch3.1 p.3): each message includes a timestamp. The control system rejects messages older than a configurable threshold (e.g., 5 seconds). This provides a second layer of replay protection and detects delayed-delivery attacks.

### Part 6 — Device Revocation

When a device is compromised, decommissioned, or physically stolen:

```
Control system operator:
  1. Adds device_ID's certificate serial to the Certificate Revocation List (CRL)
     (ch3.2 p.50–53)
  2. Pushes updated CRL to all control system nodes
  3. Terminates any active TLS session from the revoked device
  4. Rejects any new TLS handshake presenting the revoked certificate
```

**Why CRL and not just a device blacklist?** The CRL is a cryptographic mechanism — it is signed by the CA and cannot be forged. A simple IP/device-ID blacklist can be bypassed if the attacker spoofs the device ID. The CRL check is part of the TLS handshake certificate verification — a revoked certificate fails cryptographic validation, not just a lookup.

**How quickly does revocation propagate?** The control system must fetch or receive updated CRLs promptly. For a factory, the CRL is distributed internally — propagation can be near-instantaneous compared to public internet CRL propagation. Online Certificate Status Protocol (OCSP) could be used for real-time revocation checks if the control system has an always-available CA endpoint.

### Part 7 — System Security

**Network segmentation** (ch3.7 p.46): the factory IoT network is on a separate VLAN from corporate IT systems. Compromise of a corporate laptop cannot directly reach sensor/actuator devices. The control system is the only authorised gateway between networks.

**Packet filter** (ch3.7 p.51): devices may only communicate with the control system — not with each other and not with external networks. A compromised device cannot pivot to attack other devices or exfiltrate data to the internet.

**IDS** (ch3.7 p.77, p.85): detect anomalous patterns — a device sending readings outside its physical range, a device sending unusually high volumes of traffic, a device attempting to communicate with unexpected endpoints. Threshold detection triggers alerts (ch3.7 p.85).

**EPP on control system** (ch3.7 p.43): behaviour-based malware detection on the central control server infrastructure.

**Physical security**: factory devices in accessible locations should be tamper-evident. A device with a broken tamper seal should be treated as potentially compromised and revoked immediately.

### Part 8 — Why Not a Single Shared Key for All Devices

This must be explicitly rejected:

If all devices share K_master and all devices know K_master, compromising one device reveals K_master. The attacker can then:
- Forge messages from any device
- Decrypt all communications on the factory network
- Send fabricated actuator commands

The per-device derived key approach `K_device = AES(K_master, device_ID)` means compromising one device reveals only that device's K_device — not K_master. The other devices remain secure. This is the correct approach, equivalent to the transit card system (case 28).

However, in this case, X.509 certificates are used instead — even stronger, because certificate-based revocation is cryptographic, not application-level.

### Part 9 — Remaining Vulnerabilities

- **Physical device compromise**: an attacker with physical access to a device can extract its private key using hardware probing. The attacker then has a valid certificate for that device and can authenticate to the control system. Tamper-evident/tamper-resistant hardware packaging mitigates extraction. Revocation is the response once compromise is detected.
- **Control system compromise**: if the control system itself is compromised, the attacker has K_master (for symmetric approach) or the CA private key (for PKI approach). All devices are then effectively compromised. The control system must be the highest-security component in the deployment — hardened server, EPP, network isolation.
- **Replay within a session**: if sequence counter state is lost (e.g., power failure), the counter resets. An attacker who recorded messages before the reset can replay them with counter values the control system now considers fresh. Sequence counters should be stored in non-volatile memory and not reset on power cycle.
- **Timing-based DoS**: an attacker who floods the factory network with malformed TLS handshakes forces the control system to process and reject thousands of connection attempts — consuming CPU and potentially delaying legitimate connections. Rate limiting and packet filtering at the network perimeter mitigate this.

### Part 10 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Device authentication | Mutual TLS 1.3 with X.509 certificate per device | ch3.6 p.7–8; ch3.2 p.28–29 | Individually revocable; ECDHE provides forward secrecy; single shared key means one compromise = all compromised |
| Certificates | ECDSA P-256 per device | ch2.2.3 p.85–87 | 64-byte compact signature; 128-bit security; faster on constrained hardware than RSA |
| Data encryption | AES-256-GCM within TLS session | ch2.2.3 p.70–75 | AEAD; fast symmetric after initial handshake; 128-bit post-quantum security |
| Replay protection | Sequence counter + timestamp per message | ch3.1 p.3, p.7 | Counter prevents replay within session; timestamp provides absolute freshness bound |
| Enrollment | Manufacturing-time provisioning | ch3.7 p.46 | Network-based enrollment is an attack vector; physical provisioning removes remote enrollment risk |
| Revocation | CRL-based certificate revocation | ch3.2 p.50–53 | Cryptographic mechanism; cannot be bypassed by spoofing device ID; CA-signed |
| Key management | ECDHE session key per TLS session | ch2.2.4 p.10; ch3.6 p.37 | Forward secrecy; session key compromise does not expose past or future sessions |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.63–66, p.70–75, p.85–87)
- IS_UG_2_2_4_SecM_KeyExch (p.10)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.28–29, p.50–53)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.43, p.46–47, p.51, p.77, p.83, p.85)

_Status: Complete_  
_Done by: William_
