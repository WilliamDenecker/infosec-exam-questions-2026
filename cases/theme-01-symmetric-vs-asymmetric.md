# Theme 1 — Symmetric vs Asymmetric Cryptography: When to Use Each

**Appears in**: Cases 21, 24, 26, 27, 28, 30, 33, 34, 35, 36, 37

---

## The Core Question

The single most frequently recurring design decision in the course cases is: **should authentication or message integrity use a symmetric key (HMAC / AES-CBC-MAC) or an asymmetric key pair (ECDSA)?** The answer is not "asymmetric is always better". It depends on four factors:

1. Hardware constraints (battery power, CPU speed, memory)
2. Key distribution model (who holds the key, how many verifiers exist)
3. Non-repudiation requirement (must authorship be provable to a third party?)
4. Whether verification capability must NOT imply forgery capability

---

## The Fundamental Difference

| Property | Symmetric (HMAC / AES) | Asymmetric (ECDSA) |
|---|---|---|
| **Key structure** | One shared secret K | Private key (signs) + public key (verifies) |
| **Who can verify?** | Anyone who holds K | Anyone who holds the public key |
| **Who can forge?** | Anyone who holds K | Only the private key holder |
| **Key distribution risk** | Giving K to a verifier = giving them forgery capability | Public key can be freely distributed — no forgery risk |
| **Non-repudiation** | No — any K-holder could have produced the MAC | Yes — only the private key holder can produce the signature |
| **Computational cost** | Very low (few microseconds) | Higher (elliptic curve scalar multiplication) |
| **Output size** | 32 bytes (HMAC-SHA256) | 64 bytes (ECDSA P-256) |

---

## Part 1 — The Verification ≠ Forgery Argument (Critical)

This is the most important conceptual point in the course.

**With HMAC** (ch2.2.3 p.63–66):

```
MAC = HMAC(K, message)
```

Anyone who holds K can both verify a MAC (check that HMAC(K, message) == received_MAC) and produce a valid MAC for any message they choose. Verification capability **implies** forgery capability. This is a fundamental property of symmetric cryptography.

**Consequence**: if you distribute K to N verifiers, each of those N verifiers can forge. If any one of them is compromised, all future MACs are forgeable. There is no way to design around this — it is an intrinsic property of symmetric keys.

**With ECDSA** (ch2.2.3 p.85–87):

```
signature = ECDSA_sign(private_key, SHA-256(message))
ECDSA_verify(public_key, SHA-256(message), signature)  →  true / false
```

The public key can verify signatures but cannot produce them. Holding the public key tells an attacker nothing useful for forging — elliptic curve discrete logarithm is computationally infeasible (ch2.2.3 p.85–87). The public key can be published on a website, embedded in thousands of scanners, or distributed globally — zero risk.

**Practical rule**: if the verifier must not be able to forge, asymmetric cryptography (ECDSA) is mandatory. This is not a preference — HMAC is structurally incapable of providing this property.

---

## Part 2 — When to Use HMAC (Symmetric)

Use HMAC when **all** of the following hold:

1. **Hardware is constrained** — battery-powered microcontroller, passive RFID, fob with no asymmetric accelerator
2. **Single trusted verifier** — one car verifying its own fob, one cloud server verifying its own sensors
3. **No non-repudiation required** — the verifier is the only party who will ever check authorship; no audit trail needed for third parties
4. **Key can be distributed securely in advance** — physical pairing, manufacturing-time provisioning, QR code during setup

### Cases where HMAC is correct

**Case 26 (car key fob)**: the fob transmits, the car verifies. There is exactly one verifier (the car). The car already holds K_fob (provisioned at manufacturing). Non-repudiation is irrelevant — the car is not going to testify against its own fob. The fob is a 8-bit microcontroller with a tiny battery; ECDSA scalar multiplication would drain it in months of standby.

**Case 27 (smart plug ↔ cloud server)**: the plug sends encrypted sensor readings to the cloud; the cloud is the single verifier. Per-device keys derived from K_master. No third-party non-repudiation requirement for a plug turning on and off.

**Case 34 (smoke detector ↔ hub)**: local RF link; the hub is the single verifier; hardware is battery-powered and constrained.

**Case 36 (electronic stamp) — HMAC is WRONG here**: this is the key counter-example. Deutsche Post produces the stamp; thousands of postal scanners across Germany verify it. Each scanner would need K_DP. Each scanner with K_DP can forge stamps. One compromised scanner → unlimited free postage. HMAC is fundamentally wrong for this use case.

---

## Part 3 — When to Use ECDSA (Asymmetric)

Use ECDSA when **any** of the following hold:

1. **Multiple verifiers** who must verify without being able to forge (postal scanners, aircraft fleet verifying ground commands)
2. **Non-repudiation required** — individual authorship must be provable to a third party (academic grade submission, aviation command audit trail)
3. **Public distribution of verification key** — the public key can be freely embedded in devices, published online, or preloaded into firmware without creating any forgery risk
4. **Hardware is not severely constrained** — server, smartphone, mains-powered avionics, cloud infrastructure

### Cases where ECDSA is correct

**Case 36 (electronic stamp)**: Deutsche Post holds the ECDSA P-256 private key (offline, air-gapped). Every postal scanner in Germany — and internationally — holds only the public key. Public key = verify only. One compromised scanner → cannot forge a single stamp. Key compromise requires compromising Deutsche Post's offline signing system.

**Case 35 (group messaging — per-message sender attribution)**: within the group, each member must be identifiable as the authentic sender of their messages. If HMAC were used with a group MAC key, any member could produce a MAC appearing to come from any other member. ECDSA per-sender private key means: only Alice can produce Alice's signatures; all members verify against Alice's public key.

**Case 37 (aircraft remote commands)**: the ground station signs every command with its ECDSA P-256 private key. The aircraft verifies against the ground station's public key (embedded in aircraft firmware). Non-repudiation: the signed command packet is logged to the flight data recorder — if the aircraft crashes, the exact commands and their verified authorship are available for investigation.

---

## Part 4 — ECDSA P-256 vs RSA-PSS — Always ECDSA

When asymmetric signatures are needed, ECDSA P-256 is always preferred over RSA-PSS. The reason is signature size:

| Algorithm | Security Level | Signature Size | Key Size (public) |
|---|---|---|---|
| ECDSA P-256 | 128-bit (ch2.2.3 p.85–87) | **64 bytes** | 64 bytes |
| RSA-PSS 2048 | ~112-bit (ch2.2.3 p.88–93) | **256 bytes** | 256 bytes |
| RSA-PSS 3072 | 128-bit | **384 bytes** | 384 bytes |

ECDSA P-256 achieves higher security with 4× to 6× smaller signatures than equivalent RSA-PSS.

**Why size matters**:
- **Case 36 (2D barcode)**: the electronic stamp is a printed barcode with finite data capacity. 256 bytes of RSA-PSS signature + all stamp_data fields creates a high-density barcode that degrades with print quality. 64 bytes of ECDSA fits comfortably.
- **Case 37 (aviation datalink)**: satellite and VHF datalinks have limited bandwidth and strict message size limits. Every byte of signature reduces available payload for the actual command.
- **Case 35 (messaging)**: every group message carries a per-sender signature. Smaller signature = smaller message = less bandwidth per message per day per user.

RSA-PSS is never chosen in these cases. ECDSA P-256 is always the asymmetric signature selection.

---

## Part 5 — The Hybrid TLS Pattern

Many cases use both symmetric and asymmetric cryptography, reflecting how TLS itself works:

**Phase 1 — Session establishment (asymmetric)**:
- X.509 certificates authenticate both endpoints (ch3.2 p.28–29)
- ECDHE (ch2.2.4 p.10) derives a fresh ephemeral session key
- Asymmetric operations happen once per session

**Phase 2 — Bulk data transfer (symmetric)**:
- AES-256-GCM (ch2.2.3 p.70–75) encrypts all application data
- HMAC or GCM authentication tag protects integrity of every message
- Symmetric operations happen per message

**Why this is efficient**: elliptic curve scalar multiplication (ECDHE key exchange) is expensive relative to AES. If you did per-message asymmetric operations, the overhead would be unacceptable. By establishing a shared secret once via asymmetric cryptography and using it for symmetric bulk encryption, you get the security properties of both at a cost proportional to one handshake per session.

This pattern appears in cases 27, 33, 35, 37 (TLS 1.3 session establishment + AES-GCM data encryption).

---

## Part 6 — AES-CBC-MAC (the edge case for constrained RFID)

For passive RFID tags (ch2.2.3 p.61–62) that have AES hardware but no dedicated SHA hardware, AES-CBC-MAC is used instead of HMAC-SHA256. Both are symmetric MACs. AES-CBC-MAC uses the AES block cipher to compute a MAC:

```
MAC = AES-CBC-MAC(K, message)
```

The chip already has AES hardware for encryption; reusing AES for the MAC avoids needing a separate SHA computation unit. On hardware where SHA is not available but AES is, AES-CBC-MAC is the correct choice. On hardware where both are available, HMAC-SHA256 is generally preferred (stronger security properties against length-extension attacks).

---

## Part 7 — Complete Decision Flowchart

```
Does the system have multiple verifiers who receive the verification key?
│
├── YES → Can ALL verifiers be trusted unconditionally?
│         │
│         ├── NO  → Use ECDSA (verifiers get public key; cannot forge)
│         └── YES → Still prefer ECDSA (defence in depth)
│
└── NO → Single trusted verifier. Is non-repudiation required?
          │
          ├── YES → Use ECDSA (only signer can prove authorship)
          └── NO  → Is hardware severely constrained?
                    │
                    ├── YES → Use HMAC (AES-CBC-MAC if no SHA hardware)
                    └── NO  → Either works; ECDSA gives non-repudiation for free
```

---

## Part 8 — Summary Table: Which Mechanism, Which Case

| Case | Mechanism | Reason |
|---|---|---|
| 26 (car fob) | HMAC-SHA256 | Single verifier (car); constrained battery hardware; no non-repudiation |
| 27 (smart plug link) | HMAC-SHA256 | Single verifier (cloud); constrained embedded hardware |
| 28 (building access) | HMAC-SHA256 | Single verifier (door controller); per-card key |
| 30 (chip+PIN) | HMAC-SHA256 (offline PIN verify) | Card verifies own PIN hash; no external verifier needed |
| 33 (factory IoT) | Mutual TLS with X.509 (ECDSA) | Multiple verifiers; per-device certificates; revocability needed |
| 34 (smoke detector) | HMAC-SHA256 (plug ↔ hub) | Single hub verifier; battery hardware |
| 35 (group messaging) | ECDSA per sender | All group members are verifiers; sender attribution; non-repudiation within group |
| 36 (electronic stamp) | ECDSA P-256 | Thousands of scanners; scanners must not be able to forge |
| 37 (aircraft) | ECDSA P-256 per command | Non-repudiation for accident investigation; ground station identity must be unforgeable |

---

## Slide References

- IS_UG_2_2_2_SecM_AsymmEncr (p.7: RSA cost on constrained hardware; p.13: ECC vs RSA size/performance)
- IS_UG_2_2_3_SecM_HashMac (p.61–62: AES-CBC-MAC; p.63–66: HMAC; p.85–87: ECDSA P-256; p.88–93: RSA-PSS)
- IS_UG_2_2_4_SecM_KeyExch (p.10: ECDHE)
- IS_UG_3_2_Appl_AuthMeth (p.28–29: X.509 certificates)
