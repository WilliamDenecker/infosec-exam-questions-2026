# Case 37

Commercial airliners typically fly with both a pilot and a co-pilot on board. In the future it has been suggested that aeroplanes could be flown with only a single pilot in the cockpit and with a remote co-pilot on the ground, who would be able to take over in case of emergency.

**Suggest an appropriate security solution (don't forget system security) for this approach (pilot on-board + remote co-pilot on the ground). Which security functions would be essential? Which security protocols, cryptographic algorithms, etc. would you use?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions. Be sufficiently specific in your implementation (algorithms, key lengths, modes, etc.).*

*Note: I know that drones are already operated with remote control only. However a self-destruct function when things really go wrong is not an option for a commercial airliner!*

## Answer

### Part 1 — Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication (mutual)** | Yes — critical | ch1 p.22 | The aircraft avionics must verify that control commands come from the legitimate assigned ground co-pilot — not from an attacker, a rogue ground station, or a different aircraft's controller. The ground station must verify it is receiving data from the correct aircraft. | Attacker injects a fabricated "descend to 1000 ft" command; aircraft executes it. |
| **Data integrity** | Yes — critical | ch1 p.34 | Each command (altitude, heading, speed, autopilot) must not be modifiable in transit. Sensor and telemetry data from the aircraft must arrive uncorrupted at the ground station. | Man-in-the-middle changes "climb to FL350" to "descend to FL100" — catastrophic in congested airspace. |
| **Confidentiality** | Yes | ch1 p.15 | Cockpit video/audio, aircraft position, command content, and telemetry reveal flight-critical operational information. An adversary observing this data can plan an attack or gather intelligence for hijacking. | Attacker intercepts video feed and aircraft position; uses this to plan physical interference at destination. |
| **Availability** | Yes — critical | ch1 p.42 | **The most safety-critical service in this case.** If the remote co-pilot cannot reach the aircraft precisely when the onboard pilot is incapacitated, the system fails its primary purpose. The communication link must be highly reliable with automatic failover. | Satellite link fails during onboard pilot incapacitation; remote co-pilot cannot take over; 300 passengers have no pilot. |
| **Non-repudiation** | Yes | ch1 p.40 | Every command issued by the remote co-pilot must be attributable and auditable. Aviation accident investigation requires cryptographic evidence of exactly who issued which command at what time. | After an incident, the airline and co-pilot dispute which commands were issued; no auditable record exists. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | Only the co-pilot assigned to THIS flight, authenticated at THIS ground station, may issue commands to THIS aircraft. No other entity — including other authorised co-pilots, airline operations staff, or ATC — may issue flight control commands. | Ground operations staff issue an unauthorised command during a busy period; aircraft acts on it. |

### Part 2 — Critical Safety Property: Local Onboard Pilot Independence

Before designing the security architecture, one safety property must be stated as absolute:

**The onboard pilot retains full manual authority over the aircraft at all times when conscious and capable.** The remote co-pilot connection is an emergency backup layer — it does not reduce the onboard pilot's authority.

This leads to two operating modes:
1. **Advisory mode** (normal operation): the ground co-pilot monitors all aircraft data, communicates with the onboard pilot, and can advise — but the onboard pilot retains all flight controls.
2. **Override mode** (emergency): the ground co-pilot takes full authority when the onboard pilot is incapacitated. This mode requires an explicit authenticated activation sequence (see Part 7).

**Consequence for security design**: the authentication burden for override mode must be higher than for advisory mode. An accidental or attacker-triggered transition to override mode is potentially more dangerous than no remote access at all. Fail-safe: if the command link fails while in override mode, the aircraft immediately reverts to local-control-only — remote failure must never leave the aircraft in an indeterminate state.

### Part 3 — Why Not Symmetric Key Authentication for Commands

An obvious first approach: give the ground station and aircraft a shared symmetric key K. Commands are authenticated with `HMAC-SHA256(K, command || counter || timestamp)`.

**Why HMAC is rejected for command authentication** (as the sole mechanism):

1. **Non-repudiation is impossible**: the aircraft also holds K. If a disputed command is presented, the aircraft itself could have computed that HMAC. There is no cryptographic proof that the command originated from the ground station and not from a compromised avionics module. Aviation accident investigation requires external verifiability.
2. **Compromise scope**: if K is extracted from the aircraft (e.g., during maintenance, or by a nation-state with physical access), an attacker can forge commands for that aircraft indefinitely. Key rotation requires physical access to the aircraft.
3. **Key distribution at scale**: an airline operating 200 aircraft with 50 ground stations must manage 200 × 50 = 10,000 pairwise keys (or one shared key per aircraft, each meaning compromise of one ground station compromises that aircraft). Certificate-based PKI scales cleanly.

**HMAC is used** as the MAC within the TLS session (which is symmetric after the handshake), but ECDSA is required for per-command non-repudiation. This is not a contradiction — both are used at different layers for different purposes.

### Part 4 — Chosen Solution: Mutual TLS 1.3 + Per-Command ECDSA

#### Session Layer — Mutual TLS 1.3

The ground control station and aircraft avionics establish **TLS 1.3** (ch3.6 p.7–8) with **mutual certificate authentication** over the aviation datalink.

Both endpoints present **X.509 certificates** (ch3.2 p.28–29):

- **Aircraft certificate**: issued by the avionics manufacturer's internal CA, binding `aircraft_ID` to the avionics system's ECDSA P-256 public key. Loaded and signed during avionics installation; updated only during scheduled maintenance at an authorised facility.
- **Ground station certificate**: issued by the airline authority's CA, binding `ground_station_ID` to its ECDSA P-256 public key.

TLS 1.3 cipher suite: `TLS_AES_256_GCM_SHA384` (ch3.6 p.18) with ECDHE (ch2.2.4 p.10) for forward secrecy (ch3.6 p.37).

**Why AES-256 and not AES-128?** Commercial aircraft have operational lifetimes of 25–30 years. Against Grover's quantum algorithm (ch2 PQCrypto p.16), AES-128 → 64-bit effective security. A quantum computer capable of attacking 64-bit keys may feasibly exist within a 30-year horizon. For safety-critical aviation systems with multi-decade lifetimes, AES-256 → 128-bit post-quantum security is mandatory. This contrasts with case 26 (car key fob, few years lifetime) and case 34 (smoke detector, ~10 years) where AES-128 was sufficient.

**Why ECDHE for forward secrecy?** If the long-term private key of either endpoint is later compromised (e.g., during a maintenance breach), past sessions — all recorded flight communications — must not be decryptable. ECDHE ephemeral keys are discarded after the handshake; compromise of the long-term key cannot recompute past session keys (ch3.6 p.37).

**Why mutual TLS and not one-way TLS?** One-way TLS only authenticates the server (ground station) to the client (aircraft), or vice versa. With only server authentication, a rogue aircraft could connect to the legitimate ground station and receive commands meant for a different flight. Mutual authentication ensures both endpoints are verified.

#### Command Layer — ECDSA Per Command

TLS provides session-level authentication — it proves the session is with the legitimate ground station. But within an established TLS session, a session key compromise (e.g., memory-scraping malware on the avionics processor) could allow command injection. For life-safety commands, a second independent authentication layer is required.

**Each command is signed with the ground station's ECDSA P-256 private key:**

```
command_packet = {
    aircraft_ID,
    ground_station_ID,
    co_pilot_ID,
    command_type,
    parameters,
    sequence_counter,
    timestamp,
    signature = ECDSA_sign(gs_private_key,
                    SHA-256(aircraft_ID || ground_station_ID || co_pilot_ID ||
                            command_type || parameters || sequence_counter || timestamp))
}
```

**Aircraft verification of each command:**

```
Step 1 — Verify TLS session is active and authenticated.
         (Ground station certificate validated against airline CA.)

Step 2 — Verify ECDSA signature:
         h = SHA-256(aircraft_ID || ground_station_ID || co_pilot_ID ||
                    command_type || parameters || sequence_counter || timestamp)
         ECDSA_verify(gs_public_key, h, signature)
         If FAIL → reject (signature invalid; possible injection or tampering).

Step 3 — Replay check: sequence_counter > last_accepted_counter?
         If sequence_counter ≤ last_accepted → reject (replay attack).

Step 4 — Freshness check: timestamp within acceptable window (ch3.1 p.3)?
         Aviation datalinks have ~600ms satellite latency; allow ±5 seconds.
         If timestamp too old or future-dated → reject.

Step 5 — Authorisation check: is this command type permitted in current mode?
         If override mode is not active and command_type = "flight_control" → reject.

Step 6 — Execute command. Log {command_packet, accepted_at_timestamp} to black box.
         Advance last_accepted_counter.
```

**Why sequence counter AND timestamp?** Counter alone fails on session restart (counter resets). Timestamp alone fails if the adversary has control of the network and can delay packets. Together, they provide mutual reinforcement (ch3.1 p.3 and p.7).

**Why ECDSA P-256 and not RSA-PSS?** ECDSA P-256 produces 64-byte signatures (ch2.2.3 p.85–87). RSA-PSS produces 256-byte signatures (ch2.2.3 p.88–93). Commands are transmitted over bandwidth-constrained aviation datalinks; smaller signatures reduce per-command overhead. Both provide 128-bit security. ECDSA is chosen for compactness and equivalent security.

**Why per-command signatures and not just session-level authentication?** Per-command signatures provide non-repudiation — a signed command with its timestamp is auditable evidence that the specific ground station (holding the private key) issued that specific command at that time. Session-level authentication only proves the session was established with that endpoint, not which specific commands were transmitted.

### Part 5 — Remote Co-Pilot Authentication at Ground Station

The human co-pilot must authenticate to the ground control station before the station activates its private key for any flight. Three independent factors:

**Factor 1 — Physical access control**: the ground control station is a physically secured facility. Unauthorised personnel cannot enter. Physical presence is required — remote login to the ground station over the internet is not permitted. This is an administrative control that limits the attack surface to insiders.

**Factor 2 — Password authentication** (ch3.2 p.11; ch3.1 p.7):

```
Storage: SHA-512(96-bit salt || password)
Login:
  Step 1 — Ground station → Co-pilot:  { salt, nonce }
  Step 2 — Co-pilot computes:           HMAC(SHA-512(salt || password), nonce)
  Step 3 — Co-pilot → Ground station:  { response }
  Step 4 — Ground station verifies:     response matches → authenticated
```

Challenge-response with nonce prevents pass-the-hash (ch3.1 p.7) — capturing the response from one session cannot be replayed.

**Factor 3 — TOTP second factor** (ch3.7 p.20, p.46):

```
TOTP = truncate(HMAC-SHA256(K_totp, floor(current_time / 30)), 6 digits)
```

The TOTP device is a dedicated hardware token — not an app on the same device used for monitoring. If the workstation is compromised, the attacker still needs the physical TOTP token.

**Flight assignment binding**: after authentication, the ground station is bound to a specific flight (`aircraft_ID`, flight number, departure time). The station's private key will only sign commands targeting that aircraft during that flight window. An authenticated co-pilot at Ground Station A cannot issue commands to a different aircraft than their assigned one.

### Part 6 — Certificate Management and Revocation

**Revocation (ch3.2 p.50–53)**: if a ground station is compromised (private key extracted), the airline CA adds the station's certificate to the CRL. The aircraft checks the CRL on each new TLS handshake attempt. Aircraft receive updated CRLs during scheduled datalink synchronisation.

**Why this is slower than online revocation**: aircraft in cruise over the ocean may not have high-bandwidth connectivity for real-time OCSP queries. The CRL is pre-loaded and updated at each waypoint report. For the override command pathway, a revoked certificate is checked against the last-known CRL — accepted risk that a very recent revocation may not propagate instantly in remote airspace. This is mitigated by the per-command ECDSA signatures: even a revoked certificate's session commands would be rejected once the aircraft receives the CRL update.

**Aircraft certificate rotation**: during scheduled maintenance, avionics receive updated certificates. The maintenance facility is physically secured and air-gapped from the public internet (same principle as case 29 — offline key management for high-value assets).

### Part 7 — Override Authorisation: Two-Mode Safety Design

**Normal (advisory) mode**: the ground co-pilot observes all telemetry, communicates with the onboard pilot, but issues zero flight control commands. Authentication at this level requires only the TLS session.

**Override mode activation** — this is the safety-critical transition:

```
Step 1 — Co-pilot at ground station presses "Request Override" button.
Step 2 — Ground station requires fresh TOTP re-authentication (ch3.7 p.20).
         (Even within an already-authenticated session — same principle as case 34 silence command.)
Step 3 — Ground station transmits a signed override-request command:
         ECDSA_sign(gs_private_key, SHA-256("OVERRIDE_REQUEST" || aircraft_ID || timestamp || nonce))
Step 4 — Aircraft avionics receives override request:
         a. Verifies ECDSA signature.
         b. Activates visible + audible cockpit alert: "REMOTE CO-PILOT OVERRIDE REQUESTED".
         c. Sends signed acknowledgment back to ground station:
            ECDSA_sign(aircraft_private_key, SHA-256("OVERRIDE_ACK" || aircraft_ID || timestamp))
Step 5 — Override mode is active. Ground co-pilot now has flight control authority.
```

**Onboard pilot veto**: if the onboard pilot is conscious and capable, they can press a physical "REJECT OVERRIDE" button on the flight deck, which sends a signed rejection command. Override mode is deactivated immediately — the onboard pilot retakes full authority.

**Why require fresh TOTP for override activation?** The stakes of accidental or attacker-triggered override are catastrophically higher than for a smart plug (case 27) or smoke detector (case 34). An active session token is insufficient — the co-pilot must positively re-authenticate at the moment of override. This prevents session hijacking or accidental button press from seizing aircraft control.

**Deactivation**: override mode is deactivated when: (a) the onboard pilot sends a signed rejection, (b) the co-pilot explicitly releases control, or (c) the ground link is lost (fail-safe: revert to local control).

### Part 8 — Availability and Link Failure (Fail-Safe)

Availability (ch1 p.42) is the defining constraint of this entire system. Two independent communication paths:

1. **Primary**: satellite datalink (global coverage, ~600ms latency, high bandwidth for video telemetry)
2. **Backup**: VHF datalink (line-of-sight, low latency, narrow bandwidth — sufficient for text commands)

**Automatic failover**: if the primary link drops for more than 5 seconds, the avionics automatically switches to VHF backup. The co-pilot's workstation transitions to low-bandwidth mode (no video; text commands only).

**If both links fail**:
- The aircraft immediately alerts the onboard pilot: "GROUND LINK LOST — RESUME SOLO OPERATION"
- Override mode (if active) is immediately deactivated — **fail-safe is local control**
- ATC is notified automatically via transponder
- The ground co-pilot is alerted; airline operations contacts ATC for radio coordination with the aircraft

**Why "fail to local control" and not "fail to remote control"?** Remote control depends on the communications link. If the link fails, remote control is simply not available — there is nothing to "fail into" on the remote side. The onboard pilot is physically present and capable of flying the aircraft. Defaulting to local control is the only rational fail-safe. (Contrast with case 34, where the physical alarm sounds regardless of cloud availability — same principle: the local physical layer always works.)

**Redundancy note**: unlike case 34, where WiFi jamming is a residual vulnerability with no countermeasure, here we have two independent radio technologies on different frequency bands. Jamming both simultaneously requires significant capability and is an act of war, not a typical criminal threat.

### Part 9 — System Security

**Ground control station** (ch3.7 p.46, p.51):
- Air-gapped from public internet — connects only over the aviation datalink
- Dedicated hardware: avionics-grade workstation not shared with general-purpose computing
- Private key stored in tamper-evident hardware — physical intrusion destroys the key material
- EPP (ch3.7 p.43): behaviour-based anomaly detection on the workstation OS
- Packet filter (ch3.7 p.51): only aviation datalink traffic on authorised protocols; no incoming internet connections
- IDS (ch3.7 p.77, p.85): detect anomalous patterns — unusual command volume, commands issued outside assigned flight window, login attempts outside assigned shift

**Aircraft avionics**:
- **Firmware signed with ECDSA P-256** (ch2.2.3 p.85–87) by the avionics manufacturer. The aircraft verifies the signature before installing any firmware update. A malicious firmware update could remove the ECDSA signature check on incoming commands — firmware signing prevents this.
- **Network segmentation** (ch3.7 p.46): avionics network is entirely separate from in-flight passenger WiFi, IFE systems, and crew tablets. No data path exists from passenger devices to avionics.
- **Minimal attack surface** (ch3.7 p.46): avionics exposes only the TLS endpoint for the command link and the datalink broadcast. No SSH, no web interface, no debug ports accessible in flight.
- **Command processor isolation**: the module that accepts and validates remote commands runs in a hardware-isolated partition. Even if the avionics OS is compromised, the command processor's ECDSA verification cannot be bypassed from software — it runs on dedicated secure hardware.

**Audit logging**: all commands received, accepted, rejected, and executed are written to the flight data recorder (black box) with their full signed payload. This provides tamper-evident non-repudiable evidence for incident investigation. The black box is write-once from the perspective of the flight software — entries cannot be deleted in flight.

### Part 10 — Threats and Remaining Vulnerabilities

| Threat | Attack | Why it fails (or residual risk) |
|---|---|---|
| **Command injection over datalink** | Attacker injects fabricated flight control command | Dual protection: TLS session authentication + per-command ECDSA; attacker lacks ground station private key; signature check rejects injected command |
| **Replay of valid command** | Captures "increase airspeed" command; replays it later | Sequence counter + timestamp; replayed command has counter ≤ last_accepted or timestamp outside window |
| **Man-in-the-middle on satellite link** | Intercepts and modifies command in transit | TLS 1.3 AEAD (GCM tag); per-command ECDSA; any modification invalidates both |
| **Co-pilot impersonation at ground station** | Attacker accesses ground station | Physical access control + password + TOTP three-factor; ground station private key in tamper-evident hardware |
| **Override activation by attacker** | Forces override mode to seize control | Fresh TOTP required for override; onboard pilot veto via physical button; signed acknowledgment required from aircraft before override activates |
| **Satellite link jamming** | Adversary jams primary uplink | Automatic VHF backup; if both fail, aircraft reverts to solo-pilot operation safely |
| **ATC frequency spoofing** | Fake ATC issues radio instructions to onboard pilot | Not a cryptographic attack on this system; standard ATC authentication procedures apply — out of scope for this design |

**Ground station private key extraction (residual vulnerability)**: physical access to the ground station hardware by an attacker (or malicious insider) with sufficient time and equipment could extract the private key, enabling command forgery for that aircraft. Mitigated by: tamper-evident hardware, physical access logs, two-person integrity rules for entering the station, and short certificate validity periods (frequent rotation). Detected by IDS on unusual command patterns.

**Satellite link latency (600ms) under override**: a remote co-pilot taking over in an emergency has ~600ms control latency via satellite. This is acceptable for altitude/heading changes but may be insufficient for rapid manoeuvres required to avoid collision. VHF backup has lower latency when in line-of-sight range. This is an operational limitation, not a cryptographic one.

**Onboard pilot threat**: if the onboard pilot themselves is the threat (rogue pilot scenario), the remote co-pilot system is a defence — but the override activation flow requires the aircraft to receive and execute the override request, which the rogue pilot could interfere with physically. This is a non-cryptographic safety design problem that requires avionics physical hardening (tamper-proof override controls) outside the scope of cryptographic security design.

**Certificate revocation latency over remote airspace**: a freshly compromised ground station certificate may not reach the aircraft's CRL before the aircraft crosses out of datalink range. Per-command ECDSA signatures remain valid even for a revoked certificate until the CRL is received. Mitigation: CRL pushes at high priority over the VHF backup channel when revocation occurs; small window of residual risk accepted.

### Part 11 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Session authentication | Mutual TLS 1.3 with X.509 ECDSA P-256 certificates | ch3.6 p.7–8; ch3.2 p.28–29 | Both endpoints authenticated; HMAC symmetric key non-repudiable and distribution problem; one-way TLS leaves aircraft unauthenticated to ground |
| Command authentication | ECDSA P-256 per command (signed payload with counter + timestamp) | ch2.2.3 p.85–87; ch3.1 p.3, p.7 | Non-repudiation: proof of command origin; HMAC cannot provide this; per-command replay prevention |
| Encryption | AES-256-GCM within TLS 1.3 | ch3.6 p.18; ch2 PQCrypto p.16 | 25–30 year aircraft lifetime; AES-128 → 64-bit under Grover's — insufficient for long-lived aviation systems |
| Forward secrecy | ECDHE in TLS 1.3 handshake | ch2.2.4 p.10; ch3.6 p.37 | Long-term key compromise cannot decrypt past flights; mandatory for long-lifetime systems |
| Human co-pilot auth | Password (SHA-512 + salt + challenge-response) + TOTP | ch3.2 p.11; ch3.1 p.7; ch3.7 p.20 | Three independent factors; TOTP on dedicated hardware token, not same device as workstation |
| Override activation | Fresh TOTP re-auth + signed request + aircraft acknowledgment | ch3.7 p.20, p.46 | Prevents accidental/attacker-triggered override; onboard pilot notified and can veto |
| Fail-safe | Link loss → immediate revert to local-pilot control | ch1 p.42 | Remote control requires link; fail-safe must be the mode that works without the link; onboard pilot always present |
| Firmware | ECDSA P-256 signed by avionics manufacturer | ch2.2.3 p.85–87 | Malicious firmware could remove signature checks on commands; signing chain prevents this |
| Replay prevention | Sequence counter (non-volatile) + timestamp per command | ch3.1 p.3, p.7 | Counter independent of time source; timestamp prevents large-window attacks; together they cover both reset and delay attacks |
| Revocation | CRL for certificate revocation; pushed at high priority over backup channel | ch3.2 p.50–53 | Compromised ground station certificate can be invalidated; pushed proactively rather than waiting for on-demand OCSP in low-bandwidth environment |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.40, p.42)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.63–66, p.70–75, p.85–87, p.88–93)
- IS_UG_2_2_4_SecM_KeyExch (p.10)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.28–29, p.50–53)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.51, p.77, p.83, p.85)

_Status: Complete_
_Done by: William_
