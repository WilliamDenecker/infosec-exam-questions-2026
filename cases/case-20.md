# Case 20

**What security services will be needed to achieve a contactless and passive² access badge used to gain entry to a building? What security mechanisms and protocol would you use to implement these security services?**

You briefly hold the access badge in front of a contactless badge reader, which will verify the validity of the badge and grant you access to the building if you have the correct access rights.

It should of course be almost impossible to forge an access badge. The system should be sufficiently fast, although the calculation power of the badge is rather limited.

² *"Passive" means it doesn't contain any batteries or other internal power source.*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | The reader must verify the badge is genuine — issued by the legitimate authority, not forged, cloned, or replayed. The badge must prove it is the original issued card, not a copy of an eavesdropped exchange. | An attacker creates a forged badge with fabricated access rights. Any RFID card with the right format is accepted. |
| **Access control / authorisation** | Yes — critical | ch1 p.30 | A valid badge may only grant access to areas the holder is authorised for. Different roles (visitor, staff, management) have different rights. A valid badge for floor 1 must not open floor 5. | Any badge holder can access any area. The access control system enforces no role-based distinction. |
| **Data integrity** | Yes | ch1 p.34 | The access rights stored on the badge must not be modifiable by the holder. A "visitor" badge must not be alterable to a "staff" badge. | A holder modifies their access rights field, then re-presents the badge. The reader grants elevated access. |
| **Availability** | Yes | ch1 p.42 | The badge check must complete in under ~200 ms. A slow protocol makes the system impractical for high-throughput entry points. | Employees wait 10 seconds at every door. The system is abandoned for practicality. |

**Confidentiality is secondary**: the access rights (e.g. "floor 3, lab A") are not sensitive enough to require encryption. The primary goal is **unforgeability and replay-resistance**.

### Part 2 — Design Constraint: Symmetric Cryptography Only

Asymmetric algorithms (RSA, ECDSA) are computationally expensive and slow (ch2.2.2 p.7, p.13). A **passive RFID badge has no battery** — it is powered entirely by the electromagnetic field of the reader. Computation time is severely limited; an asymmetric operation cannot complete within the required time window.

**AES** (ch2.2.1 p.55) with a dedicated AES hardware core (present on modern passive RFID chips) completes in microseconds. **CBC-MAC** (ch2.2.3 p.61–62) uses only AES block cipher operations — it is the natural choice for resource-constrained message authentication.

**Why CBC-MAC over HMAC?** HMAC (ch2.2.3 p.63–66) is designed for software implementations and requires a hash function (SHA-256 or similar) which may not be available in hardware on a constrained RFID chip. CBC-MAC (ch2.2.3 p.61–62) uses the AES block cipher directly, which every RFID chip with cryptographic support already implements.

### Part 3 — Badge Contents and Key Derivation

At issuance, the badge chip is programmed with:

```
badge_data = { badge_ID, access_rights, expiry_date }
MAC = AES-CBC-MAC(K_badge, badge_data)
```

`K_badge` is a secret key unique to this badge, known only to the badge chip and the Access Control Server. The MAC proves the badge data was created by the issuing authority and has not been tampered with — any modification to any field invalidates the MAC.

`K_badge` is derived from a **master key** `K_master` stored in the Access Control Server:

```
K_badge = AES(K_master, badge_ID)
```

This allows the reader/server system to derive any badge's key on the fly from the `badge_ID` — scalable to thousands of badges without storing a separate key per badge. `K_master` never leaves the Access Control Server.

**Why derive K_badge from K_master rather than storing individual keys?** Storing a unique key per badge requires a growing per-badge database at every reader. Deriving `K_badge = AES(K_master, badge_ID)` requires only `K_master` to be available at the server. A new badge is enrolled simply by programming it with `badge_data` and the derived MAC — no per-badge key distribution needed.

### Part 4 — Challenge-Response Protocol (Anti-Replay and Anti-Clone)

A static MAC on the badge is insufficient: an attacker can eavesdrop on a legitimate badge-reader exchange and replay the captured transmission. The **nonce** (ch3.1 p.7) prevents this.

```
Step 1 — Reader → Badge:   { nonce }   (128-bit random, fresh each presentation)

Step 2 — Badge computes:
          response = AES-CBC-MAC(K_badge, badge_data || nonce)

Step 3 — Badge → Reader:   { badge_ID, badge_data, response }

Step 4 — Reader contacts Access Control Server (or uses cached K_master):
          K_badge = AES(K_master, badge_ID)
          expected = AES-CBC-MAC(K_badge, badge_data || nonce)
          if response == expected:
              → badge is genuine (authentication, ch1 p.22)
              → check access_rights and expiry_date (authorisation, ch1 p.30)
              → grant or deny entry
          else:
              → reject
```

**Why this prevents forgery**: producing a valid `response` requires knowing `K_badge`. `K_badge` is stored only in the badge chip's secure hardware and derivable only by the Access Control Server from `K_master`. Without `K_badge`, computing a valid AES-CBC-MAC (ch2.2.3 p.61–62) is computationally infeasible.

**Why this prevents replay**: each presentation the reader generates a fresh random nonce. A captured `(badge_data, response)` from a previous exchange cannot be replayed — the reader generates a different nonce next time, and the old response does not match.

### Part 5 — Access Rights Enforcement

After verifying the MAC, the reader checks `access_rights` and `expiry_date` in the authenticated `badge_data`:
- Badge expired → deny entry regardless of rights.
- Badge valid but does not include this door's access right → deny entry.
- Access rights are MAC-protected — the holder cannot alter them without invalidating the MAC.

For high-security areas: reader also queries the Access Control Server online to check whether the badge has been revoked (e.g. a lost badge). For standard doors, offline MAC verification is sufficient (ch1 p.42 — availability).

### Part 6 — Remaining Vulnerabilities

- **Physical hardware extraction**: a sophisticated attacker with physical possession of the badge can attempt to extract `K_badge` from the chip using hardware probing or fault injection. Modern RFID chips include tamper-detection circuits that erase the key on physical attack, but this is a residual hardware risk.
- **Relay attack**: an attacker uses two devices to relay the RF communication between a legitimate badge (in the victim's pocket or bag) and a reader — extending the effective range of the badge without the holder's knowledge. The challenge-response provides authenticity but not proximity proof. Physical proximity verification requires UWB distance bounding — outside the scope of slide material.
- **Master key compromise**: if `K_master` is stolen from the Access Control Server, all badge keys can be derived and any badge cloned. The Access Control Server must be the most hardened component in the system.
- **Revocation latency for offline readers**: a lost badge remains valid at offline readers until its expiry date or until the reader next synchronises with the server. Short expiry periods (annual re-issuance) and online revocation checks for sensitive areas mitigate this.

### Part 7 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Badge MAC | AES-CBC-MAC with K_badge | ch2.2.3 p.61–62; ch2.2.1 p.55 | AES hardware on RFID chip; fast enough for passive badge; no battery needed |
| Replay protection | Fresh nonce per presentation, included in MAC | ch3.1 p.7 | Eavesdropped exchange is useless; nonce changes on every presentation |
| Key management | K_badge derived from K_master per badge_ID | ch2.2.1 p.55 | No per-badge key database; new badges enrolled automatically |
| Why not ECDSA | Asymmetric too slow for passive RFID | ch2.2.2 p.7, p.13 | No battery; electromagnetic power only; public-key operations infeasible in time window |

### Sources

- IS_UG_1_Introduction (p.10, p.22, p.30, p.34, p.42)
- IS_UG_2_2_1_SecM_SymmEncr (p.55)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.61–62)
- IS_UG_3_1_Appl_Basics (p.7)

_Status: Complete_  
_Done by: William_
