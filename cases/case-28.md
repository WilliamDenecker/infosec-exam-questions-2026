# Case 28

**What security services will be needed to achieve a contactless rechargeable card for public transportation? What security mechanisms and protocol would you use to implement these security services?**

These rechargeable card allow the customer to top up their credit (using the ticket machines of the transportation company). This credit can then be used to pay for the use of public transportation. You may consider the simplified scenario where the card permits a certain number of rides at fixed rate.

The card is practically used as a parking badge: you swipe the card in front a contactless card reader, which will decrease the credit on your card by the cost of the ride and will validate your ride. This action must be sufficiently fast and the computing power of a smart card is limited. Examples of such rechargeable cards are the "Mobib" of the MIVB/De Lijn/TEC (Belgium), the "Navigo" of the RATP (Paris), the "Oyster" card of TfL (London), or the "OV Chipkaart" (The Netherlands).

*Note: I expect you to make a choice and defend this choice. Don't present a range of possible solutions. Be sufficiently specific in your implementation (algorithms, key lengths, modes, etc.).*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication / data-origin authentication** | Yes — critical | ch1 p.22 | The reader must verify the card was issued by the transport authority — not forged, cloned, or emulated by a passenger. | A passenger produces a counterfeit card chip with arbitrary ride counts. The transport operator loses revenue systemically. |
| **Data integrity** | Yes — critical | ch1 p.34 | The ride count stored on the card must not be modifiable by the holder — fraudulently adding credit, or preventing decrements. | A passenger snapshots their card's memory and restores it after each journey, paying only once for unlimited travel. |
| **Access control / authorisation** | Yes | ch1 p.30 | Only authorised transport authority terminals may decrement credit or top up. A rogue reader must not be able to drain credit or issue free top-ups. | A malicious device swipes cards and drains all credit from unsuspecting passengers. A criminal terminal issues unlimited free top-ups. |
| **Availability** | Yes — critical | ch1 p.42 | Validation must complete in under ~200 ms — passengers walk through a gate without stopping. The system must function even when readers are offline. | Passengers queue at gate readers; rush-hour transit is paralysed. A network outage prevents all validation. |

### Part 2 — Critical Design Constraint: Passive RFID Card

A contactless transit card is typically a passive RFID chip — it has **no battery**. It receives its operating power from the electromagnetic field emitted by the reader. This has profound cryptographic implications:

- **Computation time is extremely limited**: the card has power only while in the reader's field; the interaction must complete in milliseconds.
- **Only AES hardware acceleration is reliably available**: a passive RFID chip can implement AES in hardware (small gate count, very fast). SHA-256 hash functions require significantly more gates — may not be available on all chips.
- **Asymmetric cryptography (RSA, ECDSA) is categorically impossible**: RSA requires modular exponentiation (millions of multiplications); ECDSA requires elliptic curve point multiplication. Both far exceed the computation budget of a passive RFID chip (ch2.2.2 p.7, p.13).

**This eliminates asymmetric solutions entirely.** Only symmetric cryptography is viable.

### Part 3 — Key Design Difference from Case 20 (Passive Access Badge)

Case 20 is a read-only access badge — the reader reads the badge and makes a local access decision. **No data is written back to the badge.** In case 28, the reader must **write** a decremented ride count back to the card after a successful validation. This creates an additional attack vector not present in case 20:

**Rollback attack**: a passenger copies their card's EEPROM contents (high ride count) before travelling. After travel (ride count decremented), they physically restore the EEPROM to the copied snapshot. The card now shows the pre-travel ride count again — a net gain of one free ride. Cryptographic authentication of the card data is necessary but not sufficient; the system must also detect and reject rolled-back state.

### Part 4 — MAC Function Choice: Why AES-CBC-MAC and Not HMAC

The card data must be authenticated — the reader must verify that the ride count, counter, and expiry have not been tampered with since the transport authority last wrote them.

**Option A — HMAC-SHA256** (ch2.2.3 p.63–66): HMAC uses SHA-256 as its compression function. SHA-256 requires a hardware implementation of the SHA-256 compression function — a distinct hardware block from AES. Many passive RFID card chips implement AES hardware acceleration but do **not** implement a SHA-256 hash core, because AES uses fewer gates. If the chip does not have SHA hardware, HMAC is not available.

**Option B — AES-CBC-MAC** (ch2.2.3 p.61–62): CBC-MAC uses only AES block cipher operations — the same hardware already on the chip for AES. The message is processed in CBC mode and the final ciphertext block is the MAC. No separate hash hardware is required. On a chip that has AES but not SHA, AES-CBC-MAC is the only viable option.

**Option C — AES-GCM tag**: GCM uses GHASH — polynomial multiplication in GF(2¹²⁸). This requires dedicated multiplication hardware in addition to AES hardware. More complex than CBC-MAC for an integrity-only purpose on constrained hardware. Rejected.

**Chosen: AES-CBC-MAC** — uses only AES operations guaranteed to be available on AES-hardware RFID chips. HMAC is rejected because it requires SHA hardware that may not be present on the chip.

### Part 5 — Key Structure: Per-Card Derived Keys

**Option A — Single shared key K for all cards:** All readers and all cards use the same K. One compromised card's key reveals K for every card in the system — a systemic failure affecting millions of passengers. **Rejected**: catastrophic scope of compromise.

**Option B — Independent random key K_card per card:** Each card has a unique random key. The transport authority must store and distribute a database of (card_ID → K_card) to every reader. For millions of cards and thousands of readers, this is operationally complex and creates large key distribution attack surfaces. **Partially acceptable but unwieldy.**

**Option C — Per-card key derived from a master key (chosen):**

```
K_card = AES(K_master, card_ID)
```

K_master is a single 128-bit key held in tamper-resistant hardware on authorised terminals and the central key management server. Given any card_ID, any authorised reader derives K_card in microseconds — no database lookup required. K_master never leaves authorised infrastructure.

**Why this is better than Option B**: no per-card key database to distribute; instant key derivation; compromise of one card's K_card does not expose K_master (AES is a pseudorandom permutation — knowing K_card and card_ID does not allow recovery of K_master without breaking AES). Compromise of K_master would be catastrophic, so it is stored exclusively in tamper-resistant hardware with active zeroisation on intrusion.

**Card certification key K_cert** — a second independent key loaded at chip manufacturing:

```
K_cert = AES(K_cert_master, card_ID)
```

`K_cert_master` is held by the chip manufacturer — separate from the transport authority's `K_master`. Every genuine certified chip has `K_cert` loaded during manufacturing into **hardware-protected, firmware-inaccessible memory**: a region of the chip that only the internal security module can use for computation. No firmware instruction, no RFID command, no read operation can extract `K_cert` — it can only be used by the chip's hardware to compute a challenge response.

This separation is critical:
- `K_card` is used by card firmware to compute MACs over card data — firmware must access it
- `K_cert` is never touched by firmware — only the chip's hardware security module computes with it

**What K_cert proves**: a card that correctly answers a K_cert challenge proves (a) it is a genuine certified chip from the manufacturer, and (b) it is running unmodified firmware — modified firmware cannot access the hardware-protected region where K_cert is stored.

**What K_cert does not prove**: it does not defeat full hardware extraction — a semiconductor lab attack that probes the chip and extracts both K_card and K_cert can produce a clone that passes both checks. This attack is expensive and requires physical possession of the chip; it is a much higher bar than software modification.

### Part 6 — Card Data Structure

```
card_data = { card_ID, ride_count, transaction_counter, expiry_date }
MAC = AES-CBC-MAC(K_card, card_ID || ride_count || transaction_counter || expiry_date)
```

All four fields are bound by the MAC. Any modification to any field by anyone without K_card invalidates the MAC — the reader detects tampering immediately.

**Why card_ID is in the MAC**: prevents a cloning attack where an attacker copies card A's full data (including MAC) onto a blank card B with a different card_ID. The reader derives K_card_B and attempts to verify — the MAC fails because it was computed with K_card_A over data including card_A's card_ID. Without card_ID in the MAC, this attack would succeed.

**Why transaction_counter is in the MAC**: the monotonically increasing counter is part of the authenticated data. A rollback restores an old counter value — this is detected at the rollback check step (Part 7, Step 6). The MAC authentication and the counter check are two independent defences: authentication proves the data was written by K_card; the counter check proves the data is the most recent version.

### Part 7 — Ride Validation Protocol

```
Step 0 — Reader challenges card for certification (offline-capable):
         Reader → Card: { cert_nonce }   (128-bit random, fresh per transaction)
         Card's hardware security module computes:
             cert_response = AES-CBC-MAC(K_cert, card_ID || cert_nonce)
             (K_cert is in hardware-protected memory; firmware cannot read it directly)
         Card → Reader: { card_ID, cert_response }
         Reader derives: K_cert = AES(K_cert_master, card_ID)
         Reader verifies: AES-CBC-MAC(K_cert, card_ID || cert_nonce) == cert_response?
         If NO → reject immediately (not a certified chip, or modified firmware)
         If YES → card is a genuine certified chip running unmodified firmware; proceed

         Note: cert_nonce and the transaction nonce (Step 1) can be sent in one message
         to save a round trip within the <200 ms budget.

Step 1 — Reader → Card:
         { nonce }   (128-bit random, generated fresh per transaction, ch3.1 p.7)

Step 2 — Card computes:
         response = AES-CBC-MAC(K_card, card_data || nonce)

Step 3 — Card → Reader:
         { card_data, response }

Step 4 — Reader derives K_card:
         K_card = AES(K_master, card_ID)

Step 5 — Reader verifies MAC:
         AES-CBC-MAC(K_card, card_data || nonce) == response?
         If NO  → reject immediately (forgery or tampering detected)

Step 6 — Reader performs validity checks:
         ride_count > 0?                                              (has remaining credit)
         expiry_date not passed?                                      (card still valid)
         transaction_counter > last_seen_counter[card_ID]?           (rollback check)
         If any check fails → reject

Step 7 — Reader writes new state to card:
         new_card_data = { card_ID, ride_count - 1, transaction_counter + 1, expiry_date }
         new_MAC = AES-CBC-MAC(K_card, new_card_data)
         Reader sends { new_card_data, new_MAC } → card writes both to EEPROM

Step 7b — Reader reads back and verifies the write before opening the gate:
         Reader issues READ command → card returns { readback_data, readback_MAC }
         Reader checks: readback_data == new_card_data AND readback_MAC == new_MAC
         If confirmed → gate opens; ride is authorised; proceed to Step 8
         If NOT confirmed (card removed before write completed, or EEPROM write error):
             → gate does NOT open; ride is NOT authorised
             → back-end is NOT updated; last_seen_counter[card_ID] remains unchanged
             → card is in an uncertain state — MAC will fail at next reader (write was partial or absent)
             → passenger must visit a service terminal; back-end counter is used for card recovery

         âš  Threat model limitation of Step 7b:
         Step 7b defeats accidental or opportunistic card removal (the card physically loses power
         before the write completes). It does NOT defeat a malicious card with modified firmware
         that intentionally lies: the malicious card could return the correct new_card_data and
         new_MAC in response to the READ (causing Step 7b to pass and the gate to open), while
         internally reverting its EEPROM to the old state. On the next tap the card presents
         the old transaction_counter and old ride_count.
         The defence against this attack is the back-end rollback check in Step 6, not Step 7b.

Step 8 — Reader updates back-end (when online):
         last_seen_counter[card_ID] = transaction_counter + 1
         (This step executes only after Step 7b has confirmed the write — never before)
```

**Why the nonce is generated by the reader and not the card?** The card is passive — it has no power source and no hardware random number generator when not in a field. The reader is active infrastructure with a hardware RNG. The nonce travels from reader to card (Step 1), is included in the MAC response (Step 2), and is verified by the reader (Step 5).

**What does the nonce prevent?** Without a nonce, an eavesdropped `{ card_data, MAC }` from transaction N would be identical in a future transaction where card_data is the same (e.g., immediately after a top-up restores the same ride count). An attacker could replay the old response. The nonce ensures the MAC is unique per transaction even with identical card_data.

**Why verify MAC before checking ride count?** If the reader checks ride_count > 0 before verifying the MAC, a forged card with an invalid MAC but a crafted ride_count could pass the first check before being rejected at MAC verification. Verifying MAC first fails fast on any forgery without revealing which subsequent check would have failed.

### Part 8 — Rollback Prevention via Transaction Counter

The `transaction_counter` increments by 1 with every write operation (decrement or top-up). The reader network maintains a back-end database: `last_seen_counter[card_ID]`.

At Step 6, the reader checks: `transaction_counter (from card) > last_seen_counter[card_ID] (from database)`.

If a passenger physically restores their card's EEPROM to a snapshot (e.g., counter = 50 when the back-end records 55): the check fails → transaction rejected. The rolled-back MAC is still valid (it was computed by the real K_card over real data), but the counter reveals the state is stale.

**Why a counter and not a timestamp for rollback detection?** A timestamp requires a reliable real-time clock. Passive RFID cards have no battery and no RTC — the clock would reset to zero every time the card leaves the reader's field. A counter stored in non-volatile EEPROM persists across power cycles indefinitely without any timing hardware.

**Offline reader limitation**: readers operating without network access cannot check the back-end counter database. In this case, the reader verifies the MAC (Step 5) and local card data (Step 6, first two checks) but cannot perform the rollback check. A rollback attack therefore succeeds at offline readers — the restored MAC is valid. This is an accepted residual risk. High-throughput metro gates enforce online validation; remote or low-throughput locations (buses) accept this risk.

### Part 9 — Top-Up at Ticket Machines

Authorised ticket machines hold K_master in tamper-resistant hardware:

```
Step 1 — User presents card to ticket machine
Step 2 — Machine reads card; issues nonce; verifies MAC: K_card = AES(K_master, card_ID)
Step 3 — User inserts payment
Step 4 — Machine computes:
         new_card_data = { card_ID, ride_count + N, transaction_counter + 1, expiry_date }
         new_MAC = AES-CBC-MAC(K_card, new_card_data)
Step 5 — Machine writes { new_card_data, new_MAC } to card EEPROM
Step 6 — Machine logs transaction to back-end; back-end updates last_seen_counter
```

K_master never travels over any unprotected connection. Anomalous patterns (one card_ID receiving hundreds of top-ups per hour) trigger alerts at the back-end (ch3.7 p.85).

### Part 10 — Remaining Vulnerabilities

- **Intentional card removal during write (defeated by Step 7b)**: a passenger could attempt to yank the card away mid-write to prevent the ride count decrement, then walk through the gate. Step 7b defeats this: the gate stays closed until the READ-back confirms the write succeeded. The passenger gains no free ride; they risk corrupting their own card.
- **Accidental card removal (graceful failure via Step 7b + back-end)**: if the card is accidentally removed during the write, the gate does not open; the back-end last_seen_counter is unchanged (Step 8 never fires); the passenger visits a service terminal where the correct ride count is recovered from the back-end.
- **Malicious card lying during read-back**: a card with modified firmware could respond to the Step 7b READ with the correct new_card_data and new_MAC (passing the check, opening the gate) while internally keeping the old EEPROM state. Three independent defences apply in sequence:
  - **Step 0 (K_cert certification check — offline)**: modified firmware cannot access the hardware-protected K_cert region. The card cannot produce a valid cert_response → rejected before the transaction even begins, at any reader including offline ones. This is the primary defence against modified-firmware cards.
  - **Step 7b (read-back — offline)**: defeats accidental or opportunistic card removal, but cannot catch a card that lies deliberately (the card controls its own READ response). Not sufficient alone against intentional lying.
  - **Back-end rollback check (Step 6 — online only)**: if somehow the certification check is bypassed (e.g., full chip clone with K_cert extracted), the back-end catches the rollback on the next online tap: last_seen_counter advances to N+1 at Step 8; the card presents N again; N > N+1 fails → rejected.
  Residual vulnerability: a fully hardware-cloned card (both K_card and K_cert extracted via semiconductor probing) passes Step 0 and can exploit offline readers. Full hardware extraction is a high-cost, high-skill, physical-access attack.
- **Relay attack**: two radio devices extend the card's effective range — a transaction is conducted with a card in a victim's pocket without their knowledge (credit drained or ride validated). Challenge-response proves the card is genuine but not that it is physically present. Distance bounding (UWB) is the countermeasure but requires bidirectional hardware outside the scope of slide material.
- **K_master compromise**: extraction of K_master from a terminal using hardware probing or an insider attack allows the attacker to derive K_card for every card in the system. Unlimited free top-ups become possible. K_master must be in tamper-resistant hardware with active zeroisation on intrusion. This is the single highest-value target in the system.
- **K_card extraction from card chip**: semiconductor lab attack extracts K_card from a specific card. The attacker clones that card indefinitely. Retroactive detection: duplicate card_ID entries appearing simultaneously in the transaction log. Tamper-evident chip packaging is the primary mitigation.
- **Offline rollback**: rollback attacks succeed at offline readers as described. Accepted residual risk; online validation required at high-volume gates.

### Part 11 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Cryptographic primitive | Symmetric AES only | ch2.2.2 p.7, p.13 | Asymmetric (RSA, ECDSA) categorically impossible on passive RFID — no battery; insufficient computation time |
| MAC algorithm | AES-CBC-MAC | ch2.2.3 p.61–62 | Uses only AES operations; available on AES-hardware RFID chips without SHA hardware; HMAC rejected because SHA hardware not guaranteed |
| Key management | K_card = AES(K_master, card_ID) | ch2.2.1 p.55 | No per-card key database needed; instant derivation; one card compromise does not expose K_master |
| Freshness / replay protection | Fresh 128-bit nonce from reader per transaction | ch3.1 p.7 | Reader generates nonce (passive card has no RNG); nonce makes each MAC unique even for identical card_data |
| Rollback protection | Monotonic transaction_counter checked against back-end | ch1 p.34 | Timestamp impossible on passive card (no clock); counter persists in EEPROM; back-end detects restored snapshots |
| Why not HMAC | SHA hardware not guaranteed on passive RFID chip | ch2.2.3 p.61–62 | CBC-MAC uses same AES hardware already present; HMAC requires additional SHA-256 hardware block |
| Why not asymmetric | Computation time too limited, no battery | ch2.2.2 p.7, p.13 | RSA/ECDSA require millions of multiplications; passive RFID cannot complete in <200 ms |

### Sources

- IS_UG_1_Introduction (p.10, p.22, p.30, p.34, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.61–62, p.63–66, p.70–75)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_7_Appl_System (p.77, p.83, p.85, p.96)

_Status: Complete_  
_Done by: William_

---

> **Footnote — Can an offline reader detect a malicious card lying about its state?**
>
> **Short answer: not fully, but two partial mitigations exist.**
>
> **Why full offline detection is impossible**
>
> The card holds `K_card`. It therefore possesses everything needed to present any previous genuine state: the old `card_data` and its old `MAC` are both authentic — they were legitimately computed when that state existed. Old MACs never expire cryptographically. An offline reader has no external reference to compare the presented counter against. The card controls every byte it returns in response to a READ command, so the reader cannot distinguish "genuine current state" from "genuine old state being replayed".
>
> The only architecture that would make this impossible offline is asymmetric: if the card held a private signing key and readers held only the public key, the reader could verify the card's signature without being able to forge it, and the card could not produce a valid signature over a state it had not been authorised to hold. But asymmetric cryptography is categorically ruled out on passive RFID (ch2.2.2 p.7, p.13 — computation budget and no battery).
>
> **Partial mitigation 1 — Per-reader local counter cache**
>
> Each reader maintains a local persistent table: `{ card_ID → last_seen_counter }` for every card it has processed. On each transaction, before accepting:
> ```
> if transaction_counter_on_card â‰¤ local_cache[card_ID] → reject (rollback detected)
> ```
> After a successful transaction: `local_cache[card_ID] = new_transaction_counter`.
>
> This detects a rollback attack when the *same card returns to the same offline reader*. It provides no protection when the malicious card uses a *different* offline reader that has no prior history of this card.
>
> **Partial mitigation 2 — Reader-signed transaction log on the card**
>
> Each reader holds a per-reader key `K_reader = AES(K_master, reader_ID)` — same derivation pattern as `K_card`. After every successful write, the reader appends a signed log entry to the card's EEPROM:
> ```
> log_entry = { reader_ID, transaction_counter, AES-CBC-MAC(K_reader, reader_ID || transaction_counter) }
> ```
> The card stores the last few entries (e.g., 3–5, limited by EEPROM).
>
> An offline reader receiving the card:
> 1. Reads `card_data` and verifies `MAC_card` (standard Step 5)
> 2. Reads last log entry `{ reader_ID_prev, C_log, MAC_log }`
> 3. Derives `K_reader_prev = AES(K_master, reader_ID_prev)`
> 4. Verifies `MAC_log` — if authentic, this entry was written by a genuine authorised reader
> 5. Checks that `C_log == transaction_counter` on card — the last genuine reader confirmed this counter value
>
> A malicious card **cannot forge a log entry** because it does not know `K_master` and therefore cannot produce a valid `AES-CBC-MAC(K_reader, ...)` for any `reader_ID`. However, a malicious card **can delete log entries**: it rolls back its card_data to counter `N` and also deletes the log entry for transaction `N+1`, presenting a log that ends at counter `N`. The offline reader sees card data and log that are mutually consistent — it cannot know that counter `N+1` was already spent elsewhere.
>
> **Combined effect**
>
> | Attack scenario | Per-reader cache | Reader-signed log | Back-end (online only) |
> |---|---|---|---|
> | Rolled-back card returns to same offline reader | ✅ Detected (cache) | Partially (log deleted) | ✅ Detected |
> | Rolled-back card uses a different offline reader | ❌ No prior history | ❌ Log appears consistent | ✅ Detected |
> | Malicious card lying on read-back, first offline use | ❌ | ❌ | ✅ Detected on next online tap |
>
> **Conclusion**: per-reader cache plus reader-signed log entries reduce the window of exploitation at offline readers — they defeat repeat-visit attacks at the same reader and make the card's history partially auditable. They do not eliminate the vulnerability on their own.
>
> **Update — K_cert certification check changes this picture significantly.**
> The certification check (Step 0) IS fully offline and addresses the modified-firmware attack directly: a card running modified firmware cannot access the hardware-protected `K_cert` and therefore cannot produce a valid `cert_response`. The offline reader rejects it at Step 0, before any card data is read or any gate decision is made. This is a genuine offline fix for the specific threat of modified-firmware malicious cards.
>
> What it still cannot catch offline: a fully hardware-cloned card where both `K_card` and `K_cert` were extracted via semiconductor lab probing. Such a clone passes the K_cert challenge (it has the real K_cert) and can still exploit offline readers. The back-end counter remains the only defence against hardware-cloned cards.
>
> | Attack | K_cert Step 0 (offline) | Back-end counter (online) |
> |---|---|---|
> | Completely fake/custom RFID chip | ✅ Rejected (no K_cert) | ✅ |
> | Genuine chip, modified firmware | ✅ Rejected (K_cert hardware-protected) | ✅ |
> | Genuine chip, physical EEPROM restore | ❌ Passes (K_cert intact; firmware unmodified) | ✅ Detected |
> | Full hardware clone (K_card + K_cert extracted) | ❌ Passes (has real K_cert) | ✅ Detected on next online tap |
>
> The fundamental ceiling remains: the symmetric architecture means online validation is the only complete defence against a physically cloned card. K_cert raises the bar from "software modification" to "semiconductor lab attack."
