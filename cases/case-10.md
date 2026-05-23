# Case 10

Cars contain a lot of software today, not just for the multimedia system (music, navigation, etc.) but also for more critical systems (brakes, engine control, etc.).

Not all car brands allow for software updates, although updates might be desirable to patch bugs and possible security issues. Some car brands allow software updates at a car dealership or a certified maintenance centre. Other car brands allow wireless updates that do not require returning the car to the dealership.

**Suggest an appropriate security solution (don't forget system security) for these software updates for cars (both at a dealership and wireless updates). Which security functions would be essential? Which security protocols, cryptographic algorithms, etc. would you use?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions. Be sufficiently specific in your implementation (algorithms, key lengths, modes, etc.).*

## Answer

### Part 1 — Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Data integrity** | Yes — critical | ch1 p.34 | A tampered software update for brake control or engine management could cause physical harm or death. The installed software must exactly match what the manufacturer produced — no byte may differ. | A compromised update disables ABS, corrupts engine timing, or adds a remote access backdoor. Occupants are at physical risk. |
| **Authentication / data-origin authentication** | Yes — critical | ch1 p.22 | The vehicle must verify the update genuinely originated from the manufacturer, not a rogue dealership tool, malicious OTA server, or man-in-the-middle. | A counterfeit update source delivers malware. The car's ECU has no way to distinguish it from a legitimate update. |
| **Non-repudiation** | Yes | ch1 p.40 | The manufacturer must not be able to deny having released a specific firmware version. The digital signature is evidence binding the manufacturer to the release. | In a product liability case, the manufacturer denies having shipped a software version that caused a crash. No cryptographic proof otherwise. |
| **Confidentiality** | Yes | ch1 p.15 | The update channel must not reveal the car's current software version to an eavesdropper (this information reveals which known vulnerabilities are unpatched). TLS provides this at no extra cost. | An attacker learns the car is running a vulnerable version and mounts a targeted exploit before the patch is applied. |
| **Availability** | Yes | ch1 p.42 | A failed or interrupted update must not leave the car inoperable or in an inconsistent state. Rollback to the previous verified firmware must be possible. | An interrupted OTA update bricks the engine management ECU. The car is stranded with no recovery path. |

### Part 2 — Core Mechanism: Manufacturer-Signed Update Packages

Every update package released by the manufacturer is digitally signed before distribution. The signing process:

1. Compute **SHA-256** hash over the complete firmware binary (ch2.2.3 p.24–32). SHA-256 produces a 256-bit digest; any modification to the firmware changes the hash entirely.
2. Sign the hash with the manufacturer's private key using **ECDSA P-256** (ch2.2.3 p.85–87). ECDSA P-256 provides 128-bit security with compact 64-byte signatures efficiently verifiable on ECU hardware.

**Why ECDSA P-256 over RSA-PSS?** RSA-PSS (ch2.2.3 p.88–93) achieves equivalent security with a 2048-bit key but produces 256-byte signatures — 4× larger. ECDSA P-256 produces 64-byte signatures. On an ECU with limited flash storage and processing capacity, the smaller signature and faster verification matter. RSA-PSS remains acceptable for environments with RSA infrastructure already in place.

**Why SHA-256 over SHA-1?** SHA-1 has known collision vulnerabilities (two different files can produce the same hash). SHA-256 (ch2.2.3 p.24–32) has no known practical collision attack.

The manufacturer's **public key** is embedded in every vehicle's ECU at time of manufacture, stored in write-protected firmware memory. Every update is verified against this embedded public key regardless of delivery path (dealership or OTA).

**Rollback protection**: the update manifest includes a monotonically increasing **version number**. The ECU stores the highest installed version number. Any package with version number ≤ the stored value is rejected — even if its signature is cryptographically valid. This prevents an attacker from delivering a legitimate but outdated vulnerable version.

### Part 3 — Scenario A: Dealership / Certified Maintenance Centre

**Mutual authentication via X.509 certificates** (ch3.2 p.28–29, ch3.1 p.19):

1. The vehicle holds an X.509 certificate issued by the manufacturer's CA, identifying it by chassis number.
2. The dealership tool holds an X.509 certificate issued by the same CA, certifying it as an authorised service point.
3. A **TLS 1.3** session (ch3.6 p.7–8) is established between the dealership tool and the car's diagnostic port using **mutual TLS** — both certificates are verified against the manufacturer's root CA (ch3.1 p.19). Cipher suite: `TLS_AES_256_GCM_SHA384` with ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).
4. The update package (firmware binary + SHA-256 hash + ECDSA signature + version number) is transmitted over the TLS session.
5. The ECU verifies the ECDSA signature. If valid and version number is newer → install. Otherwise → reject.

**Why mutual TLS and not one-way?** One-way TLS verifies the server to the client, but does not verify the client to the server. Mutual TLS ensures that only dealerships holding a valid manufacturer-issued certificate can connect to the car's diagnostic port — an uncertified tool cannot establish the session.

### Part 4 — Scenario B: Wireless Over-the-Air (OTA) Updates

**TLS 1.3** (ch3.6 p.7–8) protects the update channel between the car and the manufacturer's update server:

1. The car connects to the manufacturer's OTA server via TLS 1.3. The server presents its X.509 certificate (ch3.2 p.28); the car verifies it against the manufacturer's root CA public key embedded at manufacture.
2. The car also authenticates to the server with its own X.509 certificate (mutual TLS) — the server identifies which vehicle is requesting the update and serves the correct package.
3. The car downloads the **manifest first**: `{ version_number, SHA-256_hash, ECDSA_signature }`.
4. The ECU verifies the ECDSA signature on the manifest. If valid and version newer → download firmware binary → recompute SHA-256 → compare to manifest hash → install.
5. If the car is in motion or a critical system update is pending, the install is deferred until the car is stationary with the engine off.

**Defence in depth**: TLS protects transport confidentiality and integrity in transit. The ECDSA signature on the package provides an independent verification that survives even a completely compromised TLS session — a man-in-the-middle who breaks the TLS tunnel still cannot deliver a malicious package (the ECDSA signature over the package would fail ECU verification).

### Part 5 — System Security

**Network separation inside the vehicle** (ch3.7 p.46 — minimise attack surface): the infotainment system (which handles OTA downloads) is on a separate network segment from critical ECUs (brakes, engine, steering). A compromise of the infotainment system through the OTA channel cannot directly reach the safety-critical CAN bus. The update agent that programmes critical ECUs sits at the boundary and accepts only signed packages that have passed signature verification.

**Packet filter on the car's network interface** (ch3.7 p.51): the car's cellular/WiFi module permits only outbound HTTPS (port 443) connections to the manufacturer's known update server IP range. No inbound connections accepted from the internet.

**IDS on in-vehicle network** (ch3.7 p.77): monitor CAN bus traffic for anomalous patterns — reprogramming commands outside an update session, unexpected ECU traffic volumes. Log for retroactive analysis (ch3.7 p.83).

**Rollback capability**: before installing, the ECU stores a snapshot of the current firmware. If the new firmware causes instability, the ECU reverts to the previous version. The rollback copy is also signature-verified before restoration.

### Part 6 — Remaining Vulnerabilities

- **Manufacturer signing key compromise**: if the ECDSA private key is stolen, an attacker can sign arbitrary malicious firmware accepted by all vehicles in the field. The signing key must be held in an offline, air-gapped facility (ch3.1 p.19 offline root CA model). This is the single highest-impact risk in the architecture.
- **Build system compromise**: malicious code inserted before signing — the manufacturer signs the tampered binary in good faith. The digital signature is valid. The ECU installs the malware. No update-system cryptography can detect this; it requires build system hardening.
- **ECU hardware replacement at dealership**: a rogue technician replaces an authenticated ECU with a modified one that accepts unsigned updates. The mutual TLS (ch3.2 p.28) verifies the dealership's identity but cannot detect hardware tampering inside the car.
- **Traffic analysis on OTA channel**: an eavesdropper correlating update timing with public vulnerability disclosures can identify which vehicles have not yet patched a known flaw. TLS confidentiality (ch3.6 p.5) limits but does not eliminate this inference.

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Package hash | SHA-256 | ch2.2.3 p.24–32 | No known collision attack; computationally efficient on ECU |
| Package signature | ECDSA P-256 | ch2.2.3 p.85–87 | 64-byte compact signature; 128-bit security; faster than RSA on embedded hardware |
| Rollback protection | Monotonic version number, ECU-enforced | ch1 p.34 | Valid old signatures rejected; cryptographic validity alone insufficient |
| Dealership auth | Mutual TLS 1.3 with X.509 | ch3.6 p.7–8; ch3.2 p.28–29 | Both parties authenticated; uncertified tools cannot connect |
| OTA transport | TLS 1.3, ECDHE, forward secrecy | ch3.6 p.7–8, p.18, p.37 | Even future TLS key compromise does not expose past update sessions |
| Network isolation | Infotainment separate from critical ECUs | ch3.7 p.46 | OTA compromise cannot reach safety-critical systems |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.34, p.40, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.85–87, p.88–93)
- IS_UG_3_1_Appl_Basics (p.16–22)
- IS_UG_3_2_Appl_AuthMeth (p.28–29)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.46–47, p.51, p.77, p.83)

_Status: Complete_  
_Done by: William_
