# Theme 3 — Replay Prevention: Counter vs Timestamp vs Both

**Appears in**: Cases 25, 26, 27, 28, 30, 33, 34, 37

---

## The Core Problem

An attacker who intercepts a valid authenticated message can replay it:

- **Case 26**: capture "unlock" command from car fob; replay it to unlock the car without the key
- **Case 34**: capture "silence alarm" command; replay it to silence the alarm every time it sounds
- **Case 37**: capture a valid "decrease altitude" command from a previous flight; replay it during a new flight
- **Case 27**: capture a "turn on plug at 100%" command; replay it after the user has turned the plug off

Authentication (HMAC or ECDSA) proves that a message was **once** valid. It cannot, by itself, prove that the message has not been replayed. Replay prevention is a separate mechanism layered on top of authentication.

---

## Part 1 — Mechanism 1: Monotonic Counter

The sender maintains a counter that increments with every message. The receiver maintains `C_last` (the counter value from the last accepted message).

**Sender**:
```
C += 1
message = { ..., C, HMAC(K, ... || C || ...) }    // or ECDSA covering C
```

**Receiver**:
```
Extract C from received message
Verify MAC/signature (covers C)
If C ≤ C_last → reject (replay or out-of-order)
If C > C_last + W → reject (gap too large; possible desync)
If C_last < C ≤ C_last + W → accept; set C_last = C
```

W is a lookahead window that tolerates occasional out-of-range transmissions (e.g., the user pressed the fob in their pocket 5 times without the car in range). Without a window, a single missed message permanently desynchronises the system.

### Properties of Counter-Based Replay Prevention

**Advantages**:
- Independent of time — works even with no clock, even with clock drift, even after power cycles (if C_last is stored in non-volatile memory)
- Automatic exhaustion — once accepted, the counter advances; the same message cannot be replayed (within-window replay impossible because the window has moved past C)
- Suitable for constrained devices with no RTC (car fob, passive RFID, hardware token)

**Disadvantages**:
- Requires persistent non-volatile storage of C_last on the receiver (must survive power loss)
- Counter state must be securely maintained — if C_last is corrupted or reset to zero after power failure, old messages can be replayed
- Does not provide absolute freshness — a captured message from 6 months ago is still accepted if it falls within the window
- Counter exhaustion: if C wraps around (e.g., 32-bit counter runs out after 2^32 presses), the system must be securely re-initialized

### When to Use Counter-Based Replay Prevention

- Transmitter-only devices (car fob, hardware OTP token) that cannot receive a challenge
- Any device without a reliable real-time clock
- When message order matters (sequence integrity required)
- Cases 25, 26, 28, 37 (sequence counters)

---

## Part 2 — Mechanism 2: Timestamp

Each message includes the sender's current wall-clock time. The receiver rejects any message with a timestamp outside an acceptable window W (e.g., ±5 seconds, ±3 days).

**Sender**:
```
t = current_unix_timestamp()
message = { ..., t, HMAC(K, ... || t || ...) }    // or ECDSA covering t
```

**Receiver**:
```
Extract t from received message
Verify MAC/signature (covers t)
If |current_time - t| > W → reject (expired or future-dated)
If t ≤ last_accepted_t → reject (replay of used timestamp)
Accept; store t as last_accepted_t (optional — within-window replay prevention)
```

W is application-specific:
- Aircraft commands (case 37): W = ±5 seconds (commands older than 5 seconds are stale)
- Electronic stamps (case 36): W = ±3 days (a stamp is valid for the posted date ±3 days)
- Smart plug commands (case 27): W = ±30 seconds (reasonable for cloud-mediated control)

### Properties of Timestamp-Based Replay Prevention

**Advantages**:
- Absolute freshness bound — a captured message expires in W seconds regardless of sequence state
- No persistent state required on receiver beyond access to a clock
- Natural expiration that fits real-world use cases (stamps expire the day after posting)
- Easy to reason about: "this message is at most W seconds old"

**Disadvantages**:
- Requires synchronised clocks between sender and receiver
  - If sender clock drifts, messages may be incorrectly rejected
  - If receiver clock is manipulated (GPS spoofing, RTC tampering), replays succeed
- Does not prevent replay within the time window W — if an attacker captures a message and replays it within W seconds, and the receiver does not track used timestamps, the replay succeeds
- Clock reset after battery change (car fob, smoke detector) may produce incorrect timestamps

### When to Use Timestamp-Based Replay Prevention

- When absolute freshness guarantee is needed (aircraft commands, financial transactions)
- When the receiver has a reliable, tamper-resistant clock
- When the data has a natural expiration that maps to real time (stamps, tokens, session cookies)
- Cases 36 (stamp date), 37 (command timestamp), 27 (cloud-mediated commands with server-side timestamp verification)

---

## Part 3 — Defence in Depth: Counter + Timestamp Together

In high-security systems, both mechanisms are used simultaneously. They cover different failure modes:

**Counter covers**: within-session replay — the same message cannot be accepted twice in the same session because the counter has advanced.

**Timestamp covers**: cross-session replay — if the session counter was reset (power failure, re-initialization), a captured message from before the reset might have a counter within the new window. The timestamp prevents this: the captured message from 3 weeks ago has an expired timestamp.

```
Receiver logic (counter + timestamp):
  Extract (C, t) from message; verify signature/MAC covering both
  If C ≤ C_last → reject (counter replay)
  If |current_time - t| > W → reject (timestamp expired)
  If C > C_last + max_window → reject (counter gap too large)
  Accept; update C_last = C
```

**Cases using both**:
- Case 37 (aircraft): per-command sequence counter + timestamp with ±5 second window
- Case 27 (cloud commands): sequence counter on cloud-to-plug messages + server timestamp
- Case 33 (factory IoT): TLS sequence numbers + server-side timestamp validation

---

## Part 4 — Hidden Counter vs Plaintext Counter (Case 26 Detail)

This is a refinement specific to one-way transmitter systems (case 26, 30). The question is: should the counter value C appear in the transmitted message, or only inside the MAC computation?

### Option A — Counter Transmitted in Plaintext

```
message = { car_ID, fob_ID, C, command, HMAC(K, car_ID || fob_ID || C || command) }
```

**Verification**:
```
Step 1: Look up (K_fob, C_last) by fob_ID
Step 2: Check C > C_last (explicit range check)
Step 3: Verify HMAC
Step 4: Accept; set C_last = C
```

**Advantage**: simple; one MAC computation; explicit range check is straightforward.

**Disadvantage**: eavesdropper learns exact C. If the attacker jams the signal (preventing car from receiving) while recording it, they know exactly what counter value was used and can infer how many presses remain before the window closes.

### Option B — Counter Hidden Inside MAC Only (Chosen)

```
message = { car_ID, fob_ID, command, HMAC(K, car_ID || fob_ID || C || command) }
```

Note: C is used in computing the MAC but is **not included** in the transmitted message.

**Verification**:
```
Step 1: Look up (K_fob, C_last) by fob_ID
Step 2: For each C_try in { C_last+1, C_last+2, ..., C_last+W }:
         if HMAC(K, car_ID || fob_ID || C_try || command) == received_MAC → accept
Step 3: No match → reject (invalid MAC for all candidates)
```

**Advantage**: eavesdropper learns nothing about counter state. No explicit counter in the message means no information about how many presses have been used in the current window. Replay protection is also structural: a MAC computed with C ≤ C_last cannot match any candidate in [C_last+1 ... C_last+W].

**Cost**: up to W MAC computations (W ≈ 16). Each HMAC-SHA256 takes ~microseconds. 16 × microseconds = still microseconds. Negligible.

**Why negligible cost matters**: the only argument against hidden counter is "it requires more computation." At 16 HMAC verifications vs 1, the additional time is on the order of 15 microseconds on a car's embedded processor. The gain in information security (no counter-state leak) is not negligible. Option B is always chosen when the receiver can afford the candidate search (i.e., when W is small and MAC is fast).

---

## Part 5 — Window Size Design

The lookahead window W balances two competing concerns:

**Too small a window (W = 1 or 2)**:
- The user presses the fob twice in a pocket (out of range) → counter advances by 2 → first in-range press has C = C_last+3, outside window → rejected
- Requires dealer re-initialization for normal pocket key presses

**Too large a window (W = 100)**:
- Attacker with a jammer can block 99 valid transmissions while recording them all
- Replays all 99 recorded messages one by one after the user gives up
- At W=100, attacker has 99 usable replay attempts

**Typical practice (case 26)**: W = 16. This tolerates 15 accidental pocket presses (very unlikely to press 15 times without noticing) while limiting the attacker's replay stockpile to 16 messages (not enough for a practical attack if the genuine user will press again).

For aircraft (case 37): W = 1 (no tolerance). Every command in sequence. Out-of-order or missed command → operator must re-authenticate. Life-safety commands must arrive in strict sequence.

---

## Part 6 — Nonce-Based Challenge-Response (When a Back Channel Exists)

When the device can receive as well as transmit, challenge-response (ch3.1 p.7) replaces counter-based replay prevention:

```
Server → Device: { nonce }           // fresh random nonce per session
Device → Server: { HMAC(K, nonce || data) }
Server checks: HMAC(K, nonce || data) == received_MAC
```

The nonce is fresh and random each session. A replay of a previous session's message includes the old nonce — the server rejects because current_nonce ≠ old_nonce.

**Why this is better when available**: no state maintenance (no C_last), no window, no desynchronisation risk. The nonce is entirely fresh and random each time. However, it requires the device to receive the nonce — which is impossible for transmitter-only devices.

**Cases where challenge-response is used**: 27 (server ↔ phone login), 33 (server ↔ control workstation), 37 (initial session authentication before per-command counter takes over).

---

## Part 7 — Summary: Mechanism Selection

| Scenario | Back channel? | Clock reliable? | Data lifetime | Chosen mechanism |
|---|---|---|---|---|
| Car fob (case 26) | No | No (battery fob) | Seconds | Counter (hidden) |
| Hardware OTP token (case 25) | No | No (battery) | 30 seconds | Counter |
| Electronic stamp (case 36) | N/A (offline barcode) | Yes (date is stamp date) | 1–3 days | Timestamp (date field) |
| Aircraft command (case 37) | Yes | Yes (atomic clock sync) | Seconds | Counter + Timestamp |
| Smart plug command (case 27) | Yes | Yes (server clock) | Minutes | Timestamp + server nonce |
| Factory IoT sensor (case 33) | Yes (TLS session) | Yes | Session | TLS sequence numbers |
| TOTP (cases 25, 27, 34, 37) | Yes (phone inputs code) | Phone clock | 30 seconds | Timestamp (T = floor(unix/30)) |

---

## Slide References

- IS_UG_3_1_Appl_Basics (p.3: timestamps for freshness; p.7: challenge-response / nonce / counter)
- IS_UG_2_2_3_SecM_HashMac (p.63–66: HMAC used in MAC computation)
- IS_UG_3_6_Appl_TLS (p.37: TLS sequence numbers / replay protection)
