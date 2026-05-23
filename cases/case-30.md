# Case 30

You have a security token (with card reader) without physical contact⁴ to the host, allowing you to confirm a payment (amount: 9.30 EUR) on the Web. After you've entered your credit card data (name, card number, validity limit, verification code), you see the following instructions on your Web browser:

1. Insert your credit card into the card reader
2. Press BUY (*the card reader then asks you for the security code*)
3. Enter the security code 14473738 and confirm with OK (*the card reader then asks you whether you want to buy on the Internet*)
4. Press OK again
5. Enter the amount (9) and confirm with OK
6. Enter your PIN and press OK (*the card reader then shows an 8 digit "signing code"*)
7. Enter the signing code below (*in your Web browser*) and press "Submit"

**How might such a system work (which cryptographic algorithms, which key sizes, which input, etc.)?**

**How vulnerable is this procedure to malware on the user's host?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

A few additional notes:

- The website itself is protected using TLS 1.3 (certificate for 2048 bit RSA public key; The connection is encrypted using AES_256_GCM, SHA-2-384 is the hash function for HMAC, ECDHE is used for the key exchange mechanism, and RSA is used in the server authentication of the handshake).
- If you repeat the procedure with the same input on the card reader, you'll obtain a different "signing code", which will also be accepted by the website
- After a few minutes, the combination "security code"/"signing code" will no longer be accepted
- If you attempt to input five erroneous "signing codes", your contract will be blocked
- If you enter a wrong PIN three times in a row on the card reader, your card will be blocked. To unblock your card, you'll need to use your card in an ATM or to perform a PIN reset in your bank agency

⁴ *This means it cannot receive data from your computer or send data to your computer. You can manually input data usind the keypad of the token and the output of the token can be read on its (small) display.*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication / data-origin authentication** | Yes — critical | ch1 p.22 | The bank must verify the transaction was authorised by the legitimate card holder — not an attacker who captured the card details from a previous session. | An attacker who intercepted the card number submits a payment without the holder's knowledge using a replayed or forged signing code. |
| **Data integrity** | Yes — critical | ch1 p.34 | The payment amount must be cryptographically bound to the authorisation. The signing code must be tied to the specific amount the user approved — not to some other amount. | A man-in-the-browser changes â‚¬9 to â‚¬900 after the user generates the signing code. The bank receives the higher amount with a valid code and processes the fraud. |
| **Non-repudiation** | Yes | ch1 p.40 | The card holder must not be able to deny having authorised this specific transaction at this specific amount and time. | A holder authorises â‚¬9, transaction goes through for â‚¬9, holder disputes it claiming they never approved any payment. No cryptographic proof exists. |
| **Confidentiality** | Yes | ch1 p.15 | The security code and signing code must be encrypted in transit. TLS provides this. | An eavesdropper captures the signing code and replays it before the few-minute window expires, authorising a duplicate payment. |

### Part 2 — Transaction Signing vs Simple Login Authentication

This case is fundamentally different from case 21 (USB card reader login) and case 25 (HardKey TOTP login).

| | Login authentication (cases 21, 25) | Transaction signing (case 30) |
|---|---|---|
| **Goal** | Prove identity of user | Prove user approved this specific transaction |
| **What is signed** | Session identifier / nonce | Amount + session nonce + time |
| **Attack prevented** | Impersonation | Man-in-the-browser amount substitution |
| **Signing output** | Used to establish a session | Authorises one specific payment |
| **Post-auth risk** | Authenticated session can be hijacked | Each transaction requires fresh authorisation |

In case 21, once the user is authenticated the session is open — a man-in-the-browser can inject fraudulent transactions into that session. In case 30, the amount is bound to the signing code — changing the amount in the browser after the user signs it makes the signing code invalid at the bank.

### Part 3 — Signing Mechanism: Why HMAC and Not Asymmetric Signature (ECDSA)

The signing code must be entered manually by the user — 8 decimal digits typed from the reader's small display into the browser.

**Could ECDSA P-256 be used?** An ECDSA signature is 64 bytes = 128 hexadecimal characters = approximately 155 decimal digits. **A user cannot manually read and type 155 digits from a small display.** The output must fit on a small display and be typeable in seconds. This constraint eliminates all asymmetric signature schemes — their output is too large for manual entry.

**Does ECDSA offer non-repudiation that HMAC doesn't?** Yes — with ECDSA, only the card's private key can produce the signature, so the bank can prove in court the card holder authorised it. With HMAC, the bank also knows K and could theoretically forge a signing code. This is a meaningful difference for legal non-repudiation. However, the practical constraint (8-digit manual entry) makes ECDSA physically impossible in this setup. HMAC is the only viable mechanism.

**Why HMAC-SHA256 and not alternatives?**

| Alternative | Why rejected |
|---|---|
| Plain `SHA-256(K \|\| inputs)` | Length-extension attack (ch2.2.3 p.24–32): knowing `SHA-256(K \|\| inputs)` allows extending the message without knowing K. HMAC's double-hashing prevents this. |
| AES-CBC-MAC(K, inputs) | Cryptographically equivalent and also valid. HMAC-SHA256 chosen because it is directly covered in slides (ch2.2.3 p.63–66) and SHA-256 hardware is available on modern card chips. |
| Plain hash without key | No secret key → any observer can compute the same hash for any inputs → zero authentication. |
| Counter-based (no timestamp) | Codes would not expire automatically. A captured code remains valid until used. "After a few minutes, no longer accepted" requires a time component. |

**Chosen: HMAC-SHA256** truncated to 8 decimal digits:

```
signing_code = truncate(HMAC-SHA256(K, security_code || amount || timestamp), 8 digits)
```

### Part 4 — The Three HMAC Inputs: Why Each Is Essential

Each input to HMAC serves a distinct, non-redundant security purpose. Removing any one creates a specific, exploitable attack.

#### security_code — Server-Generated Nonce (ch3.1 p.7)

The security code (e.g., `14473738`) displayed in the browser is a **server-generated nonce** — a fresh random value generated by the bank for this specific payment session. It is transmitted to the browser over TLS.

**Why is it in the HMAC?** It binds the signing code to this specific transaction session. Even if an attacker captures a signing code from session A, they cannot reuse it in session B — the server generates a different nonce for session B, so the HMAC input differs and the signing code is different.

**Without security_code**: the signing code depends only on amount and timestamp. An attacker who captures a signing code for a â‚¬9 payment can replay it in any other session (within the same time window) for a â‚¬9 payment to any beneficiary.

**Why it is typed manually on the reader**: the reader has no electrical connection to the PC (⁴). The user manually reads the security code from the browser and types it on the reader's keypad. This is the only data channel from the internet-side session to the isolated card reader.

#### amount — Binds Signing Code to the Specific Payment Value

The user manually types the amount (9) on the reader's keypad in step 5.

**Why is it in the HMAC?** If the amount is not in the HMAC, a man-in-the-browser can change the displayed amount in the browser after the user has already generated the signing code. The bank receives amount=â‚¬900 with a signing code that was computed over amount=â‚¬9. Without amount in the HMAC, this code verifies — the bank has no way to detect the substitution.

**With amount in the HMAC**: the bank computes `HMAC-SHA256(K, security_code || "900" || timestamp)`. The card computed `HMAC-SHA256(K, security_code || "9" || timestamp)`. These are different → the codes do not match → the transaction is rejected.

**Critical security property**: the amount is entered by the user directly on the reader's keypad — not taken from the browser. Malware in the browser cannot change what the user types on the reader. The reader is the **trusted input device** for the amount.

#### timestamp — Provides Automatic Expiry (ch3.1 p.3)

**Why is it in the HMAC?** The notes specify "after a few minutes, no longer accepted". The timestamp T = floor(unix_time / window) changes with each time window. Once the window passes, `T` advances, and any signing code computed with the old `T` no longer matches anything the bank will accept.

**Without timestamp**: the signing code depends only on security_code and amount. A captured code is valid for the lifetime of the security_code (until the session expires at the server). The expiry must be enforced by the HMAC inputs, not just by the server session timeout.

**Why does "repeating with the same input gives a different code"?** Because T advances between attempts. Even if the user enters the same security_code and the same amount twice in succession, the timestamp T has changed between the two presses — different T → different HMAC → different code. Both codes are valid (both correspond to valid T windows the server accepts).

### Part 5 — Key Material and PIN Verification

#### 256-bit Secret Key K on Card Chip

Each card contains a unique **256-bit secret key K** stored in the card's secure hardware. The bank holds K for this card in its secure backend.

**Why 256-bit and not 128-bit for K?** K is a long-lived secret embedded in a physical card that may be used for many years. Against Grover's quantum algorithm (ch2 PQCrypto p.16), 128-bit → 64-bit effective — obsolete. 256-bit → 128-bit effective post-quantum. For a long-lived key protecting financial transactions, 256-bit is the correct choice.

K never leaves the card chip. It cannot be read out via the card reader or PC under any circumstances.

#### PIN Verification: Card Chip, Not Reader (ch3.7 p.46)

The PIN is entered on the reader's keypad in step 6. The reader forwards the PIN to the **card chip**, which verifies it internally:

```
Correct PIN → card chip unlocks K for signing use
Wrong PIN   → card chip increments internal wrong-PIN counter
              (card blocked after 3 failures — enforced by card chip hardware,
               not by the reader — a faulty reader cannot bypass this)
```

**Why must PIN verification be on the card chip and not the reader?** If the reader verified the PIN, a faulty or malicious reader could simply skip the check — or allow unlimited guessing without incrementing any counter. The card chip is the **trust anchor**: it enforces the counter regardless of what the reader does. A reader that doesn't increment a counter is irrelevant if the counter lives on the chip.

### Part 6 — Server Verification

The bank holds:
- K for this card (looked up by card number sent in the initial form)
- The security_code it generated for this session
- The current time T

The bank computes:

```
candidate = truncate(HMAC-SHA256(K, security_code || amount || T), 8 digits)
```

for T and neighbouring time windows. If the submitted signing code matches → payment authorised. After 5 wrong signing codes → account blocked (ch3.7 p.85).

**Why the bank uses the expected amount from the form submission and not from the signing code?** The signing code is just 8 digits — it doesn't carry the amount. The bank takes the amount from what was submitted in the payment form (the browser-side amount), then computes the HMAC using that amount. If malware changed the browser amount from â‚¬9 to â‚¬900, the bank recomputes `HMAC(..., "900", ...)` — which doesn't match the code the card computed with `"9"` → rejected. This is the man-in-the-browser protection mechanism.

### Part 7 — Vulnerability to Malware on the PC

#### What Malware Cannot Do

**Cannot change the amount undetected**: the amount is manually typed on the reader keypad by the user. The HMAC is computed over the amount the user typed. If malware changes the browser submission from â‚¬9 to â‚¬900, the bank computes `HMAC(..., "900", ...)` but the card computed `HMAC(..., "9", ...)` — mismatch → rejected. The reader display shows what the card is signing; the user must verify this.

**Cannot capture the PIN**: the PIN is entered on the reader's keypad and forwarded directly to the card chip — it never passes through the PC. A PC keylogger sees nothing.

**Cannot steal K**: K is stored in the card chip's secure hardware and never exported to any external memory or interface.

**Cannot replay the signing code**: the security_code (nonce) is unique per session. A captured signing code is valid only for that specific security_code and amount. A different session generates a different security_code → different HMAC → different signing code.

**Cannot use an expired signing code**: the timestamp component means codes expire within minutes (ch3.1 p.3). A captured code is useless after the time window closes.

#### Residual Risks

**Social engineering (residual risk — most significant)**: malware modifies the amount displayed in the browser (showing â‚¬9.30 while the actual payment is â‚¬900). The browser tells the user "please enter amount **9** on the reader". The user types 9 on the reader (matching what the browser instructed). The card signs `HMAC(..., "9", ...)`. The malware then submits â‚¬900 in the payment form. The bank recomputes `HMAC(..., "900", ...)` → mismatch → rejected.

**Wait — this doesn't work.** If the card signs â‚¬9 and the bank receives â‚¬900, the codes don't match and the payment is rejected. The attacker must instruct the user to type a different amount — e.g., display "please type 900 on your reader". A suspicious user who checks the reader display will see "Amount: 900" and notice the discrepancy with the stated purchase of â‚¬9.30. The defence is that the reader's display is the trusted output — the user must verify what the reader shows, not just follow the browser's text instructions.

**Real-time relay (residual risk)**: an attacker controlling the browser in real time could relay the security_code to their own fraudulent payment session, get the user to unknowingly sign a different transaction. This requires real-time interception and the user not noticing the transaction context shown on the reader. The short expiry window limits the time available for such an attack.

### Part 8 — Comparison with Case 21 (USB Card Reader Login)

| | Case 21 — login | Case 30 — transaction signing |
|---|---|---|
| **What is proved** | Identity (who you are) | Authorisation (what you approved) |
| **Cryptographic primitive** | ECDSA (asymmetric, 64-byte output) | HMAC-SHA256 (symmetric, 8-digit output) |
| **Why different primitives** | Output transmitted digitally; size not a constraint | Output entered manually; must be â‰¤8 digits |
| **Amount binding** | No — not relevant for login | Yes — amount in HMAC prevents man-in-the-browser |
| **Malware post-auth** | Session can be hijacked | Each transaction requires new signing code |
| **Reader connection** | USB — nonce sent digitally | None — security code typed manually |

### Part 9 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Signing mechanism | HMAC-SHA256 truncated to 8 digits | ch2.2.3 p.63–66 | Only option for manual entry; ECDSA/RSA output is 64–256 bytes — impossible to type; AES-CBC-MAC equivalent but HMAC directly in slides |
| Why not ECDSA | Output size incompatible with manual entry | ch2.2.3 p.85–87 | 64 bytes = 155+ digits; cannot be read/typed from small display |
| HMAC input: security_code | Server nonce binds to specific session | ch3.1 p.7 | Cross-session replay defeated; different session = different nonce = different code |
| HMAC input: amount | Binds code to specific payment value | ch1 p.34 | Man-in-the-browser amount substitution detected; changed amount → HMAC mismatch → rejected |
| HMAC input: timestamp | Automatic expiry in minutes | ch3.1 p.3 | Captured code expires; replay after time window impossible; "same input → different code" explained by advancing T |
| Key K | 256-bit on card chip | ch2 PQCrypto p.16 | Long-lived key; Grover's reduces 128-bit to 64-bit obsolete; 256-bit retains 128-bit post-quantum |
| PIN verification | Card chip (not reader) | ch3.7 p.46 | Faulty reader cannot bypass PIN check; counter enforced by chip hardware; keylogger on PC sees nothing |
| Transport | TLS 1.3, AES-256-GCM, ECDHE (as specified) | ch3.6 p.7–8, p.18, p.37 | Security code and signing code confidential in transit; forward secrecy |
| Brute-force protection | Account blocked after 5 wrong signing codes; card blocked after 3 wrong PINs | ch3.7 p.85 | 10⁸ space impractical with lockout; two independent lockout mechanisms |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.34, p.40, p.42)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.61–62, p.63–66, p.85–87, p.88–93)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16)
- IS_UG_3_1_Appl_Basics (p.3, p.6, p.7)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.46, p.85)

_Status: Complete_  
_Done by: William_
