# Case 16

A university wants to replace password-based login for students and staff by passkeys (WebAuthn), usable across multiple services (learning platform, email, exam platform). Students use laptops and smartphones, sometimes shared devices.

**Design an appropriate security solution. Which security services are essential? Which cryptographic mechanisms and protocols would you choose? How are keys generated, stored, and used? What threats remain (phishing, malware, lost devices)? Discuss usability trade-offs.**

## Answer

### Part 1 — Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | Students and staff must be verified before accessing any university service. The system must resist phishing and credential theft. | A student submits exam answers under another student's identity. An attacker logs in with a stolen password and accesses confidential grade data. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | A student must not access staff-only resources; a student cannot access another student's exam submissions or grades. | Any authenticated user can access any university resource regardless of role. |
| **Data integrity** | Yes | ch1 p.34 | The authentication assertion (signed proof of login) must not be modifiable in transit between the device and the server. | A man-in-the-middle modifies the authentication assertion to substitute a different user's identity for the actual signer. |
| **Confidentiality** | Yes | ch1 p.15 | Login credentials and session data must not be readable by any eavesdropper. TLS provides this. | An eavesdropper captures the authentication exchange and replays it to gain access. |

### Part 2 — Core Mechanism: Asymmetric Challenge-Response per Device

Passkeys (WebAuthn) apply **asymmetric challenge-response authentication** (ch3.1 p.7) using device-bound key pairs.

**Why passkeys over passwords?** Passwords are a shared secret between the user and the server. A server database breach exposes password hashes; a phishing site can capture a password. An asymmetric keypair has no shared secret — the private key never leaves the device and the server only ever sees the public key and a signature.

#### Key Generation — Registration (One Time per Device per Service)

1. The student's device generates a fresh **ECDSA P-256** keypair (ch2.2.3 p.85–87) for each service independently (learning platform, email, exam platform). Each service gets a **distinct keypair** — compromise of one service's key does not affect any other.
2. The **private key** is stored inside the device's secure hardware (laptop TPM or smartphone secure enclave). It is flagged as non-exportable — the raw private key bytes never leave the hardware under any circumstances.
3. The **public key** is registered with the university's identity server alongside the student's account.

**Why ECDSA P-256 over RSA-PSS for passkeys?** RSA-PSS (ch2.2.3 p.88–93) produces 256-byte signatures with a 2048-bit key. ECDSA P-256 produces 64-byte signatures at equivalent 128-bit security. On mobile devices performing authentication many times per day, the smaller signature and faster computation matter (ch2.2.3 p.85–87).

#### Authentication — Login

```
Step 1 — Student navigates to service (e.g. learning platform)
Step 2 — Server → Device:  { nonce, service_origin, timestamp }  (ch3.1 p.7)
Step 3 — Device:           ECDSA_sign(private_key, { nonce || service_origin || timestamp })
                            → signature (protected by device PIN/biometric before signing)
Step 4 — Device → Server:  { signature, public_key_id }
Step 5 — Server:           verifies signature against registered public key
                           verifies nonce is fresh (ch3.1 p.7)
                           if valid → access granted
```

All steps over **TLS 1.3** (ch3.6 p.5, p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).

### Part 3 — Phishing Resistance

The signature in Step 3 covers the **service_origin** (the domain of the relying party, e.g. `learning.university.be`). If a student is tricked into visiting a phishing site (`1earning.university.be`), the device signs the phishing domain. The real server receives a signature bound to the wrong origin and rejects it — the authentication fails. The private key never leaves the device, so even a fully "successful" login on the phishing site yields nothing useful to the attacker.

**Why passwords cannot achieve this**: a password is a static string the user types. They type it into any site that asks — the phishing site captures it immediately. A passkey signature is bound to the exact domain; typing it into another site is impossible.

### Part 4 — Shared Devices

This is the primary usability challenge. On a shared device, anyone who unlocks the OS could use stored passkeys.

**Solution**: require a **PIN or biometric unlock specific to the passkey** before the device will sign (separate from the OS login). The private key is only accessible after this user-level authentication. On a shared device, student A's passkey is inaccessible to student B as long as A's PIN remains secret — enforced at the secure hardware level.

Students should be advised not to register passkeys on shared or untrusted devices. For shared-device access, fall back to session-scoped TOTP (ch3.7 p.20, p.46) as a temporary second factor.

### Part 5 — Lost or Stolen Device

1. The student reports the lost device to the IT department immediately.
2. The university identity server revokes the public key registered from that device — analogous to certificate revocation in PKI (ch3.2 p.50–53).
3. Signatures from the revoked public key are rejected. The attacker holding the device cannot authenticate.
4. The student registers a new keypair from a replacement device.

**Why the private key cannot be extracted from a stolen device**: the key is stored as non-exportable in the device's secure hardware (TPM/enclave). Even with physical access, the private key bytes are inaccessible without the hardware defeating mechanisms. The window of risk is the time between loss and revocation reporting.

**Account recovery**: if all devices are lost, the student authenticates in person at the IT service desk with a government-issued ID, and a new keypair is registered.

### Part 6 — Comparison with Password-Based Login

| Property | Passwords | Passkeys |
|---|---|---|
| Phishing risk | High — user types password into any site | None — signature is origin-bound to exact domain |
| Server breach impact | Password hashes stolen → offline brute-force attack | Only public key stolen → useless without private key |
| Replay attack | Password reuse across sites is common | Nonce (ch3.1 p.7) ensures each signature is for one session only |
| Credential stuffing | High | Not applicable — no shared secret exists |
| Stolen database | Hash cracking reveals passwords | Nothing to crack — server has only public keys |

### Part 7 — Remaining Threats

- **Malware with OS-level access**: sufficiently privileged malware can trigger the signing operation silently while the user is logged into the OS — using the private key without the user's knowledge during an active session. EDR behaviour-based detection (ch3.7 p.43) on university-managed devices reduces this risk.
- **Shared device PIN observation**: a shoulder-surfer observing the passkey PIN on a shared device can later authenticate as that student. Physical environment awareness is required; passkeys on shared devices should be avoided.
- **Secure hardware absence**: older devices without TPM or secure enclave store keys in software. Software-stored keys are more vulnerable to extraction by privileged malware. The university should mandate hardware attestation as a registration requirement.

### Part 8 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Keypair algorithm | ECDSA P-256 per device per service | ch2.2.3 p.85–87 | 64-byte signature; 128-bit security; faster than RSA on mobile |
| Authentication | Asymmetric challenge-response with nonce | ch3.1 p.7 | Nonce prevents replay; private key never leaves device |
| Phishing protection | Signature covers service_origin | ch3.1 p.7 | Wrong-origin signatures rejected; passwords cannot achieve this |
| Key storage | Non-exportable in device TPM/secure enclave | ch3.7 p.46 | Key cannot be extracted even with physical device access |
| Revocation | Public key deregistration on server | ch3.2 p.50–53 | Analogous to CRL; lost device cannot authenticate after revocation |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Forward secrecy; authentication assertion confidential in transit |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.30, p.34)
- IS_UG_2_2_3_SecM_HashMac (p.85–87, p.88–93)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.28, p.50–53)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.20, p.43, p.46)

_Status: Complete_  
_Done by: William_
