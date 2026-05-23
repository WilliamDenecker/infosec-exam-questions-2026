# Theme 4 — One-Way Transmitter Constraints: The Rolling Code Pattern

**Appears in**: Cases 25, 26, 30

---

## The Core Constraint

Some devices are **transmitter-only** by design. They broadcast; they never receive. This is not an oversight — it is a deliberate trade-off for simplicity, cost, battery life, or physical design. Examples from the cases:

- **Case 26 — car key fob**: battery-powered, one-directional RF transmitter; pressing a button sends a signal; the car never talks back
- **Case 25 — hardware TOTP token**: generates and displays codes; no wireless receiver; no USB receive channel
- **Case 30 — chip+PIN card (offline PIN verification)**: the PIN is verified by the card itself using its stored data; no outbound verification needed for the PIN check step

This constraint eliminates entire classes of authentication protocols. The design must solve authentication, integrity, and replay prevention **without any server-to-device communication**.

---

## Part 1 — What Becomes Impossible Without a Receive Channel

### Challenge-Response is Structurally Impossible (ch3.1 p.7)

The canonical replay-prevention mechanism is challenge-response: the server sends a fresh random nonce to the device; the device signs it and returns it; the server verifies.

```
Server → Device: { nonce }        ← requires the device to RECEIVE
Device → Server: { HMAC(K, nonce || data) }
```

Without a receive channel, there is no way to deliver the nonce to the device. Challenge-response is structurally impossible, regardless of how strong the cryptography is.

### Time-Based Code Generation is Problematic for Battery Devices

TOTP (ch3.7 p.20) requires a real-time clock (RTC) running continuously. For a hardware token or car fob:

**Battery drain**: an RTC running continuously consumes microamperes continuously. A fob stored in a drawer for 3 years while its owner is on holiday drains its battery. A TOTP fob requires a fresh battery to continue generating correct codes.

**Clock drift after battery replacement**: when the battery is replaced, the RTC resets to epoch or an incorrect time. The TOTP formula `T = floor(unix_timestamp / 30)` produces wrong codes. Re-synchronisation with the server requires a receive channel (to receive the current server time) — which the transmitter-only device does not have.

**Consequence for car fobs (case 26)**: time-based rolling code is rejected. Battery replacement at a service centre would require fob re-initialization. The fob must work correctly after battery replacement with no dealer visit.

**TOTP on smartphones (cases 25, 27, 35, 37)**: a smartphone is not constrained in the same way. It has network access (can sync clock via NTP), persistent battery, always-on RTC with drift correction. TOTP works correctly on smartphones. The problem is specific to constrained, isolated transmitter-only devices.

### Static Pre-Shared Code is Trivially Replayable

If the fob always sends the same code, an attacker records it once and replays it forever. This provides no replay protection whatsoever. Rejected immediately.

### Asymmetric Signature (ECDSA) Adds Cost Without Benefit

ECDSA can sign each transmission. But:
1. The car still needs replay prevention — a signed "unlock" command can be recorded and replayed; the signature is still valid. So a monotonic counter must still be included in the signed message.
2. The car can verify ECDSA using the fob's public key. But verification of ECDSA P-256 requires elliptic curve operations on the car's embedded processor — more complex but doable.
3. The fob **signing** with ECDSA requires the fob to perform elliptic curve scalar multiplication — much more expensive than HMAC-SHA256. On a battery-powered 8-bit microcontroller, this significantly increases power consumption per button press.
4. Non-repudiation is irrelevant for a car fob — there is no scenario where the car needs to prove to a third party that the fob commanded "unlock."

**Conclusion**: ECDSA adds computational cost on the constrained fob hardware without adding any security benefit over HMAC for this specific use case. HMAC-SHA256 is the correct choice.

---

## Part 2 — The Only Viable Solution: Counter-Based HMAC Rolling Code

Given all the eliminated alternatives, the only viable mechanism is:

1. **HMAC-SHA256** for authentication + integrity
2. **Monotonic counter in non-volatile memory** for replay prevention
3. **Lookahead window** for tolerating missed transmissions

### Complete Mechanism

**State stored in non-volatile memory on the fob**:
```
K_fob    : 128-bit symmetric key (provisioned at manufacturing; never changes)
C        : 32-bit counter (starts at 0; increments every button press; persists across power cycles)
```

**State stored in car (per fob)**:
```
fob_ID   : fob identifier
K_fob    : 128-bit symmetric key (same as fob; provisioned at manufacturing)
C_last   : counter value of last accepted message (stored in non-volatile flash)
```

**Fob button press**:
```
C += 1
command = encode(button_pressed)    // e.g., 0x01 = unlock, 0x02 = lock
MAC = HMAC-SHA256(K_fob, car_ID || fob_ID || C || command)
transmit: { car_ID, fob_ID, command, MAC }    // C is NOT transmitted (hidden counter — Option B)
save C to non-volatile memory
```

**Car verification**:
```
Receive: { car_ID, fob_ID, command, MAC }
Look up (K_fob, C_last) by fob_ID — if not found: reject (unknown fob)
For C_try = C_last+1 to C_last+16:
    if HMAC-SHA256(K_fob, car_ID || fob_ID || C_try || command) == MAC:
        set C_last = C_try; execute command; done
Reject (no matching counter found)
```

---

## Part 3 — Why the Counter Must Be in Non-Volatile Memory

The counter's entire purpose is to prevent replay. If the counter is reset to zero on every power cycle, an attacker can:

1. Record message with C=7
2. Wait for the car battery to be disconnected (repair shop)
3. After battery reconnection, C_last resets to 0
4. Replay message with C=7 — now 7 > 0 = C_last; accepted

The counter must persist across:
- Battery replacement in the fob
- Power loss in the car (battery disconnect for maintenance)
- Long storage periods (fob in a drawer for years)

**Implementation**: counters are stored in dedicated non-volatile flash memory (EEPROM) with guaranteed write-through. Modern microcontrollers include EEPROM cells rated for 100,000+ write cycles — far more than any realistic number of button presses per car's lifetime.

**What about car battery disconnect?**: when the car's battery is disconnected, C_last must be preserved in non-volatile flash on the car's body control module. Battery disconnect is a normal maintenance event; C_last must survive it. This is a specific design requirement for the car's embedded system.

---

## Part 4 — Per-Device Key Independence

Every fob has its own unique key K_fob. The car maintains a table:

```
Fob 1: { fob_ID_1, K_fob_1, C_last_1 }
Fob 2: { fob_ID_2, K_fob_2, C_last_2 }
Fob 3 (spare): { fob_ID_3, K_fob_3, C_last_3 }
```

**Why per-fob keys?** If all fobs shared one key K_shared:
- Compromising any one fob exposes K_shared → attacker can clone all fobs
- Counter synchronisation: Fob 1 used at C=50; Fob 2 at C=3. They share a counter? Impossible — they operate independently.

Per-fob keys mean:
- **Fob 1 lost**: car removes entry (fob_ID_1, K_fob_1) from the table. Fob 1 no longer works. Fobs 2 and 3 unaffected — different keys, different counters.
- **Fob 1 stolen and key extracted**: attacker can only replay Fob 1 commands. Fob 2 and 3 remain secure.
- **New fob provisioned**: dealer loads new (fob_ID_N, K_fob_N) pair into the car via physical access (OBD-II port with authentication).

---

## Part 5 — The Jam+Capture Attack and Why Hidden Counter Mitigates It

### The Attack

An attacker with signal jamming equipment:
1. Stands near the car with a jammer active
2. User presses the fob — signal jammed (car does not receive); user sees car does not unlock
3. User presses fob again — signal jammed again; attacker captures both transmissions
4. User gives up, walks away
5. Attacker turns off jammer; replays first captured transmission → car unlocks

This attack exploits the fact that the user pressed the fob twice. The attacker captured message at C=N and C=N+1. After replaying C=N, C_last advances to N. The attacker still holds C=N+1 (the second press) — still within the window — and can replay it later.

### Why Option A (Plaintext Counter) Gives More Information

With Option A, the attacker's captured messages show:
```
First capture:  { ..., C=47, command=unlock, MAC }
Second capture: { ..., C=48, command=unlock, MAC }
```

The attacker knows C_last is now 47 (after replaying first), and they have C=48 ready to use. They know they have exactly 1 remaining replay available before the window closes. They can use it immediately.

### Why Option B (Hidden Counter) Reduces Information

With Option B, the captured messages show:
```
First capture:  { ..., command=unlock, MAC_A }
Second capture: { ..., command=unlock, MAC_B }
```

The attacker does not know which counter value produced MAC_A or MAC_B. After replaying the first message (which the car accepts as some C_try in [C_last+1 ... C_last+16]), the attacker does not know how many replay opportunities remain. The attacker has MAC_B which may or may not still be within the window — they cannot calculate this without knowing K_fob (which they don't).

**Net effect**: Option B does not eliminate the jam+capture attack (it is still possible to get one replay), but it removes counter-state information from the attacker. The attacker cannot plan how many replay opportunities remain, and cannot predict when the window will be exhausted.

---

## Part 6 — Re-synchronisation After Window Exhaustion

If the fob's counter advances more than W = 16 presses beyond C_last (e.g., the user pressed the fob 20 times in a dead zone), the fob is out of sync:

```
C_fob = 70   (after 20 pocket presses)
C_last = 50  (last value the car accepted)
Car window: [51 ... 66]
C_fob = 70 is outside [51...66] → all subsequent messages rejected
```

**Recovery**: physical presence at a dealership. The dealer connects to the car's OBD-II port (authenticated physical access), reads the car's C_last, manually sets it back in sync with the fob's current counter. This re-initialization requires physical possession of both the car and the fob — the same physical access an attacker would need anyway to steal the car without the key.

**Design implication**: W = 16 is chosen specifically to make this recovery event rare (the user would need to press the fob 17 times accidentally without realising it) while limiting the attacker's replay window.

---

## Part 7 — Summary: The Rolling Code Pattern at a Glance

```
Device: transmitter-only, constrained, battery-powered
Goal:   authenticate command, prevent replay, survive power cycles

Solution:
  State:  (K_fob, C) in non-volatile memory on fob
          (K_fob, C_last) in non-volatile memory on verifier
  
  Transmit: { device_ID, command, HMAC(K, device_ID || C || command) }
            [C is NOT transmitted — hidden inside MAC only]
  
  Verify:   try C_try âˆˆ [C_last+1 ... C_last+W] until MAC matches
            accept first match; set C_last = C_try
  
  Properties:
    - No receive channel needed
    - No clock needed
    - Survives power cycles
    - Per-device unique key
    - Replay protection: structural (spent counter cannot match future window)
```

---

## Slide References

- IS_UG_2_2_3_SecM_HashMac (p.63–66: HMAC-SHA256)
- IS_UG_3_1_Appl_Basics (p.3: timestamps — rejected for battery fobs; p.7: challenge-response — rejected, no receive channel)
- IS_UG_3_7_Appl_System (p.20: TOTP — rejected for constrained fobs without reliable RTC; p.46: key provisioning at manufacturing)
