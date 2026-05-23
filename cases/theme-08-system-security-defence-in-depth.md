# Theme 8 — System Security: Defence in Depth, Attack Surface, and Hardening

**Appears in**: Cases 27, 33, 34, 37 (and referenced across all cases)

---

## The Core Principle

No single security control is sufficient. A determined attacker will probe every layer of a system. Defence in depth means applying multiple independent security controls so that breaching one layer does not compromise the entire system. If an attacker bypasses authentication, integrity controls catch the tampered message. If an attacker forges a message, IDS detects the anomalous behaviour. If malware installs on a device, firmware signing prevents it from removing cryptographic checks.

The four layers applied consistently across these cases:
1. **Minimise attack surface** — remove everything not strictly necessary
2. **Network segmentation** — isolate different risk zones
3. **Detection** — IDS, logging, anomaly detection
4. **Firmware integrity** — prevent rogue software from disabling protections

---

## Part 1 — Layer 1: Minimise Attack Surface (ch3.7 p.46)

An attack surface is the set of all entry points through which an attacker could attempt to interact with or compromise a system. Every open port, every running service, every exposed API is part of the attack surface.

### Principle of Least Functionality

Every service not required for the device's primary function is disabled or removed in production firmware:
- No SSH daemon on the smart plug or smoke detector
- No web server accessible from external networks
- No debug/JTAG interface in production firmware (disabled at manufacturing)
- No unused open ports (every open port is a potential attack vector)
- No SNMP, telnet, FTP, or legacy protocols

**Why this matters**: a smart plug with an SSH daemon accessible from the network can be attacked via SSH brute-force even if the plug's actual protocol (HMAC-authenticated sensor readings) is perfectly secure. The plug's firmware is simple and does one thing — removing all other services eliminates entire attack classes without any cryptographic effort.

### Outbound-Only Initiation (Cloud-Mediated Architecture)

A key attack surface reduction pattern across cases 27 and 34:

```
Bad: device has a listening server → internet can initiate connections to device
Good: device only initiates outbound connections to cloud → device is invisible from internet
```

The smart plug initiates an outbound TLS connection to the cloud server. The home router's NAT treats this as an established outbound connection and blocks any unsolicited inbound connection to the plug. From the public internet, the plug is invisible — there is no IP address that reaches it directly.

**Attacker's consequence**: to attack the plug directly, the attacker must first compromise either the home router, the home network, or the cloud server. Each of these is a separate layer. Direct internet scanning cannot find or reach the plug.

### Minimal Credential Exposure

- Devices store only the credentials they need
- The smart plug stores K_device but not the user's password
- The car stores K_fob for each fob but not any credential for the user's phone app
- Compromise of the car's memory exposes fob keys but not the user's cloud account password

---

## Part 2 — Layer 2: Network Segmentation (ch3.7 p.51)

Different devices and networks carry different risk levels. Connecting them all to the same flat network means a compromise in any zone can spread to all zones.

### IoT Devices on Separate VLAN

Smart plugs, smoke detectors, and other IoT devices should be on a dedicated IoT VLAN, separate from:
- Corporate IT workstations and servers
- Personal computers and smartphones on the main home/office network
- Guest WiFi networks

A packet filter (ch3.7 p.51) between the IoT VLAN and the main network enforces:
```
IoT VLAN → Internet: allow (devices can reach their cloud servers)
IoT VLAN → Main network: deny (a compromised IoT device cannot attack a PC)
Main network → IoT VLAN: deny (a compromised PC cannot attack an IoT device)
```

**Why this matters**: an IoT botnet infection on one smart plug cannot spread to other devices or PCs on the network if they are in a separate VLAN with a packet filter. The blast radius of a compromised IoT device is limited to the IoT VLAN.

### Factory IoT (Case 33): OT/IT Separation

In factory environments, Operational Technology (OT — machines, PLCs, industrial sensors) must be separate from Information Technology (IT — ERP systems, file servers, email):

```
OT network: factory sensors, actuators, PLCs
IT network: business systems, internet access
Demilitarized Zone (DMZ): data collection / historian server (read from OT, write to IT)
```

A packet filter enforces that OT devices cannot be reached from IT. An attacker who compromises an IT workstation (via phishing, for example) cannot directly send commands to a factory PLC.

**Cases**: 33 (factory IoT); 37 (aircraft avionics)

### Aircraft (Case 37): Avionics vs Passenger WiFi

Aircraft carry two completely separate networks:
- Avionics network: flight controls, navigation, engine management, communications
- Passenger WiFi: entertainment, internet access for passengers

These networks must have no data path between them:
```
Avionics: completely air-gapped from passenger WiFi
           → no shared hardware, no shared IP space, no routing
Passenger WiFi: internet-connected
               → can be compromised by any of thousands of passengers
```

If these networks were connected, a passenger with a laptop could attempt to attack avionics systems. Physical separation at the hardware level (different cables, different switches, no route between them) is the only reliable control.

---

## Part 3 — Layer 3: Detection — IDS, Logging, EPP (ch3.7 p.77, p.85)

Even with strong cryptography and authentication, an attacker with a compromised account can send authenticated malicious commands. Detection identifies abnormal patterns that suggest attack even when individual commands authenticate correctly.

### Intrusion Detection System (IDS) — Threshold-Based (ch3.7 p.77)

IDS monitors the stream of authenticated requests and flags anomalies:

**Alarm silence flooding (case 34)**:
```
Normal: 0–2 silence commands per week
Attack: 50 silence commands in 1 hour
IDS rule: more than 5 silence commands within 60 minutes → alert + require manual review
```

**Geographic impossibility (case 27, 33)**:
```
User logs in from Brussels at 09:00
User logs in from Singapore at 09:05
Impossible to travel 10,000 km in 5 minutes → flag as concurrent session attack
```

**Mass device commands (case 27)**:
```
Normal: user controls 2–3 plugs per day
Attack: 500 plugs controlled in one second → attacker sending batch commands after account compromise
IDS rule: > 10 device commands within 1 second → freeze account, notify user
```

**Repeated authentication failures (all cases)**:
```
Normal: 0–2 failed logins per day
Attack: 1000 failed logins within 10 minutes → brute force attempt
IDS rule: > 10 failed logins in 5 minutes → account lock + CAPTCHA
```

**Command type anomaly (case 37)**:
```
Normal: ground co-pilot issues commands only during scheduled operational hours
Attack: commands at 03:00 local time from an authenticated session
IDS rule: commands outside defined operational window → require re-authentication
```

### Audit Logging (ch3.7 p.83)

Every significant event is logged with:
- Timestamp
- Source identity (user_ID, device_ID, IP)
- Action taken
- Authentication method used
- Success/failure

Logs must be:
- **Tamper-evident**: if an attacker alters logs to hide their activity, the alteration is detectable (sign log batches with ECDSA or append to an append-only log with periodic HMAC chaining)
- **Retained**: GDPR and operational requirements mandate log retention periods
- **Separate from the operational system**: an attacker who compromises the web server must not be able to delete their own log entries

**Case 37 (flight data recorder)**: all commands logged to the black box with their full signed payload. Write-once during flight (physical hardware characteristic). Provides tamper-evident, non-repudiable evidence for incident investigation. The ECDSA signatures on each command mean even the investigator can verify which ground station issued which command — the signature is unforgeable.

### EPP — Endpoint Protection Platform (ch3.7 p.43)

Behaviour-based malware detection on servers and control workstations. Traditional signature-based antivirus misses zero-day malware (malware not in the signature database). Behaviour-based EPP identifies malicious patterns regardless of whether the specific malware is known:

- Unusual memory access patterns (shellcode injection)
- Processes attempting to read credential files
- Unusual network connections from normally-offline processes
- Binary executing from /tmp or unexpected directories

**Cases**: 27 (cloud server running EPP), 33 (control workstation running EPP), 37 (ground station running EPP)

---

## Part 4 — Layer 4: Firmware Integrity

Embedded devices run firmware. If an attacker can push a modified firmware:
- Remove the ECDSA signature verification from incoming commands → accept any unsigned command
- Disable alarm triggers → smoke detector never sounds
- Add a backdoor for remote access
- Exfiltrate K_device to an external server

**Solution**: firmware must be cryptographically signed by the manufacturer. The device only installs firmware that verifies against the manufacturer's embedded public key.

```
Firmware signing:
  Manufacturer: signature = ECDSA_sign(manufacturer_private_key, SHA-256(firmware_binary))
  Distributed: { firmware_binary, signature }

Device before installing:
  Verify: ECDSA_verify(manufacturer_public_key, SHA-256(firmware_binary), signature)
  If FAIL → reject firmware; do not install
```

The manufacturer's public key is stored in **read-only protected memory** (e.g., one-time programmable (OTP) fuses) on the device. It cannot be overwritten by any firmware update — even the manufacturer cannot change the public key after manufacturing. A rogue firmware cannot replace the public key with one it controls.

**Manufacturer key protection** (same principle as Deutsche Post private key in case 36):
- ECDSA private key stored offline (air-gapped)
- Multiple authorised personnel required to sign a firmware release (key ceremony)
- Private key never on internet-connected systems

**Cases**: 27 (smart plug firmware), 33 (factory sensor firmware), 34 (smoke detector firmware), 37 (avionics firmware — most critical)

---

## Part 5 — Cloud-Mediated vs Direct Architecture

**Direct architecture**: the device is directly accessible over the internet. The user's app connects directly to the device.
- Attack surface: device's IP address is publicly known and reachable; any internet-connected system can probe it
- Security burden falls on the constrained embedded device
- Embedded devices are difficult to patch and monitor

**Cloud-mediated architecture (chosen in all cases)**:
- Device initiates outbound only to cloud server
- User's app connects to cloud server
- Device is unreachable from internet (NAT + outbound-only)
- Cloud server handles authentication, authorisation, rate limiting, IDS
- Security burden is on the professional-grade cloud server (easy to harden, patch, monitor)

**Trade-off**: cloud availability dependency. If cloud is down, device cannot be remotely controlled. Accepted in cases 27 and 34 (local function continues). Mitigated in case 37 (dual redundant channels; fail-safe local control).

**Why the cloud architecture is better for security**:
- Cloud server runs on patched, monitored infrastructure — not a 5-year-old embedded firmware
- Cloud server can receive security updates continuously
- Cloud server runs IDS, EPP, rate limiting
- Constrained device is protected behind NAT and does zero inbound listening
- Even if the device firmware has a vulnerability, it cannot be exploited remotely (there is no way to initiate a connection to it)

---

## Part 6 — Encrypted Key and Secret Storage at Rest

Sensitive material stored on servers and devices must be encrypted at rest:

**Server-side key database**:
```
database_row = AES-256-GCM encrypt(K_database, { device_ID, K_device }, nonce)
K_database stored in HSM — not in the database
```

If the database file is stolen (disk theft, backup compromise, SQL injection), K_device values are not immediately exposed. K_database is in a separate, hardware-protected store.

**Device-side key storage**:
- K_device or K_fob stored in dedicated secure memory cells with read-protection
- On microcontrollers with security features: key stored in "lock bits" region where reads are disabled after provisioning
- On higher-end devices: TPM (Trusted Platform Module) or secure enclave stores keys

---

## Part 7 — The Complete Defence Posture (All Layers Together)

```
Attacker wants to: send a fake "silence alarm" command to smoke detector
                   without the user's password, phone, or physical access

Layer 1 — Attack surface: detector not reachable from internet; outbound-only
  → cannot initiate connection to detector; must compromise cloud account

Layer 2 — Authentication: cloud requires password + TOTP for login
  → attacker needs stolen password AND physical possession of user's phone

Layer 3 — Per-action auth: silence command requires fresh TOTP
  → even a stolen session token is insufficient; needs phone again

Layer 4 — IDS: repeated silence commands from impossible location trigger alert
  → even if attacker has session token from a previous location, geographic anomaly detected

Layer 5 — Firmware integrity: rogue firmware cannot be installed without manufacturer key
  → attacker cannot push firmware that removes TOTP requirement

Layer 6 — Audit log: all silence commands logged with authentication proof
  → if attack succeeds, the audit trail identifies exactly when and from where
```

Each layer is independent. Defeating layer 1 (finding the account) still requires defeating layers 2, 3, 4, 5, and 6. The attacker must succeed against all simultaneously.

---

## Slide References

- IS_UG_1_Introduction (p.42: availability as a security service)
- IS_UG_2_2_3_SecM_HashMac (p.85–87: ECDSA for firmware signing)
- IS_UG_3_7_Appl_System (p.43: EPP; p.46: attack surface minimisation; p.51: network segmentation / packet filters; p.77: IDS; p.83: log retention; p.85: threshold-based anomaly detection)
- IS_UG_3_6_Appl_TLS (p.7–8: TLS for all communication channels)
