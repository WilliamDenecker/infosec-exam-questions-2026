# Case 15

**What security services will be needed to enable electronic cash for small online payments? What security mechanisms and protocol would you use to implement these security services?**

The goal is to develop a user-friendly, but correctly secured, mechanism for paying small amounts online. For these small payments, systems such as credit card, debit card, whether or not using a token such as the Digipass, are not ideal in terms of usability or cost structure.

The emphasis is therefore on ease of use and simplicity of use, but the security must still be sufficient for users to be able to trust the system. Such a system could be somewhat similar to an electronic version of cash.

Consider the following aspects for your implementation:

- Avoid an overly cumbersome payment procedure on the client side (otherwise we might as well use two-factor authentication).
- Avoiding replay is essential. With regular cash, this problem is simple: you hand over the money on payment and printing is (approximately) impossible. With an electronic payment, of course, things are somewhat different.
- If necessary, you may use a trusted third party.

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication / data-origin authentication** | Yes — critical | ch1 p.22 | The merchant must verify the coin was genuinely issued by the bank — not forged by a user printing their own. | A user creates fake coins with arbitrary denominations. The entire currency system collapses. |
| **Data integrity** | Yes — critical | ch1 p.34 | The denomination and serial number on the coin must not be alterable. A â‚¬0.10 coin must not be modifiable to â‚¬100. | A user alters a coin's denomination field. The merchant receives â‚¬0.10 but credits â‚¬100 worth of goods. |
| **Non-repudiation** | Yes | ch1 p.40 | The bank must not be able to deny having issued a specific coin; the merchant must not deny having received payment. | A bank claims it never issued a valid coin the user paid with; a merchant denies receiving payment. No proof exists either way. |
| **Confidentiality** | Yes | ch1 p.15 | Payment amounts and coin contents must not be readable in transit. TLS provides this at no extra cost. | An eavesdropper learns the user's spending patterns and coin values. Traffic analysis reveals financial behaviour. |
| **Availability** | Yes | ch1 p.42 | The clearinghouse that validates coins must be reachable at point of payment. | Merchants cannot validate coins; either they refuse payment (availability failure) or accept unvalidated coins (replay risk). |

### Part 2 — Design: Bank-Issued Signed Coins with a Clearinghouse

The system uses a **trusted third party bank** (ch3.1 p.6) that issues digital coins. The anti-forgery mechanism is a **digital signature** on each coin (ch2.2.3 p.76–78). The anti-replay mechanism is a **clearinghouse** that tracks spent serial numbers.

**Why a clearinghouse is necessary**: physical cash cannot be copied — handing it over transfers the unique physical object. A digital coin is just bytes and can be trivially copied. Without a clearinghouse tracking spent serial numbers, the same digital coin could be spent an unlimited number of times. The clearinghouse is the electronic equivalent of physically handing over a unique object.

#### Coin Structure

```
coin = {
    denomination,          // e.g. â‚¬0.10
    serial_number,         // 128-bit random value, unique per coin
    expiry_date,           // bounds the clearinghouse database lifetime
    bank_signature         // ECDSA P-256 signature over the above fields
}
```

The bank signs using **ECDSA P-256** (ch2.2.3 p.85–87) over a **SHA-256** hash (ch2.2.3 p.24–32) of the coin fields. The bank's public key is publicly known and embedded in merchant software and user wallets. Anyone can verify the signature; only the bank can produce it.

**Why ECDSA P-256 over RSA-PSS?** RSA-PSS (ch2.2.3 p.88–93) achieves equivalent security with a 2048-bit key but produces 256-byte signatures. ECDSA P-256 produces 64-byte signatures — 4Ã— more compact, faster to verify, and smaller per coin structure. Given that millions of small-denomination coins are expected, compactness is important.

### Part 3 — Protocol

All steps over **TLS 1.3** (ch3.6 p.7–8), `TLS_AES_256_GCM_SHA384`, ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37).

#### Phase 1 — Coin Purchase (Done in Advance, Not Per Payment)

```
Step 1 — User → Bank:     authentication (username + challenge-response via nonce, ch3.1 p.7)
Step 2 — User → Bank:     request for coins of specified denominations
Step 3 — Bank:            debits user account, generates coins, signs each with ECDSA P-256
Step 4 — Bank → User:     { coinâ‚, coinâ‚‚, ... } delivered over TLS
Step 5 — User:            stores coins locally in wallet application
```

Authentication (Step 1) is required only at top-up time — not at each individual payment. Password stored as `SHA-512(salt || password)` (ch3.2 p.11); challenge-response nonce (ch3.1 p.7) prevents pass-the-hash.

#### Phase 2 — Payment (Simple, No Authentication Required)

```
Step 1 — User selects appropriate coin from wallet
Step 2 — User → Merchant: { coin, merchant_id, timestamp }
Step 3 — Merchant:        verifies ECDSA signature on coin (confirms genuine bank issuance,
                           correct denomination, not expired)
Step 4 — Merchant → Clearinghouse: { serial_number }
Step 5 — Clearinghouse:   if serial_number NOT in spent list → mark spent, return OK
                          if serial_number IN spent list → reject (replay/double-spend)
Step 6 — Merchant:        on OK → deliver goods/service to user
```

The user experience at payment time: click "pay" → wallet selects and sends a coin automatically. No password, no MFA, no one-time code. The merchant handles clearinghouse validation transparently.

### Part 4 — Usability

- The user pre-loads a wallet with coins (one-time authentication at top-up time).
- At each payment: fully automatic — as simple as physical cash.
- The clearinghouse validation (Step 4–5) happens between merchant and bank, invisible to the user.
- Coins expire after a configurable period, bounding the size of the clearinghouse spent-serial database.

### Part 5 — Remaining Vulnerabilities

- **Wallet theft**: if an attacker accesses the user's wallet file, they can spend all stored coins before the user notices. Wallet must be encrypted at rest with **AES-256-GCM** (ch2.2.3 p.70–75) protected by a wallet PIN. No unencrypted wallet file on disk.
- **Clearinghouse unavailability**: if the clearinghouse is unreachable, merchants cannot validate coins (ch1 p.42). The clearinghouse must be a high-availability, replicated service. Merchants must choose between accepting unvalidated coins (double-spend risk) or refusing payment (availability impact).
- **Bank signing key compromise**: if the bank's ECDSA P-256 private key is stolen, an attacker can mint unlimited valid coins accepted by all merchants. The bank's signing key must be kept in an offline, air-gapped facility (ch3.1 p.19 — offline root CA model). This is the single highest-impact risk.
- **Anonymity limitations**: the bank knows which coins it issued to which user at top-up time. This design is not fully anonymous. For full anonymity, blind signatures would be needed — beyond the scope of slide material.

### Part 6 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Coin authenticity | ECDSA P-256 signature per coin | ch2.2.3 p.85–87; ch1 p.22, p.40 | Compact 64-byte signature; only bank can sign; publicly verifiable |
| Replay prevention | Serial number + clearinghouse | ch1 p.34; ch3.1 p.6 | Electronic equivalent of handing over physical cash; spent serials cannot be reused |
| Coin hash | SHA-256 | ch2.2.3 p.24–32 | No known collision attack; denomination and serial tamper-evident |
| Transport | TLS 1.3, AES-256-GCM, ECDHE | ch3.6 p.7–8, p.18, p.37 | Forward secrecy; coin values confidential in transit |
| Wallet security | AES-256-GCM at rest | ch2.2.3 p.70–75 | Physical wallet theft yields only ciphertext |

### Sources

- IS_UG_1_Introduction (p.10, p.22, p.34, p.40, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.76–78, p.85–87, p.88–93)
- IS_UG_3_1_Appl_Basics (p.6–7, p.19)
- IS_UG_3_2_Appl_AuthMeth (p.11)
- IS_UG_3_6_Appl_TLS (p.7–8, p.18, p.37)

_Status: Complete_  
_Done by: William_
