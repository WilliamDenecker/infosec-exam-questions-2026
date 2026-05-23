# Case 21

You have a security token (with card reader) *connected* by a USB cable to the host. This token (the bank card reader) also has a small display and a small keypad (numerical digits and a few functional keys, e.g. "OK" and "Cancel"), see also Fig. 1. You have installed a security plug-in (software) that enables the communication between your computer/browser and your token. The bank card has already been inserted in the bank card reader.

You see the following instructions on your Web browser when you log in to your bank web site:

1. **Card number**
   Welcome, your card with number 6703 1234 1234 1234 1 has been correctly read by the bank card reader.

2. **PIN code**
   Please follow the instructions on your bank card reader.
   *The bank card reader then asks for your PIN code*
   Type your PIN code and press "OK" to log in
   (Or cancel the login by pressing "Cancel")

After having typed your (correct) PIN code on the bank card reader, you are logged in to your bank web site.

**How might such a system work (which cryptographic algorithms, which key sizes, which input, etc.)?**

**How vulnerable is this procedure to malware on the user's host?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

A few additional notes:

- The website itself is protected using TLS 1.3 (certificate for 2048 bit RSA public key; The connection is encrypted using AES_256_GCM, SHA-2-384 is the hash function for HMAC, ECDHE is used for the key exchange mechanism, and RSA is used in the server authentication of the handshake).
- If you enter a wrong PIN three times in a row on the card reader, your card will be blocked. To unblock your card, you'll need to use your card in an ATM or to perform a PIN reset in your bank agency

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | The bank must verify that the entity logging in is the legitimate card holder — not an attacker who stole credentials. The card reader provides hardware-bound authentication that the browser alone cannot. | A stolen password gives full access to the bank account. There is no second barrier. |
| **Confidentiality** | Yes | ch1 p.15 | The authentication exchange must not be readable by any eavesdropper. TLS provides this. | An eavesdropper captures the nonce and signature and replays them to authenticate as the victim. |
| **Data integrity** | Yes | ch1 p.34 | The signed authentication challenge must not be modifiable in transit. | A man-in-the-middle modifies the nonce-response to a different transaction, gaining authenticated access. |

### Part 2 — How the System Works

The bank card contains an asymmetric keypair — a **private key** (ECDSA P-256, ch2.2.3 p.85–87) that never leaves the card chip, and a corresponding public key registered with the bank at card issuance. Authentication is a **challenge-response** protocol (ch3.1 p.7) using this keypair.

**Why ECDSA P-256 over RSA-PSS?** RSA-PSS (ch2.2.3 p.88–93) achieves equivalent security with a 2048-bit key but produces 256-byte signatures and requires more computation. ECDSA P-256 produces 64-byte signatures at 128-bit security, faster to compute on the card's constrained hardware.

All communication runs over the already-established **TLS 1.3** session (as specified: AES-256-GCM, SHA-384, ECDHE for key exchange, ch3.6 p.7–8). TLS provides the confidentiality and integrity layer; the ECDSA challenge-response provides the authentication above TLS.

#### Full Protocol

```
Step 1 — Browser → Bank (over TLS):
         { card_number }
         Bank looks up the registered public_key for this card_number in its database.
         Bank stores { session → card_number, public_key } server-side.

Step 2 — Bank generates a fresh 128-bit random nonce (ch3.1 p.7).
         Bank stores { session → nonce } server-side.
         Bank → Browser → Plugin → Card reader:  { nonce }

Step 3 — Card reader display:  "Enter PIN"
         User types PIN on card reader keypad
         (PIN entered on reader keypad, NOT on PC keyboard)

Step 4 — Card reader forwards PIN to the card chip.
         The card reader does NOT check the PIN itself — it is untrusted.
         The CARD CHIP verifies the PIN against its own internally stored value.
         Correct PIN → card chip unlocks private key for signing.
         Wrong PIN   → card chip increments its own internal wrong-PIN counter
                       (card blocked after 3 failures — enforced by card chip hardware,
                        not by the reader; a faulty or malicious reader cannot bypass this).

Step 5 — Card chip computes:
         signature = ECDSA_sign(private_key, SHA-256(nonce || card_number))
         card_number is stored on the card chip and included in the signed input
         to bind the signature to this specific card.

Step 6 — Card reader → Plugin → Browser → Bank (over TLS):
         { signature }
         Only the signature is sent — the bank already has card_number (step 1)
         and nonce (step 2) from its server-side session state.

Step 7 — Bank reconstructs the expected hash using its own session state:
         expected_hash = SHA-256(nonce || card_number)
         ECDSA_verify(registered_public_key, expected_hash, signature)
         If valid → authenticated; session established.
         If a different card submitted a signature under a different nonce,
         the hash would not match → rejected.
```

**Why is the nonce essential?** The nonce (ch3.1 p.7) ensures freshness: a signature captured from a previous session is bound to the old nonce and is completely useless for any new login session. The bank generates a fresh nonce for every login attempt.

**Why is the PIN on the card reader and not the PC?** The PIN is forwarded from the reader's keypad directly to the card chip — it never passes through the PC. Entering it on the card reader's dedicated keypad ensures the PC (including any malware running on it) never receives the PIN value. The card reader itself is also untrusted for PIN verification: the card chip enforces the counter, so a faulty or compromised reader cannot allow unlimited PIN attempts.

### Part 3 — Vulnerability to Malware on the PC

**Malware cannot capture the PIN**: it is entered on the card reader's own dedicated keypad, not routed through the PC. A keylogger on the PC sees nothing. The PC software plugin only transfers the nonce to the card and the signature back — it never handles the PIN. Even a compromised reader cannot allow unlimited PIN guessing: the wrong-PIN counter and lockout are enforced by the card chip's hardware, not by the reader.

**Malware cannot steal the private key**: the private key is stored in the card's secure hardware and never exported to any external device or memory. Even with full OS compromise, the raw key bytes are inaccessible.

**Malware cannot replay a captured signature**: each login uses a fresh server-generated nonce. A captured `(nonce, signature)` pair from a past login is useless once the server moves to the next session — the new nonce will not match the old signature.

**Malware cannot forge a signature**: producing a valid ECDSA signature requires the private key. Without it, any forgery attempt is computationally infeasible (ch2.2.3 p.85–87).

**Residual risk — session hijacking**: once the user is successfully authenticated, the TLS session exists in the browser. Man-in-the-browser malware can inject fraudulent transactions into that authenticated session (e.g. initiate a bank transfer with a different amount or beneficiary). The login authentication step is secure, but post-login actions are not separately protected by the card. This is why transaction signing (case 30) adds an additional HMAC signing step per transaction, binding the transaction amount to the signed output.

**Residual risk — DNS/TLS spoofing**: malware that successfully intercepts the TLS handshake and presents a forged server certificate (e.g. by poisoning the browser certificate store) could substitute a different nonce. Proper TLS certificate validation (ch3.2 p.28, ch3.6 p.5) and user attention to browser warnings prevent this.

### Part 4 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Signature algorithm | ECDSA P-256 per session | ch2.2.3 p.85–87 | 64-byte compact signature; faster on card hardware than RSA |
| Authentication | Asymmetric challenge-response with nonce | ch3.1 p.7 | Private key never leaves card; nonce prevents replay |
| PIN entry | Card reader keypad (not PC keyboard) | ch3.7 p.46 | Keylogger on PC sees nothing; PIN isolated from OS |
| Transport | TLS 1.3, AES-256-GCM, ECDHE (as specified) | ch3.6 p.7–8, p.18, p.37 | Confidentiality; forward secrecy; server authenticated by certificate |
| Brute-force protection | Card blocked after 3 wrong PINs | ch3.7 p.85 | Online PIN guessing infeasible; card must be physically unblocked at ATM |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.34)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.85–87, p.88–93)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.28–29)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.46, p.85)

_Status: Complete_  
_Done by: William_
