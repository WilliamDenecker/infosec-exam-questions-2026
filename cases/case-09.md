# Case 9

A (wireless) Body Area Network (BAN) is a network of wirelessly communicating sensors embedded in wearable computing devices. The sensors typically collect health related data about the person wearing the BAN. These data are then transmitted through the BAN to a collecting device (i.c. a smartphone).

**Design a security solution for the collection of data by the BAN, transmission to the smartphone, and storage on the smartphone.**

**What are the most essential security services? What security mechanisms would you use to implement those services (be sufficiently specific)? What could be remaining vulnerabilities?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

*Note: sensors in a BAN are typically battery-operated, low-power, resource-constrained (i.e. limited memory, bandwidth, and computational capability) devices.*

## Answer

### Part 1 — Most Essential Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Confidentiality** | Yes — critical | ch1 p.15 | Health data (pulse, blood glucose, ECG) is highly sensitive. Wireless transmission is inherently broadcast — anyone in radio range can receive frames. | An attacker passively captures all health readings from a patient's sensors. Medical conditions are disclosed to unauthorised parties. |
| **Data integrity** | Yes — critical | ch1 p.34 | Medical readings that drive clinical decisions must not be tampered with or corrupted in transit. A modified blood glucose reading could trigger a dangerous insulin response. | An attacker injects a forged reading that causes the patient or clinician to take a harmful medical action. |
| **Authentication** | Yes — critical | ch1 p.22 | The smartphone must be certain that received data originates from the legitimate sensor on the wearer's body, not an attacker injecting fake readings from a nearby device. | An attacker in radio range injects fabricated sensor data. The phone cannot distinguish it from legitimate readings. |
| **Availability** | Yes | ch1 p.42 | The sensors must deliver data reliably. Jamming or battery depletion attacks that prevent sensor data from reaching the phone constitute a denial of service with potential health consequences. | Continuous glucose monitor data stops arriving; patient goes unmonitored during a dangerous hypoglycaemic episode. |

### Part 2 — Design Constraint: Symmetric Cryptography Only

Asymmetric algorithms (RSA, ECDSA, ECDH) require significant computation and power. The slides note explicitly that asymmetric cryptography is slow and resource-intensive (ch2.2.2 p.7 and p.13). Battery-operated BAN sensors with limited memory and computational capability cannot sustain public-key operations on every transmitted sample.

The design therefore uses **symmetric cryptography exclusively** for the sensor-to-smartphone link.

**Why AES-128 and not AES-256 for the sensor link?** AES-128 (ch2.2.1 p.55) provides 128-bit security — sufficient against any known classical attack. Choosing AES-128 over AES-256 conserves power (10 rounds vs. 14 rounds) and reduces computation time on the sensor microcontroller, at no practical security loss for this application. The smartphone storage tier uses AES-256 where resources are not constrained.

### Part 3 — Pre-Shared Key Distribution

Because the sensor cannot perform asymmetric key exchange, the symmetric key must be established out-of-band. During **device pairing** (performed once, offline, at initial setup):

1. The smartphone generates a unique 128-bit random key K for each sensor.
2. K is loaded onto the sensor over a short-range wired interface or a proximity-based wireless pairing process protected by physical proximity (the attacker is not present during pairing).
3. The smartphone stores K in its OS hardware-backed keystore, accessible only after user authentication.

Each sensor has a **unique K** — compromise of one sensor's key does not affect any other sensor.

### Part 4 — Sensor-to-Smartphone Protocol (Per Data Frame)

Each transmitted data frame is encrypted with **AES-128-GCM** (ch2.2.3 p.70–75). GCM is an AEAD mode: a single operation provides both confidentiality and integrity authentication. The 128-bit GCM authentication tag detects any modification, injection, or replay of frames. This satisfies confidentiality (ch1 p.15), integrity (ch1 p.34), and authentication (ch1 p.22) in one pass — critical for resource-constrained sensors.

```
plaintext:  { sensor_id, counter, health_data, timestamp }
nonce:      96-bit unique nonce per frame (counter-based)
output:     AES-128-GCM_encrypt(K, nonce, plaintext)  →  ciphertext + 128-bit GCM tag
```

The **counter** (monotonically increasing integer) embedded in the plaintext serves as both the nonce seed and a **replay protection** mechanism — the smartphone rejects any frame whose counter ≤ the last accepted counter from that sensor. This applies the freshness/nonce principle (ch3.1 p.7): old frames cannot be replayed.

The smartphone decrypts with K, verifies the GCM tag (invalid tag → drop frame), verifies the counter (replayed counter → drop frame), and stores the valid plaintext health data.

### Part 5 — Storage on Smartphone

The smartphone has significantly more resources than the sensors. Collected health data is stored encrypted with **AES-256-GCM** (ch2.2.3 p.70–75). The storage key is protected by the smartphone's hardware-backed keystore, accessible only after user authentication (PIN or biometric).

**Why AES-256 for storage vs AES-128 for the sensor link?** The smartphone is not resource-constrained (ch2.2.2 p.13). Data stored on the phone may persist for months or years; AES-256 retains 128-bit effective security against quantum adversaries (Grover's, ch2 PQCrypto p.16). AES-128 would drop to 64-bit effective security — obsolete.

Integrity of the full stored dataset is verified with **HMAC-SHA256** (ch2.2.3 p.63–66) computed over the complete health data archive. Before any export or clinical use, the HMAC is verified to confirm no tampering has occurred since last write.

If the smartphone uploads data to a cloud health service, **TLS 1.3** (ch3.6 p.5, p.7–8) with ECDHE (ch3.6 p.18) protects the upload channel with forward secrecy (ch3.6 p.37).

### Part 6 — Remaining Vulnerabilities

- **Physical compromise of a sensor**: an attacker who physically possesses a sensor can extract the pre-shared key K from its memory (via JTAG or hardware probing). Since K is static for the sensor's lifetime, all past and future data from that sensor is exposed. There is **no forward secrecy** with pre-shared symmetric keys — this is the fundamental trade-off made for resource constraints.
- **Battery depletion attack (DoS)**: an attacker floods the sensor with forged wireless frames. Decrypting and rejecting each (GCM tag fails) still consumes power. Sustained flooding drains the battery, preventing legitimate data collection (ch1 p.42). No cryptographic solution prevents this at the RF layer.
- **Pairing interception**: if the key loading during pairing is done wirelessly and the physical proximity guarantee is not strict, an attacker present during pairing could eavesdrop and recover K.
- **Smartphone compromise**: malware on the smartphone (ch3.7 p.38) can access decrypted health data after processing. EPP on the smartphone (ch3.7 p.43) reduces this risk for the storage tier.
- **No sensor key revocation**: once a sensor's key K is compromised, there is no revocation mechanism analogous to PKI CRL (ch3.2 p.50–53). The only remediation is physical sensor replacement and re-pairing with a new key.

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Sensor link encryption | AES-128-GCM per frame | ch2.2.1 p.55; ch2.2.3 p.70–75 | AEAD in one pass; power-efficient 10 rounds; asymmetric crypto infeasible on sensors |
| Replay protection | Counter in plaintext, rejected if ≤ last seen | ch3.1 p.7 | Stateless sender; no round-trip required; lightweight for constrained devices |
| Key establishment | Pre-shared via wired/proximity pairing | ch2.2.2 p.13 | No asymmetric operations on sensor; unique per-sensor key limits blast radius |
| Smartphone storage | AES-256-GCM + HMAC-SHA256 over full archive | ch2.2.3 p.70–75, p.63–66; ch2 PQCrypto p.16 | Quantum-safe at 256 bits; HMAC covers completeness beyond per-record GCM |
| Cloud upload | TLS 1.3, ECDHE, forward secrecy | ch3.6 p.5, p.7–8, p.18, p.37 | Forward secrecy; AEAD; application-independent |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.34, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.63–66, p.70–75)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.38–43)

_Status: Complete_  
_Done by: William_
