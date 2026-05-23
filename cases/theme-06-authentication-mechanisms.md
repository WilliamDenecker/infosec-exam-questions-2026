# Theme 6 — Authentication Mechanism Selection: Passwords, MFA, Certificates, and Challenge-Response

**Appears in**: Cases 21, 24, 27, 30, 33, 34, 35, 37

---

## The Three Levels of Authentication

Authentication problems in the course cases occur at three distinct levels. Each level has different requirements and solutions:

| Level | Who authenticates to what | Mechanism |
|---|---|---|
| **Machine-to-machine** | Device authenticates to server; sensor authenticates to hub | Pre-shared symmetric key + HMAC; or X.509 certificate + mutual TLS |
| **Human-to-machine** | User authenticates to app or server | Password + challenge-response + TOTP |
| **Command-level** | Individual high-stakes commands authenticated per-action | ECDSA signature per command; fresh TOTP per critical action |

Each level is independent. A user can authenticate to a server (human-to-machine), and within that session, the server still requires per-command authentication for life-safety operations (command-level).

---

## Part 1 — Human Authentication: The Full Stack

### Step 1 — Password Storage (Never Store Raw Passwords)

Passwords are stored as salted hashes (ch3.2 p.11):

```
stored_value = SHA-512(96-bit_salt || password)
```

Stored alongside the password hash: the salt. The database stores `(user_ID, salt, hash)`.

**Why SHA-512 (not SHA-256)**:
- Password hashes are long-lived — a database breach today exposes hashes that an attacker will try to crack for years
- Grover's quantum algorithm (ch2 PQCrypto p.16) halves effective hash security
- SHA-256: 128-bit preimage resistance → 64-bit post-quantum — potentially insufficient for long-lived passwords
- SHA-512: 256-bit preimage resistance → 128-bit post-quantum — adequate

**Why 96-bit salt**:
- Prevents rainbow table attacks: pre-computed tables of (password → hash) are useless because each password has a unique salt
- The attacker must brute-force each user's password individually with their specific salt
- 96 bits = 2^96 possible salts; attacker cannot pre-compute a table for any specific salt in advance

**Why not bcrypt/Argon2id?** These are not in the course slides. The exam expects SHA-512 + salt from ch3.2 p.11. Do not cite bcrypt or Argon2id.

---

### Step 2 — Login: Challenge-Response with Nonce (ch3.1 p.7)

Direct password transmission (even over TLS) has a weakness: if the stored hash is stolen, an attacker who replays the hash can authenticate. Challenge-response prevents this:

```
Server → Client: { salt_for_user, nonce }       // fresh random nonce each login
Client computes: H = SHA-512(salt || password)   // derive hash from entered password
                 R = HMAC-SHA256(H, nonce)        // bind hash to this session's nonce
Client → Server: { username, R }
Server verifies: HMAC-SHA256(stored_H, nonce) == R ?
```

**Why the nonce matters**: the nonce changes every session. A captured response R from session 1 is `HMAC(H, nonce_1)`. To replay it in session 2, the attacker needs to produce `HMAC(H, nonce_2)` — which requires H (the actual password hash). Stealing R does not give you H. This is the "pass-the-hash" attack prevention: even if the attacker intercepts R on the network, they cannot reuse it in any future session.

**Why this is not a substitute for TLS**: TLS still encrypts the login exchange. The challenge-response adds resistance against a scenario where TLS is somehow compromised or the response is intercepted in transit. Defence in depth.

---

### Step 3 — TOTP Second Factor (ch3.7 p.20)

TOTP (Time-based One-Time Password) adds a second authentication factor that requires physical possession of the authenticator device (phone/hardware token).

**Enrollment** (done once):
```
Server generates K_totp randomly (e.g., 160-bit random secret)
Server encodes K_totp as QR code; displays to user during account setup
User scans QR code with authenticator app; K_totp stored in app
K_totp is stored on server; never transmitted again after enrollment
```

**Code generation** (on the authenticator device — no network needed by the phone):
```
T    = floor(current_unix_timestamp / 30)    // time step; same value for both parties for 30 seconds
HMAC = HMAC-SHA256(K_totp, T)               // 32-byte MAC over the time step
code = dynamic_truncate(HMAC)               // extract 4 bytes starting at HMAC[last_byte & 0xF]; mod 10^6
```

The authenticator app computes this using only the locally stored K_totp and the device's clock. No network call is made by the app during code generation.

**Verification** (on the server):
```
Server computes code for T, T-1, T+1 (±1 time-step tolerance for clock drift)
Server accepts if received code matches any of the three computed codes
Server marks this time-step as used (prevents within-window replay)
```

**The critical precision point**: TOTP code generation is offline (phone needs no network). But authentication overall is NOT offline — the user types the code into the login form, which goes over the existing TLS session to the server. The phrase "TOTP works offline" means the authenticator device needs no inbound server contact. The submission still goes over TLS.

**Advantage over push notification 2FA**:
- Push notification: server must reach the phone (inbound connection to phone). Phone must be online and receive the notification. Phone sends approval back to server.
- TOTP: phone never needs to receive anything from the server. Code is computed locally. Phone's network state is irrelevant to code generation. This means TOTP works even when the phone has no data signal (subway, international roaming without data, airplane mode — as long as clock is synced).

---

### Step 4 — Re-Authentication for Critical Actions

For life-safety or irreversible commands, session authentication is insufficient. A session token can be stolen (XSS, session hijacking, malware on the device). Per-action authentication requires:

**Case 34 (silence smoke alarm)**: even if you are logged in, silencing the alarm requires a fresh TOTP code. An attacker who steals the session token cannot silence an alarm without also stealing the physical authenticator device.

**Case 37 (override mode activation)**: even if the ground co-pilot is authenticated to the system, activating override mode requires:
1. Fresh TOTP code (second factor possession at time of action)
2. ECDSA-signed override request (ground station private key)
3. Aircraft must send ECDSA-signed acknowledgment

Three independent checks. An attacker who compromises the session token does not have the TOTP device. An attacker who compromises the TOTP device cannot forge the ECDSA signature without the private key.

---

## Part 2 — Machine-to-Machine Authentication

### Pre-Shared Symmetric Key (for Constrained Devices)

Used when:
- The device is too constrained for certificate operations
- The number of devices is manageable
- Physical pairing at provisioning time is feasible
- Revocation can be handled at the server (application-level blacklist)

**Pattern**:
```
K_device = unique 128-bit key per device
Established during: physical provisioning, QR pairing, or manufacturing
Used in: HMAC-based message authentication; or as TLS pre-shared key
```

Revocation: add device_ID to a server-side blacklist. When K_device authenticates, server checks blacklist. Cryptographic revocation is not possible (the key remains mathematically valid) — it is enforced by server-side policy.

**Cases**: 26 (fob ↔ car), 27 (plug ↔ cloud), 28 (access card ↔ controller), 34 (detector ↔ hub)

### X.509 Certificates + Mutual TLS (for Higher-Value Devices)

Used when:
- Devices have sufficient compute for asymmetric operations
- Individual cryptographic revocability is required
- Devices must authenticate to multiple servers (the public key trust model scales)

**Pattern**:
```
Internal CA (Certificate Authority) issues certificates at manufacturing:
  Certificate = { device_ID, public_key, validity_period, CA_signature }

At TLS handshake:
  Device presents certificate → server verifies CA signature
  Server presents certificate → device verifies CA signature
  Both endpoints are mutually authenticated (ch3.6 p.7–8)
```

Revocation: CA publishes Certificate Revocation List (CRL) (ch3.2 p.50–53). The CRL is a CA-signed list of revoked certificate serial numbers. When a device's certificate is added to the CRL, all verifying parties automatically stop trusting it — without any physical access to the device. This is **cryptographic revocation**: the CA's signature on the CRL is unforgeable; simply sending a fake device_ID cannot bypass it.

**Why X.509 is needed when pre-shared keys are not sufficient**:
- Pre-shared key: device (K_device) is compromised → server adds device_ID to blacklist. But what if the device sends messages pretending to be device_ID_2? The server cannot distinguish spoofed device_IDs from the real thing unless the key is also correct. So for pre-shared key systems, K_device implicitly authenticates device_ID (you have the key → you are the device).
- X.509: the certificate binds device_ID to a public key, signed by the CA. Even if an attacker spoofs device_ID, they cannot produce a valid X.509 certificate signed by the internal CA (they do not have the CA's private key). Identity spoofing is cryptographically impossible.

**Cases**: 33 (factory IoT with PKI), 37 (aircraft ↔ ground station via TLS certificates)

---

## Part 3 — TOTP vs Hardware Tokens vs Push Notifications — Comparison

| Property | TOTP (authenticator app) | Hardware token (RSA SecurID type) | Push notification |
|---|---|---|---|
| K_totp storage | App secure storage | Hardware ROM | N/A |
| Code generation | Offline (phone clock + K_totp) | Offline (hardware clock + K_totp) | N/A — server sends approval request |
| Network required by auth device | No (generation only) | No | **Yes** (phone must receive push) |
| Works in airplane mode | **Yes** | **Yes** | No |
| Works when phone has no signal | **Yes** | **Yes** | No |
| Device lost = ? | Enroll new device | Replace hardware token | Re-register device |
| Clone resistance | App PIN + K_totp in secure storage | Hardware TPM; cannot extract K_totp | N/A |
| Slide reference | ch3.7 p.20 | ch3.7 p.20 | ch3.7 p.20 |

**For exam purposes**: all course cases use TOTP as the second factor. TOTP from ch3.7 p.20 is the standard choice. Do not use hardware tokens or push notification unless the case specifically mentions them.

---

## Part 4 — Command-Level Authentication: ECDSA Per Command

For high-stakes non-repudiable actions, session-level authentication (once per login) is insufficient. The solution is ECDSA signatures on individual commands:

```
command_packet = {
    data: { source_ID, target_ID, command_type, parameters, sequence_counter, timestamp },
    signature: ECDSA_sign(sender_private_key, SHA-256(data))
}
```

The receiver verifies:
```
ECDSA_verify(sender_public_key, SHA-256(data), signature) → accept / reject
```

**Why not just HMAC per command?** With HMAC(K, command), the receiver also holds K. If the aircraft verifies using K, the aircraft can also forge commands appearing to come from the ground station. The signed log in the flight data recorder is worthless — the aircraft itself could have written those "ground commands." ECDSA gives the ground station an unforgeable identity in the signed record.

**Cases using per-command ECDSA**: case 37 (aircraft commands), case 35 (per-message sender signature in group chat), case 36 (electronic stamp — Deutsche Post signs each stamp).

---

## Part 5 — Physical Authentication (What Proves Ownership)

Several cases involve physical authentication — proving you are physically present with a device:

**Case 26 (fob pairing)**: to add a new fob to the car, physical access to the car's OBD-II port is required (plus PIN entry). This proves you have the car. Fob pairing cannot be done remotely.

**Case 27 (plug pairing)**: pressing the physical button on the plug, then scanning the QR code, proves physical access to the plug. Remote enrollment is impossible.

**Case 34 (detector pairing)**: pressing the physical button on the smoke detector during pairing proves physical access to the detector. A remote attacker cannot press the button.

**Why physical authentication matters**: an attacker who has compromised the cloud server or the user's phone cannot pair a rogue device unless they physically possess the device. Physical presence is an authentication factor that cannot be stolen over a network.

---

## Part 6 — Summary: Mechanism by Scenario

| Scenario | Mechanism | Reference |
|---|---|---|
| Password storage | SHA-512(96-bit salt \|\| password) | ch3.2 p.11 |
| Login | Challenge-response with nonce + stored hash | ch3.1 p.7 |
| Second factor | TOTP: HMAC-SHA256(K_totp, T), ±1 window | ch3.7 p.20 |
| Device ↔ server (constrained) | HMAC-SHA256 with per-device K; symmetric pairing | ch2.2.3 p.63–66 |
| Device ↔ server (full compute) | Mutual TLS 1.3 with X.509 certificates | ch3.6 p.7–8; ch3.2 p.28–29 |
| Certificate revocation | CRL signed by internal CA | ch3.2 p.50–53 |
| Critical action | Fresh TOTP + ECDSA-signed command | ch3.7 p.20; ch2.2.3 p.85–87 |
| Physical pairing | Button press + QR code scan | ch3.7 p.46 |

---

## Slide References

- IS_UG_3_1_Appl_Basics (p.7: challenge-response; nonce-based login)
- IS_UG_3_2_Appl_AuthMeth (p.11: password storage; p.28–29: X.509; p.50–53: CRL)
- IS_UG_3_7_Appl_System (p.20: TOTP formula; p.46: MFA; physical pairing)
- IS_UG_2_2_3_SecM_HashMac (p.63–66: HMAC; p.85–87: ECDSA per-command)
- IS_UG_3_6_Appl_TLS (p.7–8: mutual TLS; p.18: certificate authentication)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16: SHA-512 choice over SHA-256 for password storage)
