# Theme 7 — Key Management: Per-Device Keys, Distribution, and Revocation

**Appears in**: Cases 21, 26, 27, 28, 33, 35, 36, 37

---

## The Core Problem

When a system has N devices, should all devices share one symmetric key, or should each device have its own unique key?

**The answer is always: per-device keys.**

If all N devices share K_shared, then:
- Compromising any one device exposes K_shared
- K_shared is valid for authenticating all other devices in the system
- An attacker who extracts K_shared from one cheap consumer device can impersonate any device in the fleet
- Revoking one compromised device is impossible without rotating the key for every device simultaneously

Per-device keys mean:
- Compromising device i exposes only K_i
- All other devices remain secure
- Revoking device i requires only removing K_i from the verifier's table — no impact on any other device

---

## Part 1 — Three Key Management Architectures

### Architecture 1 — Master Key Derivation

**Used when**: the number of devices is very large (or unbounded), and a full key table would be impractical to manage. The server holds one K_master.

```
K_device = AES(K_master, device_ID)
```

Any device's key can be derived on demand from K_master and the device's ID. No lookup table needed — the server computes K_device when the device authenticates.

**Security properties**:
- K_master is stored in tamper-resistant hardware (ch3.7 p.46) on the server — if the server is compromised but the tamper-resistant module protects K_master, K_device values cannot be mass-computed
- Compromising one device exposes K_device but NOT K_master — the adversary cannot reverse AES to obtain K_master from K_device
- Compromising K_master is catastrophic — all device keys can be derived. K_master must never leave the tamper-resistant module.

**Revocation**:
- Add device_ID to a server-side revocation list
- When the server receives a message authenticated with K_device, check device_ID against the revocation list before accepting
- Cryptographically, K_device remains valid — the compromised device could still compute correct MACs. Revocation is enforced by server policy.

**Cases**: 27 (smart plug fleet), 28 (access card fleet)

**Weakness vs PKI**: revocation is application-level only. If the server's revocation-check logic is bypassed (bug, compromise), revoked devices can still authenticate. Certificate-based PKI revocation is cryptographic — it cannot be bypassed by the device.

---

### Architecture 2 — Pre-Provisioned Keys at Manufacturing

**Used when**: each device gets its unique key in a physically controlled factory environment. No server derivation needed — each device has an independently generated random key.

```
At manufacturing:
  Generate random K_i for device i
  Load K_i into device's non-volatile secure storage (one-time write, read-protected)
  Record (device_ID_i, K_i) in manufacturer's secure key database
  Ship device_i and database entry to customer/integrator
```

**Security properties**:
- K_i is generated randomly (not derived from a master); no master key exists whose compromise would be catastrophic
- K_i is loaded in a physically secure manufacturing environment — no network exposure during provisioning
- The secure database of (device_ID, K_i) pairs is an asset that must be physically protected

**Why physical provisioning (not network)?** Network-based enrollment has a chicken-and-egg problem: how does the server authenticate the device during enrollment if the device has no key yet? An attacker could connect a rogue device before the genuine device connects and claim the genuine device's key. Physical provisioning avoids this entirely — the key is in the device before it ever touches a network.

**Revocation**:
- Remove (device_ID_i, K_i) from the server's table
- Subsequent authentication attempts by device_i are rejected (server looks up device_ID_i → not found → reject)

**Cases**: 26 (car fob — unique K_fob per fob, loaded at manufacturing or dealer binding), 33 (factory IoT sensors — keys loaded at factory)

---

### Architecture 3 — X.509 PKI with Internal CA

**Used when**: individual cryptographic revocability is essential, or devices must authenticate to multiple servers (public key trust scales easily), or the computational resources for asymmetric crypto are available.

```
At manufacturing:
  Generate (private_key_i, public_key_i) ECDSA P-256 keypair on device i
  Device generates CSR (Certificate Signing Request): { device_ID, public_key_i }
  Internal CA signs the CSR → Certificate_i = { device_ID, public_key_i, CA_signature, validity }
  Load Certificate_i into device's secure storage
  Internal CA's public key is pre-loaded into all verifying parties (servers, other devices)
```

**At authentication (mutual TLS — ch3.6 p.7–8)**:
```
Device → Server: presents Certificate_i
Server verifies: CA_signature over { device_ID, public_key_i } → confirms this device is genuine
Server → Device: presents its own Certificate_server
Device verifies: CA_signature over server certificate → confirms it is talking to the real server
```

**Revocation via CRL** (ch3.2 p.50–53):
```
When device i is compromised:
  CA revokes Certificate_i → adds serial number to CRL
  CA signs the updated CRL with its own private key
  Verifying parties download CRL (periodically, or on demand)
  When verifying Certificate_i: check serial against CRL → reject if listed
```

CRL is CA-signed: an attacker cannot remove their compromised certificate from the CRL (they do not have the CA's private key to produce a valid CRL signature). Revocation is cryptographic and cannot be bypassed by the device.

**Why PKI over pre-shared keys for factory IoT (case 33)**:
- Factory IoT devices are mains-powered and have sufficient compute
- Devices must be individually revocable when maintenance replaces a sensor
- A compromised sensor's certificate can be revoked without touching any other device
- The CA distributes revocation information automatically — no manual key table update needed

**Cases**: 33 (factory IoT PKI), 37 (aircraft ↔ ground station with TLS certificates)

---

## Part 2 — Physical Pairing Pattern (Consumer Devices)

For consumer devices (smart plug, smoke detector), the symmetric key K is established through **physical proximity**:

```
Step 1 — User presses physical button on device (proves physical access to the device)
Step 2 — Device enters pairing mode; displays QR code encoding K (or generates K during pairing)
Step 3 — User scans QR code with app (K transferred from device to app over camera)
Step 4 — App sends K to cloud server over existing TLS session (K never travels unencrypted)
Step 5 — Server stores (device_ID, K) in database
Step 6 — Device is registered; pairing mode exits
```

**Security properties of this pairing**:
- K never travels over the internet unencrypted — it goes from device to phone via QR (physical channel), then phone to server via TLS
- Attacker must be physically present with the device to intercept K at the QR step
- No enrollment attack: attacker cannot pair a rogue device without physical access to the button

**Why QR code (not Bluetooth or NFC) for key transfer?** QR code is directional (camera must face the code) and short-range (must be close enough to photograph). It provides implicit physical access verification without requiring pairing protocols. The QR code method from the slides is the expected answer.

---

## Part 3 — Per-Counter Independence for Rolling-Code Systems (Case 26)

For fobs, keys and counters must both be per-fob:

```
Car state:
  Entry 1: { fob_ID_1, K_fob_1, C_last_1 }
  Entry 2: { fob_ID_2, K_fob_2, C_last_2 }
```

**Why per-counter?** Fobs operate independently. Fob 1 may be in the user's pocket (used 20 times today). Fob 2 may be in the spare key drawer (used 0 times). If they shared a counter:
- Shared counter after Fob 1's 20 uses: C = 20
- Fob 2 (spare key): stored counter = 0
- Fob 2 tries to unlock: sends C=1 (its stored value); car has C_last=20; 1 < 20 → rejected
- Every spare key in the world would desynchronize from shared counter usage by the primary key

Per-counter: Fob 1 has C=20; Fob 2 has C=0. Each is independent. Fob 2 with C=0 is compared against its own C_last_2=0 → C=1 is the next valid press.

**Revoking a fob**: delete the fob's entry from the car. Fob 1 lost → delete (fob_ID_1, K_fob_1, C_last_1). Fob 2 unaffected — different entry, different key, different counter.

---

## Part 4 — Key Storage at Rest on Servers

The cloud server's key database is a high-value target. If an attacker exfiltrates the database:

**Unprotected**: attacker has all (device_ID, K_device) pairs → can impersonate every device indefinitely.

**Protected**: keys stored encrypted at rest:

```
ciphertext = AES-256-GCM encrypt(K_database, K_device || device_ID, nonce)
```

K_database is separate from the key material and is protected by the server's hardware security module (HSM) or key management service. If the database is exfiltrated, the attacker has only ciphertext. K_database is not in the database — it is in the HSM.

**This is defence in depth for key management**: if the database is stolen (SQL injection, backup theft), the per-device keys are not immediately exposed. The attacker must also compromise the HSM.

---

## Part 5 — Key Rotation

**Symmetric key rotation** (cases 26, 27, 28, 34):
- K_device cannot be rotated without physically reaching the device (or re-pairing)
- For a car fob: dealer re-pairing with a new key requires physical possession of both fob and car
- For a smart plug: re-pairing (press button + scan new QR) resets K
- Rotation is infrequent; it requires user action; it is the user's responsibility

**Certificate rotation** (cases 33, 37):
- New certificate issued by CA when the old one approaches expiry
- Old certificate revoked via CRL when the new one is deployed
- For short-lived certificates (1 year), rotation is automatic and frequent
- No physical access to the device needed for revocation; physical access needed at new certificate provisioning (during scheduled maintenance)

**Why short certificate lifetimes are preferred**:
- A compromised private key has a bounded window of misuse (1 year maximum)
- Key rotation is forced by certificate expiry — even if no compromise occurs, keys are refreshed regularly
- Shorter lifetime = less damage if the key is compromised and undetected

---

## Part 6 — Key Compromise: Impact Analysis by Architecture

| Architecture | If one device key is compromised | If master key is compromised |
|---|---|---|
| Master key derivation | Attacker impersonates that one device | Attacker can derive keys for ALL devices |
| Pre-provisioned per-device | Attacker impersonates that one device; all others unaffected | N/A — no master key |
| X.509 PKI | Attacker uses private key until CRL revocation; then blocked | CA compromise = all certificates untrusted (catastrophic) |

**Protecting the CA private key** (case 37 analogy with Deutsche Post private key, case 36):
- Offline air-gapped storage
- Multi-person authorization required (key ceremony)
- Private key never touches an internet-connected system
- Signing operations done in batches in a physically secure facility

---

## Part 7 — Summary: Architecture Selection Guide

```
Step 1 — How many devices?
  < 100: pre-provisioned per-device keys (simple, manageable table)
  100–10,000: master key derivation (K_device = AES(K_master, device_ID))
  > 10,000 OR individual cryptographic revocability needed: X.509 PKI

Step 2 — What is the computational capability?
  Constrained (battery, 8-bit MCU): symmetric keys (HMAC)
  Full compute (mains-powered, server, smartphone): X.509 + TLS acceptable

Step 3 — How are keys distributed?
  Physical proximity: QR pairing (plug, detector)
  Manufacturing: factory provisioning (fob, sensor)
  CA issuance: PKI (factory IoT, aircraft)

Step 4 — How is revocation handled?
  Application-level blacklist: symmetric key architectures
  Cryptographic CRL: X.509 PKI (device cannot bypass CRL)
```

---

## Slide References

- IS_UG_2_2_1_SecM_SymmEncr (p.55: AES; key derivation)
- IS_UG_2_2_3_SecM_HashMac (p.63–66: HMAC with per-device keys)
- IS_UG_3_2_Appl_AuthMeth (p.28–29: X.509 certificates; p.50–53: CRL revocation)
- IS_UG_3_6_Appl_TLS (p.7–8: mutual TLS with certificate authentication)
- IS_UG_3_7_Appl_System (p.46: key storage in tamper-resistant hardware; physical provisioning)
