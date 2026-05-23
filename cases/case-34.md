# Case 34

Smoke, fire and carbon monoxide detectors are getting smarter (e.g. Google's Nest Protect).

The smart detector combines a physical smoke detector and an app to be installed on the user's smartphone. When something is wrong (smoke, fire, etc.), the smart detector will set off an alarm signal and send an alert to the user's app with information about the detected issue. The user can check the alert and decide whether or not to silence the alarm using his/her app. The detector connects to the Internet through the wireless home network.

**What are the most essential security services? What security mechanisms could be used to ensure proper and (reasonably) secure operation of these smart detectors? What threats are there to this security? What vulnerabilities might remain?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | The app must verify that alarm alerts come from the legitimate detector, not a spoofed device. The detector must verify that silence commands come from the registered owner's app, not an attacker. | Attacker sends a fake "all clear" notification to the owner's app while a real fire is in progress. Owner dismisses a real alarm. |
| **Integrity** | Yes — critical | ch1 p.34 | Alert data (alarm type, severity) must not be modifiable in transit. A silence command must not be injectable without the owner's action. | Man-in-the-middle changes a "FIRE" alert to "test alarm" → owner does not evacuate. Attacker injects a silence command without the owner pressing anything. |
| **Availability** | Yes — critical | ch1 p.42 | **This is the most safety-critical service in this case.** The detector must send alerts and the app must receive them reliably during an emergency. A DoS attack that silences alerts could cost lives. The physical alarm must function even if the internet is unavailable. | DDoS on cloud server prevents alert delivery. Owner sleeping through a fire because the app notification never arrived. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | Only the registered owner's app may silence the alarm. No attacker, no neighbour, no cloud server operator may issue a silence command. | Any party can send a silence command; attacker silences smoke alarm to conceal arson. |
| **Confidentiality** | Yes | ch1 p.15 | Alert data reveals home occupancy (fire alarm at 2am = person is home and asleep). Command traffic timing reveals whether the owner is monitoring their home. | Attacker observes alert/silence patterns to determine occupancy and plan a burglary. |

### Part 2 — Critical Safety Property: Local Alarm Independence

Before designing the network security, one safety property must be stated as absolute:

**The physical alarm on the detector (the loud audible alert) must function regardless of internet/cloud availability.**

If the cloud server is down, unreachable, or under attack, the detector must still:
1. Sound its local alarm when smoke/CO is detected
2. Continue sounding until physically silenced at the device

The app silence functionality is a **convenience layer** on top of the physical alarm. The physical alarm is the primary safety mechanism and cannot be disabled by any network event.

**Consequence for security design**: a network DoS attack may prevent app notifications from arriving, but it cannot and must not silence the local alarm. The cloud server and app only receive alerts and relay silence commands — they do not control whether the local alarm sounds.

### Part 3 — Architecture: Cloud-Mediated (Same Principle as Case 27)

```
Smart detector  →  Cloud server  ←→  Smartphone app
```

The detector sends alert notifications to the cloud server (outbound only from detector perspective for normal operation). The app receives push notifications from the cloud server. Silence commands travel: app → cloud server → detector.

**Why cloud-mediated and not direct detector-to-app?** Same rationale as case 27 (smart plug): the detector should not have a publicly reachable internet address. Direct app-to-detector communication requires the detector to have a public IP or port-forwarded address — exposing constrained embedded hardware directly to the internet. The cloud server acts as an authenticated relay with access control enforcement.

**Key difference from case 27**: alerts (detector → app) are **outbound-initiated**. The detector pushes alerts when they occur. The cloud server must support push notification delivery to the app (which may not be running). This requires a persistent or polling connection from the cloud server to the mobile push notification infrastructure.

### Part 4 — Detector to Cloud Server: Authentication and Encryption

The detector is resource-constrained embedded hardware — same constraints as case 27 (smart plug). Symmetric AES-128-GCM is appropriate.

Each detector has a unique **128-bit pre-shared symmetric key K** loaded during manufacturing/pairing.

**Why AES-128 and not AES-256 for this link?** The detector is constrained embedded hardware — AES-256 is ~40% slower. Alert messages are short-lived (seconds to minutes of relevance). The threat lifetime is not years of stored data. AES-128 provides 128-bit classical security — sufficient for this use case (ch2.2.1 p.55).

**Why not asymmetric (ECDSA/ECDHE) for the detector link?** Asymmetric operations are computationally expensive on embedded hardware without dedicated crypto accelerators (ch2.2.2 p.7, p.13). An alert must be delivered immediately when smoke is detected — not after a multi-second ECDH computation.

Alert messages:

```
alert_packet = AES-128-GCM encrypt(K, { detector_ID, alert_type, severity, timestamp, counter })
```

**Counter** (ch3.1 p.7): monotonically increasing per alert. The cloud server rejects any alert with `counter â‰¤ last_seen_counter[detector_ID]`. This prevents an attacker from replaying old "all clear" notifications to suppress a real alert.

**Timestamp** (ch3.1 p.3): additional freshness. The cloud server rejects alerts with timestamps more than a few minutes in the past.

**Pairing**: same as case 27. Physical button press + QR code scan loads K into the cloud server via the TLS-protected app connection. Physical presence required — K never transmitted over the internet during pairing.

### Part 5 — Silence Command: Why Additional Authentication Is Required

The silence command is safety-critical — silencing a real fire alarm is potentially life-threatening. A silence command from the app must require **fresh MFA re-authentication** before being executed, not just a valid session token.

**Why not allow silence with just the session token?** If malware compromises the user's smartphone, it has access to the active session token and could issue a silence command silently while the user is unaware. Requiring fresh TOTP re-authentication means the attacker needs physical access to the TOTP device.

```
Silence command flow:
  1. User opens app, sees alert
  2. User presses "Silence alarm"
  3. App requires fresh TOTP code entry (ch3.7 p.20, p.46)
  4. User enters TOTP code
  5. App → Cloud: { silence_command, detector_ID, TOTP_verification, timestamp }
     (over TLS 1.3)
  6. Cloud verifies TOTP → forwards encrypted silence command to detector:
     AES-128-GCM encrypt(K, { silence, detector_ID, counter, timestamp })
  7. Detector verifies → silences alarm
```

**Why not biometric for silence?** Biometrics are not covered in slide material. TOTP is the appropriate second factor from slide material.

**Fail-safe**: if the cloud server is unreachable, the silence command cannot be delivered via the app. The user must physically press the button on the detector to silence it. This is the correct fail-safe — internet unavailability should never enable attacker to silence a fire alarm.

### Part 6 — Smartphone App to Cloud Server

Same as case 27 (smart plug), with the additional TOTP requirement for silence:

**Password storage**: `SHA-512(salt || password)` with 96-bit salt (ch3.2 p.11).

**Login**: challenge-response with nonce (ch3.1 p.7) to defeat pass-the-hash:

```
Step 1 — Server → App:    { salt, nonce }
Step 2 — App computes:    HMAC(SHA-512(salt || password), nonce)
Step 3 — App → Server:    { response }
Step 4 — Server verifies: response matches → authenticated
```

**MFA** (ch3.7 p.20, p.46): TOTP `truncate(HMAC-SHA256(K_totp, T), 6 digits)` for login, and additionally required for every silence command.

All traffic over **TLS 1.3** (ch3.6 p.7–8), `TLS_AES_256_GCM_SHA384`, ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).

**Why AES-256 for app-to-cloud (not AES-128)?** The smartphone and cloud server are not constrained. AES-256 is appropriate for all non-constrained links.

### Part 7 — Firmware Security

Firmware updates for the detector are signed with **ECDSA P-256** (ch2.2.3 p.85–87) by the manufacturer. The detector verifies the signature before installing any update.

**Why firmware signing matters for a smoke detector specifically**: a malicious firmware update could:
- Remove the alarm-triggering code so the device never triggers physically
- Remove the GCM tag verification so fabricated silence commands are accepted
- Add code to silence the alarm in response to an attacker's command

A signed firmware update prevents any of these attacks — only the manufacturer can push valid firmware.

**Why ECDSA P-256 and not RSA-PSS?** 64-byte compact signature vs 256-byte RSA signature. Faster verification on constrained hardware. (ch2.2.3 p.85–87 vs p.88–93)

### Part 8 — System Security

**Packet filter on home router** (ch3.7 p.51): the detector initiates outbound connections only. No inbound connections from the internet to the detector's IP are permitted.

**Minimise attack surface** (ch3.7 p.46): the detector runs minimal firmware — only smoke/CO detection logic, alert transmission, and command reception. No web server, no SSH, no debug interfaces.

**IDS on cloud server** (ch3.7 p.77, p.85): detect anomalous patterns — a rapid succession of fake alert/silence cycles (could indicate a false alarm flooding attack to exhaust batteries or habituate the user to ignore alerts), unusual geographic locations for silence commands, silence commands arriving before any alert was sent.

**EPP on cloud servers** (ch3.7 p.43).

### Part 9 — Specific Threats

| Threat | Attack description | Mitigation |
|---|---|---|
| **Fake alert injection** | Attacker sends spoofed fire alert to owner's app, causing panic or evacuation | AES-128-GCM on detector→cloud link: only K-holder can produce valid alert. Cloud rejects alerts with invalid GCM tags. |
| **Alarm silencing** | Attacker sends silence command to suppress a real alarm | Silence requires fresh TOTP re-auth; counter prevents replay of captured silence commands |
| **False alarm DoS** | Attacker triggers repeated false alarms to exhaust owner's patience → owner disables detector | IDS detects abnormal alarm frequency; rate limiting on alerts; physical button required for local acknowledgment |
| **Cloud DoS** | DDoS on cloud server prevents alert delivery | Physical alarm continues regardless; cloud is for notification only, not for alarm triggering |
| **Replay of old silence command** | Attacker captures a valid silence command and replays it later | Monotonic counter + timestamp: replayed packet has counter â‰¤ last_seen or timestamp too old → rejected |
| **Malicious firmware** | Attacker pushes firmware that disables alarm logic | ECDSA P-256 signed firmware; detector verifies signature before installing |

### Part 10 — Comparison with Case 27 (Smart Plug)

| | Case 27 — Smart plug | Case 34 — Smart smoke detector |
|---|---|---|
| **Safety consequences of attack** | Moderate (appliance disruption) | Life-threatening (fire alert suppressed) |
| **Silence/disable command** | Normal command, session auth sufficient | Safety-critical: requires fresh TOTP MFA every time |
| **Cloud unavailability** | Plug uncontrollable | Physical alarm still sounds; app notification fails but alarm continues |
| **Alert direction** | App → Detector (commands) | Detector → App (alerts) + App → Detector (silence) |
| **Physical failsafe** | Not applicable | Physical button on detector can silence locally, independently of cloud |

### Part 11 — Remaining Vulnerabilities

- **WiFi jamming**: an attacker jamming the home WiFi network prevents alert delivery to the cloud and thus to the app. The physical alarm still sounds — but the owner receives no remote notification. There is no cryptographic countermeasure to radio jamming. Multiple communication channels (WiFi + cellular backup) would mitigate this.
- **Smartphone compromise with TOTP app on same device**: if the TOTP authenticator app runs on the same device as the smart detector app, a compromised smartphone gives the attacker both factors simultaneously. A separate hardware TOTP token (case 25 HardKey) would prevent this.
- **Cloud server breach**: attacker gains K for all detectors managed by that server. Can forge alerts or suppress alert delivery. K must be stored encrypted at rest, key separate from data.
- **Alarm fatigue from false alarms**: repeated false alarms (from cooking, dust, etc.) train the user to dismiss alerts quickly without checking. This is a social engineering vulnerability — not cryptographic — but it reduces the effective security of the alerting system.

### Part 12 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Detector→cloud authentication | AES-128-GCM with pre-shared K, counter, timestamp | ch2.2.3 p.70–75; ch2.2.1 p.55 | Fast symmetric for constrained hardware; AEAD; asymmetric too slow for immediate alert delivery |
| Silence command | Requires fresh TOTP MFA re-authentication every time | ch3.7 p.20, p.46 | Session token alone insufficient for life-safety command; TOTP requires physical second device |
| Physical alarm independence | Alarm sounds regardless of cloud/network state | ch1 p.42 | Network attack cannot silence physical alarm; cloud is notification layer only |
| App authentication | SHA-512 + challenge-response + TOTP | ch3.2 p.11; ch3.1 p.7; ch3.7 p.20 | Pass-the-hash defeated; two independent factors |
| Firmware | ECDSA P-256 signed by manufacturer | ch2.2.3 p.85–87 | Rogue firmware cannot disable alarm logic or remove security checks |
| Transport (app↓cloud) | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Mandatory AEAD; forward secrecy; non-constrained link justifies AES-256 |
| Key distribution | Physical pairing via QR code | ch3.7 p.46 | Physical presence required; K never transmitted over internet |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.63–66, p.70–75, p.85–87, p.88–93)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11)
- IS_UG_3_6_Appl_TLS (p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.51, p.77, p.83, p.85)

_Status: Complete_  
_Done by: William_
