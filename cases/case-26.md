# Case 26

You may know these remote controls that allow, simply by pressing a button, to lock and unlock the doors of a car.

**What security mechanisms could be used to ensure proper and (reasonably) secure operation of these door openers? What threats are there to this security? What vulnerabilities might remain?**

*Note: When the door opener button is pressed sixteen times out of range, it will no longer work. In this case, a re-initialization of the system will be required from your local distributor.*

*Note: You shall assume that the door opener only operates as a transmitter, not as a receiver.*

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication / data-origin authentication** | Yes — critical | ch1 p.22 | The car must verify the unlock signal originates from the legitimate key fob, not a forger or replay attacker. | An attacker transmits a recorded signal and unlocks the car. Any radio transmitter can lock or unlock any car. |
| **Data integrity** | Yes | ch1 p.34 | The command (lock vs unlock) and the counter must not be alterable in transit. An attacker must not be able to change a captured "lock" into an "unlock". | A captured lock signal has one bit flipped and is retransmitted as an unlock — the car opens. |
| **Availability** | Yes | ch1 p.42 | The car must respond to legitimate presses within range. The 16-press synchronisation limit must tolerate real-world accidental presses without forcing a dealer visit. | The counter desynchronises after accidental pocket presses; the user cannot unlock their car. |

**Confidentiality is explicitly not required**: the command (lock or unlock) is not sensitive information. An observer who learns that a car was locked or unlocked gains nothing useful. The security goal is **unforgeability** (only the legitimate fob can produce a valid message) and **replay resistance** (a captured message cannot be reused later).

### Part 2 — Design Constraint: Transmitter-Only Fob

The key fob **only transmits — it cannot receive any data**. This is the most important constraint and eliminates entire classes of solutions.

#### Why Not Challenge-Response with a Server Nonce?

Challenge-response (ch3.1 p.7) requires the car to send a fresh nonce to the fob, which signs or MACs it and returns the response. **The fob cannot receive data.** The car has no channel back to the fob. Challenge-response is structurally impossible.

#### Why Not a Time-Based Code (as in case 25)?

A time-based code requires a reliable real-time clock (RTC) running continuously on the fob. This creates two problems:

1. **Battery dependency**: the RTC must run even when the fob is not in use. Key fob batteries last years in normal use; a continuously running RTC would drain the battery in months.
2. **Clock drift after battery change**: when the battery is replaced, the RTC resets to epoch 0 or a wrong time. The fob's time is now misaligned with the car's reference time — every code computed is wrong. Resynchronisation requires a dealer visit for every battery change.

A counter is persistent in non-volatile memory and requires no continuous power — correct after a battery change, correct after years of storage. **Rejected on reliability grounds**.

#### Why Not a Static Pre-Shared Code?

A fixed code transmitted on every press gives full replay protection: zero. An attacker with a radio receiver captures the code once and can unlock the car at any time later. **Trivially broken. Rejected.**

#### Why Not Asymmetric Signature (ECDSA)?

ECDSA (ch2.2.3 p.85–87) would provide strong unforgeability. The fob signs the message with its private key; the car verifies with the public key. Problems:

1. **No incoming channel**: the car still cannot send a nonce to the fob, so the signed message must contain a counter — the replay protection mechanism is identical to the symmetric solution.
2. **Computation cost**: ECDSA signature generation requires elliptic curve scalar multiplication — significantly more computation than HMAC on a small battery-powered chip.
3. **Key storage**: the private key must be stored securely in the fob hardware — same requirement as K in the symmetric solution.
4. **No additional security benefit over HMAC here**: HMAC with a 128-bit pre-shared key provides sufficient unforgeability for this use case. Asymmetric adds complexity and power cost with no practical benefit.

**Rejected**: more complex and slower than HMAC with no gain in the one-way transmitter scenario.

### Part 3 — Counter Transmission: Two Options

Each fob has a **unique 128-bit key K_fob** and its own **counter C** initialised at manufacturing or re-initialisation. The car stores a separate entry per registered fob: `(fob_ID, K_fob, C_last_fob)`.

The counter C is essential as an HMAC input — it binds the MAC to a specific counter position, making every button press produce a different MAC and preventing replay. Without C, every press of the same command produces the same HMAC (K_fob and command are fixed) — an attacker captures one message and replays it forever. **C must be in the HMAC input.**

The design question is: **should C also be transmitted in plaintext alongside the MAC, or kept hidden inside the HMAC only?** Two options exist.

---

#### Option A — C Transmitted in Plaintext

The fob transmits:

```
message = { car_ID, fob_ID, C, command, HMAC-SHA256(K_fob, car_ID || fob_ID || C || command) }
```

**Car verification (Option A)**:

```
Step 1 — Look up (K_fob, C_last_fob) by fob_ID.
         If fob_ID not registered → reject immediately.

Step 2 — Explicit replay check: is C > C_last_fob?
         If C â‰¤ C_last_fob → reject (replay or old message).
         If C > C_last_fob + 16 → reject (out of sync; re-init required).

Step 3 — Compute expected_MAC = HMAC-SHA256(K_fob, car_ID || fob_ID || C || command)
         If expected_MAC â‰  received_MAC → reject (forgery or tampering).

Step 4 — Valid. Execute command. Set C_last_fob = C.
```

**Advantage**: simple — one HMAC computation; the car knows exactly which counter value to use; the range check is explicit and easy to reason about.

**Disadvantage**: an eavesdropper who captures the message learns the fob's exact current counter value C. In the jam+capture attack (Part 6), knowing C tells the attacker precisely where the fob is in its counter sequence and exactly how many window slots remain — the attacker can plan the attack with full counter-state information.

---

#### Option B — C Hidden Inside HMAC Only (Chosen)

The fob transmits:

```
message = { car_ID, fob_ID, command, HMAC-SHA256(K_fob, car_ID || fob_ID || C || command) }
```

**C is included inside the HMAC computation but is NOT transmitted in plaintext.** The car reconstructs C internally by trying candidate values within its lookahead window (see Part 4 for full verification protocol).

**Advantage**: an eavesdropper cannot determine the fob's current counter value from the transmission. In the jam+capture attack, the attacker must proceed without knowing the counter position. Replay protection is also **automatic** — a replayed MAC was computed with some C â‰¤ C_last_fob; the car only tries C_last_fob+1 through C_last_fob+16; the replayed MAC cannot match any candidate. No separate `C > C_last_fob` range check is needed — it falls out structurally from the candidate search.

**Cost**: up to 16 HMAC computations instead of 1. On a car's microcontroller, each HMAC-SHA256 takes microseconds — 16 computations is completely negligible.

---

#### Comparison

| Property | Option A — C in plaintext | Option B — C hidden (chosen) |
|---|---|---|
| **Transmitted fields** | car_ID, fob_ID, **C**, command, MAC | car_ID, fob_ID, command, MAC |
| **Car computation per press** | 1 HMAC + explicit range check | Up to 16 HMAC attempts |
| **Counter state visible to attacker** | Yes — attacker learns exact C | No — attacker learns nothing |
| **Replay check** | Explicit `C > C_last_fob` required | Automatic — replay MAC never matches candidates |
| **Jam+capture attack info** | Attacker knows remaining window slots | Attacker has no counter-state information |

**Option B is chosen.** The computation cost (up to 16 Ã— microseconds) is negligible on the car side. In exchange, no counter-state information is leaked to an eavesdropper. Replay protection becomes a structural property of the candidate search rather than a separate explicit check. The design is cleaner and slightly more robust against the jam+capture attack at zero meaningful cost.

---

#### Why Multiple Fobs Require Separate Per-Fob State

If two fobs shared the same K and the same C_last on the car, they would be permanently out of sync:
- Fob 1 presses 50 times → its C = 50; car's C_last = 50
- Fob 2 was unused → its C = 3
- Fob 2 presses → MAC computed with C=3; car tries C_last+1=51 through C_last+16=66 → no match → Fob 2 never works again

**Fix**: each fob has its own `(fob_ID, K_fob, C_last_fob)` entry in the car. Fob 1's counter advancing has no effect on Fob 2's C_last_fob. The counters are completely independent.

**Why each fob needs its own unique K_fob (not a shared K)**: if all fobs share one K, the compromise of any one fob (e.g., it is stolen and extracted) exposes K for all fobs — the attacker can clone any fob or forge messages indefinitely. With per-fob keys, compromising Fob 1 exposes only K_fob1; K_fob2 remains safe. At the dealer, Fob 1's entry is deleted from the car without affecting Fob 2.

#### Why Each Input to HMAC Is Necessary

- **K_fob**: the secret unique to this fob. Only the holder of K_fob can produce a valid MAC. Without K_fob, an attacker cannot produce a valid message for any C or command.
- **car_ID**: binds the message to this specific car. A fob registered to car A cannot unlock car B, even if car B uses the same fob_ID.
- **fob_ID** (inside HMAC and in plaintext): in plaintext so the car knows which `(K_fob, C_last_fob)` entry to use; inside HMAC so an attacker cannot change the fob_ID field without invalidating the MAC (preventing fob impersonation).
- **C** (inside HMAC only): makes every press produce a different MAC; prevents replay.
- **command**: binds the MAC to lock vs unlock specifically. Without command in the MAC, an attacker who captures a valid "lock" message could change the command byte to "unlock" — the MAC would still verify. Including command means any modification invalidates the MAC.

#### Why HMAC-SHA256 and Not Alternatives

| Alternative | Why rejected |
|---|---|
| Plain `SHA-256(K \|\| car_ID \|\| C \|\| command)` | Length-extension attack (ch2.2.3 p.24–32): knowing this hash allows computing `SHA-256(K \|\| car_ID \|\| C \|\| command \|\| padding \|\| X)` for arbitrary X without knowing K. HMAC's structure prevents this. |
| AES-CBC-MAC(K, ...) | Cryptographically equivalent; uses AES hardware. Also valid. HMAC-SHA256 chosen because it is directly covered by slides (ch2.2.3 p.63–66) and SHA-256 is well-supported in embedded hardware. Either would work. |
| AES-GCM(K, ...) | GCM provides confidentiality + integrity. Confidentiality is not needed here — the command type is not sensitive. GCM is overkill; adds nonce management complexity. CBC-MAC or HMAC is sufficient. |
| Encrypt-only with AES-ECB | Encryption provides confidentiality, not integrity. An attacker can flip bits in the ciphertext and change the command without knowing K. Integrity is what's needed here, not secrecy. Rejected. |

#### Why 128-bit K and Not 256-bit?

For a car key fob, the adversarial model is an attacker who intercepts radio transmissions and attempts to forge or replay. 128-bit K means 2¹²⁸ brute-force attempts are needed — infeasible with current and foreseeable classical compute. 

Against Grover's quantum algorithm (ch2 PQCrypto p.16), 128-bit → 64-bit effective. **Is this a concern?** The value secured is a car; the threat lifetime is the car's ownership period (a few years at most); a quantum computer capable of attacking 64-bit HMAC does not currently exist and is not expected within this timeframe. For a physically portable device with battery and compute constraints, 128-bit is the practical choice. Long-lived sensitive data (cases 19, 29) justifies AES-256; a car key fob does not.

### Part 4 — Option B Verification: Car Tries Counter Candidates

*This section describes the full verification protocol for Option B (the chosen design — C hidden inside the HMAC).*

The car stores a separate entry `(K_fob, C_last_fob)` per registered fob. On receiving a message `{ car_ID, fob_ID, command, MAC }`:

```
Step 1 — Look up (K_fob, C_last_fob) by fob_ID.
         If fob_ID not registered → reject immediately.

Step 2 — For each candidate C_try in { C_last_fob+1, C_last_fob+2, ..., C_last_fob+16 }:
             candidate_MAC = HMAC-SHA256(K_fob, car_ID || fob_ID || C_try || command)
             if candidate_MAC == received_MAC:
                 → valid; execute command; set C_last_fob = C_try; done

Step 3 — If no candidate matched → reject.
         (Either forgery — attacker lacks K_fob —
          or replay — captured MAC was computed with C â‰¤ C_last_fob, outside the window.)
```

**Why this design is superior to transmitting C:**

- **Replay protection is automatic**: a replayed message was computed with some C â‰¤ C_last. The car tries only C_last+1 through C_last+16 — the replayed MAC cannot match any of these. No explicit "C > C_last" check is needed; it falls out naturally.
- **Attacker learns nothing about counter state**: without plaintext C in the transmission, an eavesdropper cannot determine the fob's current counter position. This modestly increases the difficulty of the jam+capture attack.
- **Cost**: up to 16 HMAC computations per button press, each taking microseconds on a microcontroller — completely negligible.

A captured message is permanently spent once the car accepts it — `C_last` advances and the spent MAC cannot match any future candidate window.

### Part 5 — The 16-Press Out-of-Range Window

If the fob is pressed W times out of range (the fob increments C each time, but the car receives nothing), the fob's counter has advanced to `C_last + W`. The car must accept any counter value in `[C_last + 1, C_last + 16]`.

**Why 16 specifically?** This is given by the question. The tradeoff is:
- **Too small (e.g., 3)**: pressing the fob 4 times in a pocket — common scenario — permanently desynchronises the system. User cannot unlock; dealer visit required.
- **Too large (e.g., 1000)**: an attacker who intercepts 1000 consecutive valid messages in sequence can replay them all in the future before the window closes. A large window weakens replay protection.
- **16**: a reasonable balance. Normal usage rarely causes 16 out-of-range presses. The captured-future-message attack requires an attacker to collect 17+ consecutive valid messages — difficult in practice.

If more than 16 presses occur out of range, the fob's counter exceeds the window. Re-initialisation at the distributor resets both fob and car to the same counter value (requires physical possession of both, providing implicit authentication of the ownership).

### Part 6 — Threats and Remaining Vulnerabilities

| Threat | Attacker action | Why it fails |
|---|---|---|
| **Replay attack** | Captures message with counter N; retransmits later | C â‰¤ C_last after first acceptance → rejected at step 2 |
| **Forgery** | Constructs message with fabricated HMAC | Requires K; 2¹²⁸ brute-force attempts needed without K |
| **Command modification** | Flips lock↓unlock bit in captured message | Command is in HMAC input; modification invalidates the tag |
| **Brute-force K** | Tries all possible 128-bit keys | 2¹²⁸ keys; infeasible with classical compute |
| **Eavesdropping** | Records { car_ID, C, command, MAC } | Cannot produce valid MAC for C+1 without K; data is useless |

**Relay attack (residual vulnerability, cannot be cryptographically prevented)**: two radio devices extend the fob's effective range. Device 1 (near victim's pocket/bag) captures the fob's RF signal; Device 2 (near the car) retransmits it at correct power. The car receives a message with the correct K, a valid counter C it has not seen before, and a valid HMAC — because the message is genuine; it was produced by the real fob. The HMAC is correct, the counter is fresh, everything checks out. **No cryptographic mechanism can detect this** — the message is not a forgery and not a replay. Distance bounding (measuring round-trip time to verify physical proximity) is the countermeasure, but it requires the fob to be a receiver — which contradicts the problem's constraint. This is a fundamental limitation of the one-way transmitter design.

**Signal jamming + selective capture (residual vulnerability)**: an attacker jams the car's receiver on press N (car stays at `C_last = N-1`, never receiving the message) while recording the transmitted message. The fob's counter advances to N. Later, the attacker stops jamming and replays the recorded message with counter N — the car is still at `C_last = N-1`, so N is fresh and valid; the car unlocks. This is a one-shot attack (the captured message is spent immediately). Against a rolling-code system, this attack is feasible with commodity RF hardware. The window is narrow — the attacker must jam and capture simultaneously — but it is a real residual vulnerability.

**Key extraction from fob hardware (residual vulnerability)**: physical access to the fob enables extraction of K using semiconductor probing or fault injection. With K and any valid counter value, the attacker can produce valid messages for all future counter values — a perfect clone. Tamper-evident chip packaging and active chip destruction on physical intrusion attempt are hardware mitigations outside the scope of cryptographic design.

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Authentication mechanism | Counter-based rolling code | ch3.1 p.7 (why not nonce); ch3.1 p.3 (why not timestamp) | Only option for transmitter-only fob; challenge-response impossible; time-based requires RTC (clock drift on battery change) |
| MAC algorithm | HMAC-SHA256(K, car_ID \|\| C \|\| command) — Option B chosen: C inside HMAC only, not in plaintext (vs Option A: C also transmitted in plaintext) | ch2.2.3 p.63–66 | No length extension; all inputs bound by MAC; Option B hides counter state from eavesdropper; replay protection automatic via candidate search |
| Why not ECDSA | Asymmetric adds cost with no benefit | ch2.2.2 p.7, p.13 | ECDSA slower on constrained hardware; non-repudiation not needed; pre-shared symmetric K achieves same authentication goal |
| Why not encrypt-only | Confidentiality is not the goal | ch1 p.22, p.34 | Need integrity and authenticity, not secrecy of command; HMAC provides this; encryption alone does not |
| Key size | 128-bit K | ch2.2.1 p.55 | Sufficient against classical attacks; Grover's threat minimal for short-lived car key use case; 256-bit unnecessary for constrained fob hardware |
| Replay protection | Car tries C_last+1…C_last+16; no explicit C check needed | ch1 p.34 | Replay MAC cannot match any candidate in window; attacker learns no counter-state information; automatic without separate bounds check |
| Synchronisation window | Lookahead of 16 presses | ch1 p.42 | Tolerates pocket presses without dealer visit; limits captured-future-message window |

### Sources

- IS_UG_1_Introduction (p.10, p.22, p.34, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.61–62, p.63–66, p.85–87)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.3, p.7)

_Status: Complete_  
_Done by: William_
