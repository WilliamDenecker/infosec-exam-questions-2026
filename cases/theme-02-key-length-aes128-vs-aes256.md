# Theme 2 — Key Length Selection: AES-128 vs AES-256 (and SHA-256 vs SHA-512)

**Appears in**: Cases 26, 27, 29, 33, 34, 35, 37

---

## The Core Question

When should AES-128 be used and when should AES-256 be used? The default "just use 256 everywhere" ignores a legitimate trade-off: AES-256 requires a more complex key schedule and is approximately 40% slower than AES-128 on hardware that lacks AES-256 acceleration. On constrained microcontrollers, this matters.

The correct answer depends on three factors:
1. **Hardware constraints** — constrained embedded vs unconstrained server/smartphone
2. **Data lifetime** — how long must the encrypted data remain confidential?
3. **System operational lifetime** — how long will this system be deployed?

---

## Part 1 — The Grover's Quantum Argument (Why the Choice Matters)

Grover's algorithm (ch2 PQCrypto p.16) provides a quadratic speedup for unstructured search, which translates directly to symmetric key attacks:

```
Classical brute-force of AES-128: 2^128 operations  →  infeasible
Quantum brute-force of AES-128:   2^64 operations   →  feasible with a sufficiently powerful quantum computer

Classical brute-force of AES-256: 2^256 operations  →  infeasible
Quantum brute-force of AES-256:   2^128 operations  →  infeasible (NIST minimum threshold)
```

**The question is not "do cryptographically relevant quantum computers exist today?" but "will they exist before the data or system becomes worthless?"**

If you deploy a system today that uses AES-128 to protect data with a 25-year confidentiality requirement, and a quantum computer capable of 2^64 operations emerges in year 15, your data is compromised retroactively — because an attacker who archived your ciphertext can now decrypt it.

---

## Part 2 — The Performance Trade-Off

AES-256 uses a 14-round key schedule vs AES-128's 10-round schedule. On hardware without AES acceleration:

- AES-256 is approximately **40% slower** than AES-128 (ch2.2.1 p.55)
- On a modern smartphone or server with AES-NI, this difference is negligible (both complete in nanoseconds per block)
- On a constrained 8-bit microcontroller (smart plug, smoke detector sensor node, battery fob), 40% slower means 40% more CPU time, 40% more battery drain for every encrypted block

**Rule**: on constrained hardware, AES-128 is acceptable when data lifetime is short (the session expires before quantum computers exist). On unconstrained hardware, or when data is long-lived, always AES-256.

---

## Part 3 — Decision Matrix Across Cases

| Case | Hardware | Data Lifetime | System Lifetime | Quantum Concern | Choice |
|---|---|---|---|---|---|
| 26 — car key fob | Constrained microcontroller, battery | Seconds (one command) | ~5 years (car ownership) | Low | **AES-128** |
| 27 — plug ↔ hub link | Constrained embedded, mains | Seconds (sensor reading) | ~10 years | Low | **AES-128** |
| 27 — phone ↔ cloud | Unconstrained smartphone + server | Years (retained logs/history) | ~10 years | Moderate | **AES-256** |
| 29 — cloud backup | Unconstrained server | **Decades** (backups kept for years) | Indefinite | **High** | **AES-256** |
| 33 — factory IoT | Mains-powered control system | Hours per session | 10–20 years | Moderate | **AES-256** |
| 34 — smoke detector ↔ hub | Constrained, battery | Seconds per alert | ~10 years | Low | **AES-128** |
| 35 — group messaging | Unconstrained smartphones + server | Years (stored message history) | Indefinite | High | **AES-256** |
| 37 — aircraft avionics | Unconstrained avionics computers | Decades (black box records) | **25–30 years** | **High** | **AES-256** |

---

## Part 4 — Detailed Justification: Aircraft (Case 37)

The aircraft case provides the clearest justification for AES-256.

An aircraft enters service today (2026). Typical operational lifetime: 25–30 years. The aircraft is retired around 2051–2056. The black box flight data recorder stores all communications — if an incident occurs in year 20 (2046), investigators will attempt to decrypt those communications.

If AES-128 is used:
- A quantum adversary in 2046 can perform 2^64 operations (Grover's speedup)
- 2^64 operations is within reach of a cryptographically relevant quantum computer (estimate: 2030–2040 range is speculative, but not impossible by 2046)
- All archived communications from the aircraft's entire operational history are exposed

If AES-256 is used:
- Even with Grover's speedup: 2^128 operations required
- 2^128 is the current NIST post-quantum security threshold — considered infeasible for the foreseeable future
- Aircraft communications remain confidential through retirement and beyond

**Conclusion**: the 25–30 year operational lifetime pushes the risk into the range where quantum computers might plausibly exist. AES-256 is the only defensible choice.

---

## Part 5 — The Same Argument Applies to Hash Functions

Grover's algorithm applies equally to hash preimage attacks:

```
SHA-256: 256-bit output → 128-bit preimage resistance classical → 64-bit post-quantum (Grover)
SHA-512: 512-bit output → 256-bit preimage resistance classical → 128-bit post-quantum (Grover)
```

The same lifetime analysis applies:

| Use case | Data lifetime | Quantum concern | Hash choice |
|---|---|---|---|
| TOTP code (ch3.7 p.20) | 30 seconds — code expires | None — code is worthless in 30s | HMAC-SHA256 specified |
| Password storage (ch3.2 p.11) | Years — password may remain unchanged | High — hash in database could be attacked years later | **SHA-512** |
| Key ratchet (case 35) | Potentially years (archived messages) | High | **SHA-512** |
| Firmware integrity (case 33, 37) | Lifetime of device | High | **SHA-512** as part of ECDSA |
| Electronic stamp serial (case 36) | Day of posting | Low — stamp expires same day | SHA-256 sufficient |

**Rule**: for password hashes, key derivation, and any hash that protects long-lived data, use SHA-512. For ephemeral data (TOTP codes, ephemeral session keys, short-lived stamps), SHA-256 is sufficient.

---

## Part 6 — Why Not Always Use SHA-512 and AES-256?

On constrained hardware, the cost is real:

**SHA-512 on constrained hardware**: SHA-512 processes 128-byte blocks vs SHA-256's 64-byte blocks. On a 32-bit microcontroller, SHA-512 uses 64-bit operations that must be emulated with two 32-bit operations. SHA-512 is slower than SHA-256 on 32-bit hardware — counterintuitively, SHA-256 may even be faster on 32-bit microcontrollers.

**AES-256 on 8-bit microcontrollers**: the AES-256 key schedule requires 240 bytes of expanded key storage vs 176 bytes for AES-128. On a device with 512 bytes of RAM, this matters.

**Battery lifetime**: a device running on two AA batteries that performs AES-256 instead of AES-128 will see measurably shorter battery life if it encrypts frequently (multiple times per second). A smoke detector sending one alarm message per alarm event does not encrypt frequently; a key fob encrypting every button press does.

The trade-off is real. Use the smallest key length that provides adequate security for the specific data lifetime and system lifetime.

---

## Part 7 — ECDHE Key Exchange: Shor's Algorithm Concern

For key exchange, the quantum concern is different. Shor's algorithm (ch2 PQCrypto p.12) breaks elliptic curve and RSA key exchange in polynomial time — far more devastating than Grover's quadratic speedup.

```
Classical ECDHE P-256 security: 128-bit
With Shor's quantum algorithm: broken
```

However, Shor's algorithm is orders of magnitude harder to implement than Grover's. A cryptographically relevant Shor's implementation requires far more qubits and error correction than Grover's. ECDHE P-256 with TLS 1.3 is still recommended for the current exam — the course does not require post-quantum key exchange (Ch3.6 covers TLS 1.3 with ECDHE as current best practice).

**For exam purposes**: ECDHE P-256 remains correct for key exchange. For symmetric encryption of data with a 25+ year lifetime, use AES-256.

---

## Part 8 — Complete Decision Rule (Memorisable Form)

```
Step 1 — Is hardware severely constrained (battery, 8-bit MCU)?
         YES → consider AES-128 / SHA-256
         NO  → use AES-256 / SHA-512

Step 2 — What is the data lifetime?
         < 1 hour → AES-128 / SHA-256 safe regardless of hardware
         1 hour – 10 years → consider system lifetime (Step 3)
         > 10 years → AES-256 / SHA-512 required

Step 3 — What is the system operational lifetime?
         < 5 years → quantum concern low; AES-128 acceptable
         5–15 years → quantum concern moderate; prefer AES-256
         > 15 years → quantum concern high; AES-256 mandatory

Step 4 — Password hashing and key derivation:
         Always SHA-512 (passwords are long-lived; database can be exfiltrated and attacked later)
```

---

## Slide References

- IS_UG_2_2_1_SecM_SymmEncr (p.55: AES key lengths, performance comparison)
- IS_UG_2_2_3_SecM_HashMac (p.24–32: SHA family, preimage resistance values)
- IS_UG_2_2_SecM-adv-PQCrypto (p.12: Shor's algorithm; p.16: Grover's algorithm, symmetric key halving)
- IS_UG_3_2_Appl_AuthMeth (p.11: password storage with SHA-512 + salt)
- IS_UG_3_7_Appl_System (p.20: TOTP using HMAC-SHA256)
