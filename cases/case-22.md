# Case 22

A company implements two-factor authentication (2FA) for access to its servers (e.g. mail server). This 2FA relies on a password (first factor) and an authenticator app (second factor) installed on a mobile device (e.g. a smartphone).

When the user requests access to the company server from his PC, the user will enter her/his password on her/his PC. Then the authenticator app (on the user's mobile device) will ask the user to approve this access. Only after this approval will access be granted to the company server.

**Describe a plausible protocol, with adequate security mechanisms, that could implement such a 2FA.**

**What are the improvements with respect to simple password-based authentication? What are the possible drawbacks?**

**What might be the remaining vulnerabilities?**

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | The server must verify the identity of the connecting user using two independent factors. Password alone is insufficient — it can be stolen without the user knowing. | A stolen password gives full server access. Single-factor auth is the leading cause of account compromise (ch3.7 p.20). |
| **Confidentiality** | Yes | ch1 p.15 | Credentials and the push notification content must be encrypted in transit. | An eavesdropper captures the approval signature and replays it before the nonce expires. |
| **Data integrity** | Yes | ch1 p.34 | The push notification and approval must not be modifiable in transit. | A man-in-the-middle substitutes a different approval context, tricking the user into approving a different login. |
| **Availability** | Yes | ch1 p.42 | Employees must be able to log in when needed. Push delivery failures must not permanently block legitimate access. | The push service is unreachable; all company employees are locked out of the mail server. |

### Part 2 — Protocol Description

This is a **push-based** second factor, as opposed to the time-based TOTP code (case 6). The server pushes a login request to the app for the user to approve, rather than the user reading and typing a code.

#### Setup (Enrolment — One Time)

During enrolment, the user's smartphone app generates an **ECDSA P-256 keypair** (ch2.2.3 p.85–87). The **public key** is registered with the server alongside the user's account. The **private key** is stored in the phone's hardware-backed secure storage and never leaves the device.

**Why ECDSA P-256 over a symmetric shared secret (as in TOTP)?** TOTP requires the server to store a shared secret K per user — if the TOTP key database is breached, all users' OTP generators are exposed. With ECDSA, the server only stores the public key — which is useless to an attacker without the private key. The private key never leaves the phone's hardware.

The password is stored on the server as `SHA-512(salt || password)` with a per-user random salt of at least 96 bits (ch3.2 p.11).

#### Login Protocol

All steps over **TLS 1.3** (ch3.6 p.5, p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).

**First factor — password verification:**

```
Step 1 — Client → Server:   username
Step 2 — Server → Client:   { salt, nonce_pw }
Step 3 — Client computes:   HMAC(SHA-512(salt || password), nonce_pw)
Step 4 — Client → Server:   response
Step 5 — Server:            verifies response; if no match → reject
                             threshold failures → account lockout (ch3.7 p.85)
```

**Second factor — push approval:**

```
Step 6 — Server:            generates approval nonce N; valid for 60 seconds
Step 7 — Server → Phone:    push notification:
                             { N, username, requesting_device_info, source_IP, timestamp }
Step 8 — Phone display:     "Login from device X at IP Y at time T — Approve or Deny?"
Step 9 — User taps Approve
Step 10 — Phone computes:   approval = ECDSA_sign(private_key, SHA-256(N || username || timestamp))
Step 11 — Phone → Server:   { approval }   (via its own TLS connection)
Step 12 — Server:           ECDSA_verify(registered_public_key,
                                          SHA-256(N || username || timestamp),
                                          approval)
                             N marked spent; if valid and within 60s window → access granted
```

**Why is the nonce N essential?** The nonce (ch3.1 p.7) binds the approval to this specific login attempt. An approval signature captured from one login attempt cannot be replayed to a new attempt — N is different each time. The 60-second validity window limits the replay window.

### Part 3 — Improvements over Password-Only Authentication

**1. Two independent factors** (ch3.7 p.20): a stolen password alone is insufficient — the attacker must also approve the push on the physical phone. MFA is absent in 59% of incidents (ch3.7 p.20).

**2. Context awareness**: unlike TOTP where the user types a code without context, the push notification shows the **requesting device type, source IP, and timestamp**. A user who receives an unexpected push (they did not just try to log in) knows immediately that their password has been stolen and can deny the request — providing active fraud detection.

**3. No code to type**: eliminates shoulder-surfing and TOTP code interception risk. The user only taps Approve on their phone.

**4. Private key is non-exportable**: the phone's ECDSA private key is stored in hardware-backed secure storage. Even if the phone OS is compromised, extracting the raw key requires bypassing hardware security.

**5. Fresh nonce per request** (ch3.1 p.7): the approval signature is bound to a specific N — it cannot be replayed to a different login request.

### Part 4 — Possible Drawbacks

**1. MFA push fatigue (bombing attack)**: an attacker who has stolen the password can trigger repeated push notifications, hoping the user approves one out of annoyance, habit, or confusion. Users must be trained to never approve unexpected push requests.

**2. Internet connectivity required on phone**: the phone must be reachable to receive the push notification. No network on the phone = no second factor = legitimate user is locked out (ch1 p.42).

**3. Push infrastructure dependency**: the server requires a reliable push delivery mechanism. Downtime of the push service blocks all employee logins — a single point of failure.

**4. Login friction** (ch3.2 p.14 — usability vs. security trade-off): the user must have their phone available, switch to it, read the context, and tap Approve at every login. More friction than a saved password.

### Part 5 — Remaining Vulnerabilities

- **Real-time phishing (adversary-in-the-middle)**: an attacker proxies the entire login in real time. On the legitimate server, the server pushes N to the victim's phone. The attacker relays before N expires. The user approves — thinking it is their own login. The attacker acquires the authenticated session. TLS strict certificate validation (ch3.2 p.28) prevents proxy attacks if properly implemented and the user does not override warnings.
- **Phone theft with unlocked screen**: an attacker holding the unlocked phone can manually approve a push request. Requiring a phone PIN or biometric to open the authenticator app mitigates this.
- **Push notification interception**: if the push delivery infrastructure is compromised, a forged push could be sent to the user. The ECDSA signature verification ensures only the legitimate phone's private key can produce a valid approval — a forged push that never reaches the real phone will never generate a valid signature.
- **Simultaneous compromise of both factors**: malware on the PC captures the password; malware on the phone silently approves the push. Both factors are defeated together. EPP on both devices (ch3.7 p.43) reduces this risk.

### Part 6 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Second factor keypair | ECDSA P-256 on phone | ch2.2.3 p.85–87 | Server stores only public key; no shared secret to steal; hardware-bound private key |
| Push approval binding | ECDSA signature over nonce + username + timestamp | ch3.1 p.7; ch2.2.3 p.85–87 | Nonce prevents replay; signature proves phone possession |
| Password storage | SHA-512 + 96-bit salt | ch3.2 p.11 | Rainbow table attack defeated; no known practical collision |
| Password transmission | HMAC challenge-response with nonce | ch3.1 p.7; ch3.2 p.11 | Pass-the-hash defeated; stolen hash cannot be replayed |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Forward secrecy; AEAD; real-time MitM prevented |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.34, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.63–66, p.85–87)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11, p.13–14, p.28)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46, p.85)

_Status: Complete_  
_Done by: William_
