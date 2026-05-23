# Case 27

Smart devices (lights, doorbells, locks, thermostats, etc.) can be switched on and off, and controlled using an app on a smartphone.

When you have an existing installation however, you may not want to replace all existing devices with new smart devices. A *smart plug* might be a partial solution in this case. They can be simply plugged into a regular power outlet and will control the electrical power fed to the device plugged into the smart plug (e.g. on/off/dimmed).

The user can control the smart plug using an app on his smartphone. There is also a remote cloud server that can interact with both the controller app and the smart plug. The smart plug is connected to the Internet through the wireless home network.

**What are the most essential security services? What security mechanisms could be used to ensure proper and (reasonably) secure operation of these smart plugs? What threats are there to this security? What vulnerabilities might remain?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | The smart plug must verify commands come from the legitimate owner, not a neighbour or remote attacker. The cloud server must verify the user's identity before forwarding any command. | An attacker sends a command to disable a security camera, unlock a smart lock wired through the plug, or disrupt heating. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | Only the registered owner may control their specific plug. The cloud server enforces this — authentication proves identity but not ownership of a specific plug. | Any authenticated platform user can control any other user's plug. A neighbour turns off your refrigerator at night. |
| **Data integrity** | Yes | ch1 p.34 | A command must arrive unchanged. "Turn off for 5 minutes" must not become "turn off permanently". | A man-in-the-middle changes a timed command to a permanent disable, cutting power to medical equipment. |
| **Confidentiality** | Yes | ch1 p.15 | Command traffic and timing reveals behavioural patterns — when the owner is home, sleep schedule, whether the house is occupied. | An attacker monitors command timing and infers the owner's daily routine; the house is burgled during predicted absence. |
| **Availability** | Yes | ch1 p.42 | The plug must respond to legitimate commands reliably. Denial of service must not permanently disable control. | A DDoS on the cloud server makes all plugs uncontrollable. Heating shuts off in winter; a ventilator for medical equipment stops. |

### Part 2 — Architecture: Why Cloud-Mediated and Not Direct

Three architectural options exist:

**Option A — Direct communication (smartphone → plug directly):** The plug accepts commands directly from the smartphone app on the local WiFi network. No cloud needed. Problem: remote control from outside the home is impossible — the plug has no publicly reachable address. Modern home routers use NAT; the plug is not directly reachable from the internet. Additionally, the plug would need to accept unsolicited inbound connections — a security risk on constrained hardware.

**Option B — Plug exposed directly on the internet:** The plug gets a public IP (or port-forwarded address). Commands arrive directly from the smartphone. This exposes the plug — a constrained embedded device with limited security capabilities — directly to internet-scale scanning, exploitation attempts, and denial-of-service. Embedded IoT devices have a poor security track record; direct internet exposure is dangerous (ch3.7 p.46 — minimise attack surface). **Rejected**.

**Option C — Cloud-mediated (chosen):**

```
Smartphone app  ←→  Cloud server  ←→  Smart plug
```

The plug **initiates an outbound connection** to the cloud server and waits for commands. It has no publicly reachable address; no uninitiated inbound connections from the internet can reach it. The cloud server is the only internet-facing component — and it is professional infrastructure with proper security hardening, not a constrained embedded device. Remote control is possible because the cloud server relays authenticated commands down the persistent outbound connection the plug maintains.

**Trade-off**: if the cloud server is unavailable, the plug cannot be controlled remotely. This is an inherent property of the cloud-mediated design and is accepted as a residual risk.

### Part 3 — Smartphone to Cloud Server

#### TLS 1.3 Transport

All smartphone-to-cloud traffic over **TLS 1.3** (ch3.6 p.7–8) with cipher suite `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).

**Why TLS 1.3 and not TLS 1.2?** TLS 1.2 permits non-AEAD cipher suites (e.g., AES-CBC with separate HMAC) which are vulnerable to padding oracle attacks. TLS 1.3 mandates AEAD-only, removes deprecated features, and is faster (one round-trip handshake). (ch3.6 p.7–8)

**Why AES-256-GCM for this link and not AES-128-GCM?** The smartphone is not a constrained device — it has full compute capability. The cloud server likewise. The 40% performance cost of AES-256 over AES-128 is imperceptible on modern hardware. AES-256 provides 128-bit effective security post-quantum (ch2 PQCrypto p.16) — appropriate for a consumer IoT service where user data may be retained for years.

#### User Authentication: Password + TOTP MFA

**Why MFA is required** (ch3.7 p.20, p.46): MFA is absent in 59% of security incidents. A smart plug controlling heating or cameras represents significant physical-world impact — single-factor password authentication is insufficient.

**Factor 1 — Password:** stored as `SHA-512(salt || password)` with 96-bit random salt (ch3.2 p.11).

**Why SHA-512 and not SHA-256 for password storage?** Against Grover's quantum algorithm (ch2 PQCrypto p.17), SHA-256 provides 128-bit preimage resistance → 64-bit post-quantum. SHA-512 provides 256-bit preimage resistance → 128-bit post-quantum. For a long-lived password hash, SHA-512 is the correct choice.

**Login: challenge-response with nonce** (ch3.1 p.7) to defeat pass-the-hash attacks:

```
Step 1 — Server → App:    { salt, nonce }
Step 2 — App computes:    response = HMAC(SHA-512(salt || password), nonce)
Step 3 — App → Server:    { response }
Step 4 — Server verifies: HMAC(stored_hash, nonce) == response → Factor 1 passed
```

**Why challenge-response and not direct password submission?** If the user sends the raw password, a network capture (despite TLS) or server DB leak reveals it. If the user sends `SHA-512(salt || password)` directly — a stored hash on the server can be replayed as-is (pass-the-hash attack). Challenge-response means a stolen stored hash is useless without the fresh nonce — the nonce changes every login. A captured response from session A does not work in session B.

**Factor 2 — TOTP** (ch3.7 p.20, p.46):

TOTP (Time-based One-Time Password) generates a short numeric code from a shared secret key K_totp combined with the current time. Both the authenticator app and the cloud server know K_totp and independently compute the same code — without any network communication at code-generation time. The code changes every 30 seconds and is valid for exactly that window, giving replay protection by design.

**Enrollment — how K_totp is established**: when the user enables MFA, the cloud server generates a unique random K_totp for this account. The server encodes K_totp as a QR code and displays it once. The user scans the QR code with their authenticator app (Google Authenticator, etc.), which stores K_totp locally. After this moment, both the server and the app independently hold K_totp — it is never transmitted again. The QR scan happens over the already-active TLS session, so K_totp is not exposed to the network.

**Code generation** (ch2.2.3 p.63–66):

```
T    = floor(current_unix_timestamp / 30)    // integer time step; increments every 30 s
HMAC = HMAC-SHA256(K_totp, T)               // 32-byte MAC keyed by the secret, input is the time step
code = truncate(HMAC, 6 digits)             // take 4 bytes from a position indicated by HMAC's last nibble;
                                            // reduce modulo 10^6 → 6-digit decimal code
```

The truncation is deterministic and produces the same 6-digit output on the app and the server as long as T is the same — which it is when both clocks agree to within 30 seconds.

**Verification**: when the user types the 6-digit code into the login screen, the cloud server:
1. Looks up K_totp for this account.
2. Computes `HMAC-SHA256(K_totp, T_current)` and truncates to 6 digits.
3. Compares with the submitted code. Accepts a Â±1 time-step window (i.e., also checks T-1 and T+1) to tolerate minor clock drift between the phone and server.
4. If accepted, marks this time step as used — a second login attempt with the same code within the same 30-second window is rejected (prevents replay within the window).

**Why TOTP provides replay protection**: the code is bound to the current time step T. A code captured at step T=N is only valid at T=N. Thirty seconds later the server computes a different value and the captured code no longer matches. An attacker who intercepts a code has at most 30 seconds to use it — by which time the legitimate user has already authenticated and the time step is marked spent.

**Why HMAC-SHA256 and not plain SHA-256(K_totp || T)?** The length-extension attack (ch2.2.3 p.24–32) allows computing `SHA-256(K || T || padding || X)` without knowing K. HMAC's nested structure prevents this. The TOTP code must be unforgeable — HMAC provides the keyed MAC property that guarantees only the holder of K_totp can produce the correct code for any T.

**Why TOTP and not SMS 2FA?** SMS codes are delivered through the telephone network. An attacker who SIM-swaps the victim's number (convinces the carrier to transfer the SIM to an attacker-controlled device) receives all SMS messages, including the OTP. SS7 protocol flaws also allow interception of SMS traffic. TOTP is entirely local — no SMS infrastructure is involved; the code is computed on-device from K_totp and the clock.

**Why TOTP and not push notification 2FA?** With push notification, the server must actively reach the authenticator device — the phone must be online and reachable to receive the push, and it sends the approval back to the server. With TOTP, the authenticator device (phone) never needs to receive anything from the server: the code is computed locally from K_totp and the device clock, then typed into the login form over the existing TLS session. The phone's network state is irrelevant to code generation. Note that the overall authentication still goes over the network (the TLS session) — the distinction is that the code-generating device requires no server contact, not that authentication is "fully offline". This also eliminates the MFA-fatigue attack vector — there is no "approve" button for an attacker to repeatedly push.

### Part 4 — Cloud Server to Smart Plug

#### Why Symmetric Encryption and Not Asymmetric

The smart plug is a constrained embedded device — it has a small microcontroller, limited RAM, and must operate on the power available from the wall outlet (but with minimal energy waste on computation).

RSA and ECDSA (ch2.2.2 p.7, p.13) require asymmetric operations — modular exponentiation or elliptic curve point multiplication. These are orders of magnitude more expensive than AES. An embedded microcontroller without dedicated crypto hardware would take seconds to complete an ECDH handshake per command — completely impractical.

**Symmetric AES is the correct choice** for the plug's cipher — fast, energy-efficient, and available as hardware acceleration even on constrained microcontrollers.

#### AES-128-GCM for Commands: Why 128-bit and Not 256-bit Here

**Why AES-128 and not AES-256 for the plug link?** The smart plug runs on constrained embedded hardware. AES-256 requires a more complex key schedule and is approximately 40% slower than AES-128 on hardware without AES-256 acceleration. Furthermore, the commands sent to the plug are short-lived (seconds to minutes), not long-lived data. The threat model for IoT commands is an attacker compromising the current session, not a quantum computer decrypting archived traffic from years ago. AES-128 provides 128-bit classical security — sufficient for this use case. (ch2.2.1 p.55)

**Contrast with TLS 1.3 on the smartphone link**: the smartphone is not constrained; there is no performance penalty for AES-256. The cloud→plug link specifically justifies AES-128 because of the hardware constraint.

#### Why AES-128-GCM and Not AES-128-CBC

AES-128-CBC (with a separate HMAC for integrity):
- Requires two operations: CBC for confidentiality, then HMAC for integrity
- Padding oracle vulnerability if padding errors are distinguishable
- Two separate keys needed (encryption key + MAC key)

AES-128-GCM (ch2.2.3 p.70–75):
- Single AEAD operation: confidentiality + integrity in one pass
- No padding (stream cipher mode within GCM)
- No padding oracle attack surface
- 128-bit GCM authentication tag detects any tampering

**Chosen**: AES-128-GCM.

#### Command Packet Structure and Replay Protection

```
command_packet = AES-128-GCM encrypt(K, { command, plug_ID, counter, timestamp })
                 ↑ includes 128-bit GCM authentication tag
```

The **counter** is a monotonically increasing integer maintained by the cloud server per plug. The plug stores `counter_last` and rejects any packet with `counter â‰¤ counter_last`. The cloud server increments the counter with every command issued.

**Why a counter and not a nonce for replay protection?** The plug is embedded hardware — it cannot generate a cryptographically random nonce per connection. The server could generate nonces, but the plug then needs a round-trip acknowledgement to confirm receipt before advancing state. A monotonic counter incremented by the server is simpler and stateless from the plug's perspective: just check counter > counter_last.

**Why include timestamp in addition to counter?** The timestamp provides an additional expiry: even if an attacker captures a valid packet and the counter mechanism somehow fails (e.g., counter_last state corrupted), a timestamp too far in the past causes the plug to reject it. Belt-and-suspenders freshness (ch3.1 p.3).

#### Pairing: Key Distribution

During initial setup, the user presses a physical button on the plug (requires physical presence) and the smartphone app scans a QR code printed on the plug. The QR code encodes K. K is transmitted from the app to the cloud server over the already-established TLS connection.

**Why physical presence and not internet-based pairing?** If K is generated on the server and transmitted to the plug over the internet during setup, an attacker who intercepts or compromises that transmission obtains K. Physical proximity during pairing means the attacker must be physically present in the home to intercept the QR code — a significantly harder attack than remote interception.

**Why QR code and not NFC?** Both require physical proximity. NFC would also work. QR code is chosen because all smartphones have cameras and no special NFC hardware configuration is needed. The security property (physical proximity required) is equivalent.

**Why not Diffie-Hellman during pairing?** The plug could perform a DH key exchange during initial pairing to establish K without the QR code. DH during pairing would require the plug to display or transmit its DH public value — but the plug has no display. The QR code is simpler and achieves the same physical-proximity requirement without the plug needing display capability.

### Part 5 — Access Control

The cloud server enforces (ch1 p.30): `owner(plug_ID) == authenticated_user_ID` before forwarding any command. This check happens server-side, after authentication, before any packet is sent to the plug.

A user who somehow learns another user's plug_ID cannot control it — the ownership check prevents forwarding the command even with valid account credentials.

### Part 6 — Firmware Security

Firmware updates for the smart plug are signed with **ECDSA P-256** (ch2.2.3 p.85–87) by the manufacturer. The plug stores the manufacturer's public key in read-only memory and verifies the signature before installing any update.

**Why ECDSA P-256 and not RSA-PSS?** RSA-PSS (ch2.2.3 p.88–93) at equivalent security (2048-bit) produces 256-byte signatures. ECDSA P-256 produces 64-byte signatures. On a constrained microcontroller with limited flash storage for the update verification routine, the smaller signature is significant. ECDSA verification is also faster on constrained hardware.

**Why sign firmware at all?** If firmware is not signed, the cloud server (or an attacker who compromises it) can push a malicious firmware update that removes all security checks, opens a backdoor, or turns the plug into a bot. The manufacturer's private key is kept offline — only the manufacturer can produce a valid firmware signature.

### Part 7 — System Security

**Packet filter on home router** (ch3.7 p.51): the plug initiates outbound connections only. The home router's NAT and packet filter block all uninitiated inbound connections from the internet to the plug's private IP address. The plug is completely invisible from the public internet.

**Minimise attack surface** (ch3.7 p.46): the plug runs minimal firmware — command processing and connectivity only. No web server, no SSH daemon, no Telnet, no debug interfaces left enabled. Every service not needed for the plug's function is a potential attack vector.

**IDS on cloud server** (ch3.7 p.77, p.85): detect anomalous patterns — rapid repeated commands, commands from geographically impossible login locations, mass commands across all plugs of one user (indicator of compromised account). Threshold detection (ch3.7 p.85) triggers alerts and temporary lockout.

**EPP on cloud servers** (ch3.7 p.43): behaviour-based malware detection on the cloud infrastructure.

**Encrypted K storage on cloud server**: the cloud server stores K for each plug in an encrypted form (AES-256-GCM at rest, key separate from data). If the cloud server is compromised and the database is extracted, the plug keys are not immediately readable.

### Part 8 — Remaining Vulnerabilities

- **Cloud server breach**: if the cloud server is compromised and the encrypted K database is decrypted, the attacker gains K for all plugs. All plugs become controllable. Encrypting K at rest and separating the encryption key from the data mitigates but does not eliminate this risk. (ch3.7 p.83, p.96 — log retention for retroactive detection)
- **WiFi network compromise**: an attacker on the same local network as the plug cannot forge valid AES-128-GCM commands without K, but can perform denial-of-service by flooding or jamming the plug's WiFi connection.
- **Availability dependency on cloud**: if the cloud server is unavailable, the plug cannot be controlled remotely or locally (since all commands route through the cloud). This is an inherent trade-off of the cloud-mediated architecture.
- **Physical access to plug**: an attacker with physical access can factory-reset the plug and re-pair it with their own account. Physical security is a prerequisite in sensitive environments. The plug should be in a physically secured location for high-security use cases.
- **MFA fatigue / social engineering**: if the attacker obtains the user's password and bombards the user with TOTP requests, a distracted user might approve one. TOTP mitigates push-fatigue (unlike push notification 2FA) because the code must be actively generated — there is no "approve" button to click accidentally.

### Part 9 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Architecture | Cloud-mediated; plug initiates outbound only | ch3.7 p.46 | Plug not internet-facing; no inbound attack surface; professional cloud infrastructure more secure than constrained embedded device |
| Smartphone auth | SHA-512 password + challenge-response + TOTP MFA | ch3.2 p.11; ch3.1 p.7; ch3.7 p.20 | Pass-the-hash defeated; two independent factors; TOTP over SMS (SS7 attacks) |
| Smartphone transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Mandatory AEAD; forward secrecy; AES-256 appropriate for non-constrained device |
| Plug command protection | AES-128-GCM per command | ch2.2.3 p.70–75; ch2.2.1 p.55 | AEAD in one pass; no padding oracle; AES-128 appropriate for constrained hardware; symmetric avoids asymmetric computation cost |
| Replay protection | Monotonic counter + timestamp | ch3.1 p.3, p.7 | Old commands permanently spent; timestamp provides additional expiry; no round-trip needed |
| Key distribution | Physical pairing via QR code | ch3.7 p.46 | Physical presence required; K never transmitted over internet during setup; attacker cannot intercept remotely |
| Firmware security | ECDSA P-256 signed by manufacturer | ch2.2.3 p.85–87 | 64-byte compact signature; fast verification on constrained hardware; rogue firmware cannot be pushed without manufacturer private key |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.63–66, p.70–75, p.85–87, p.88–93)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16–17)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46–47, p.51, p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_
