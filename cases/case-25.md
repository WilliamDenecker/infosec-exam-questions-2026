# Case 25

![image](../images/case_25.png)

You have a so-called HardKey, which is a security token (without card reader, but with a small display, see also Fig. 2) without physical contact³ to the host, allowing you to log in to your bank web site. You see the following instructions on your Web browser:

1. Enter your username then click on "next step" (both on the web site)
2. Get your HardKey and press the OK button to switch it on
3. When your see "1.Login" press the OK button
4. Enter your PIN on your HardKey and press the OK button
5. Enter the (8 decimal digit) code displayed on your HardKey (on the web site)

**How might such a system work (which cryptographic algorithms, which key sizes, which input, etc.)?**

**How vulnerable is this procedure to malware on the user's host?**

*Note: I expect you to make a choice and to defend this choice. Don't present a range of possible solutions.*

A few additional notes:

- The website itself is protected using TLS 1.3 (certificate for 2048 bit RSA public key; The connection is encrypted using AES_128_GCM, SHA-2-256 is the hash function for HMAC, ECDHE is used for the key exchange mechanism, and RSA is used in the server authentication of the handshake).
- If you repeat the procedure with the same input on the token, you'll obtain a different 8 digit code, which will also be accepted by the website
- After a few minutes, the original 8 digit code will no longer be accepted
- If you attempt to input five erroneous 8 digit codes, your contract will be blocked
- If you enter a wrong PIN three times in a row on the token, your token will be blocked. To unblock your token, you'll need to contact the bank, which can reset it

³ *This means it cannot receive data from your computer or send data to your computer. You can manually input data usind the keypad of the token and the output of the token can be read on its (small) display.*

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Authentication** | Yes — critical | ch1 p.22 | The bank must verify the person logging in is the legitimate account holder — not an attacker who obtained the username. The HardKey provides a hardware-bound second factor. | A stolen username alone gives access to the bank account. There is no second barrier. |
| **Confidentiality** | Yes | ch1 p.15 | The 8-digit code and username must be encrypted in transit. TLS provides this. | An eavesdropper captures the code and reuses it before it expires. |
| **Availability** | Yes | ch1 p.42 | The HardKey must generate a valid code whenever the user needs to log in. Clock drift between device and server must not permanently block legitimate access. | The user's code falls outside the server's acceptance window; login becomes impossible until re-synchronisation at the bank. |

### Part 2 — Design Constraint: No Connection to the Host

The HardKey has **no connection whatsoever to the PC** (³) — it cannot receive data from the host. This single constraint eliminates the most obvious approach and forces a specific class of solution.

#### Why Not Challenge-Response with a Server Nonce?

A classic challenge-response (ch3.1 p.7) works as follows: the server sends a fresh nonce → the device signs or MACs it → the response is sent back. This is the solution in case 21 (USB card reader) and case 30 (payment card reader). **It requires the device to receive the nonce.** The HardKey cannot receive anything — a challenge-response is structurally impossible without a data channel from the PC to the device.

#### Why Not Asymmetric Authentication (ECDSA)?

With ECDSA (ch2.2.3 p.85–87), the device could sign a challenge with its private key. But again, the challenge must be received from the server — impossible without a data channel. Furthermore, an ECDSA signature is 64 bytes = approximately 155 hexadecimal characters. A user cannot manually read and type this from a small display. Rejected on both grounds.

#### Why Not a Static Password on the Device?

A static code stored on the device provides no replay protection. A captured code is valid forever. Rejected.

#### Why Not a Counter-Based Code?

A counter-based approach (like case 26, the car key fob) maintains a counter that increments with each use. The problem for the HardKey scenario is **synchronisation**: if the user presses the button many times without submitting a code, the device counter advances but the server counter does not. Resynchronisation requires a bank interaction. More critically, for a login scenario, if codes expire after a few minutes regardless, the time window naturally handles freshness — no counter needed. The question states "after a few minutes the code will no longer be accepted" which is a time-based expiry, not a use-based expiry. Time-based is the right model.

#### Why Time-Based Code (Chosen)?

Both the HardKey and the bank independently know two things: the shared secret K and the current time. If they compute the same function of these two values, they arrive at the same code — no communication needed. The current time acts as an implicit shared nonce (ch3.1 p.3 — timestamps for freshness): it changes every time window, so old codes become invalid automatically. This is the only mechanism that works for a device with no incoming data channel.

### Part 3 — How the System Works

#### Key Material

Each HardKey contains a unique **256-bit secret key K** loaded during device manufacturing and bank account association. The bank stores K alongside the user's account.

**Why 256-bit and not 128-bit for K?** K is a long-lived secret embedded in a physical device that may be used for years or decades. Against Grover's quantum algorithm, a 128-bit key provides only 64-bit effective security — considered obsolete (ch2 PQCrypto p.16). A 256-bit key retains 128-bit effective security post-quantum. For a long-term device secret, the extra cost is negligible.

K never leaves the HardKey hardware. The PIN is stored on the HardKey and verified locally by the device — it prevents code generation if the device is found or stolen.

#### Code Generation

When the correct PIN is entered, the HardKey computes:

```
T    = floor(current_unix_timestamp / time_window)    // time window index (ch3.1 p.3)
code = truncate(HMAC-SHA256(K, T), 8 digits)          // (ch2.2.3 p.63–66)
```

**T** is the time window index — an integer that increments once per window. All codes generated within the same window produce the same T, hence the same code. The next window produces a completely different T and a completely different code.

**Why HMAC-SHA256 and not alternatives?**

| Alternative | Why rejected |
|---|---|
| Plain `SHA-256(K \|\| T)` | Vulnerable to length-extension attacks (ch2.2.3 p.24–32): an attacker who knows `SHA-256(K \|\| T)` can compute `SHA-256(K \|\| T \|\| padding \|\| X)` for arbitrary X without knowing K. HMAC's double-hashing construction prevents this. |
| `SHA-256(T \|\| K)` | Same length-extension vulnerability; additionally, K appears at the end which is even more exploitable in some constructions. |
| AES-CBC-MAC(K, T) | AES-CBC-MAC (ch2.2.3 p.61–62) would also work cryptographically, but it uses AES as its core operation. The HardKey may have SHA-256 hardware rather than AES hardware. HMAC-SHA256 is specified in the slides and is the natural choice. |
| HMAC-SHA512 | Produces a 512-bit output — still truncated to 8 decimal digits. The extra computation provides no additional security for 8-digit output. HMAC-SHA256 is sufficient. |
| Plain HMAC without truncation | An 8-digit decimal output is needed for manual user entry. Truncation is necessary; the truncation to 8 decimal digits is specified by the question. |

**Why the HMAC input includes only T and not more?** K already makes the output device-specific. T makes it time-specific. There is no additional information available on a disconnected device. These two inputs are sufficient for the security properties required.

#### Time Window and Clock Tolerance

The time window size is not specified exactly in the question. A **60-second window** is consistent with "a few minutes" expiry when combined with a Â±1 or Â±2 window tolerance at the server:

- At 60-second windows with Â±1 tolerance: a code is valid for up to ~3 minutes after generation
- At 60-second windows with Â±2 tolerance: valid for up to ~5 minutes

The server accepts the code computed for `T-1`, `T`, and `T+1` to tolerate clock drift between the device and server. Outside this tolerance: code rejected.

**Why 8 digits and not 6?** 6 digits give 10⁶ = 1,000,000 possible codes. With account lockout after 5 wrong attempts, the per-attempt probability of guessing a valid code is 5/1,000,000 = 0.0005%. 8 digits give 10⁸ = 100,000,000 possible codes — a factor of 100 reduction in brute-force probability per attempt window. The additional two digits provide meaningful security margin, particularly as the window is a few minutes (giving the server a small but real exposure window).

#### Server Verification

The bank independently computes `HMAC-SHA256(K, T)` for the current and neighbouring windows, truncates to 8 digits. If the submitted code matches any valid window → access granted. After 5 failed attempts → account blocked (ch3.7 p.85 — threshold detection). After 3 wrong PINs → HardKey blocked (separate hardware-enforced counter on device).

#### Full Login Protocol (over TLS 1.3 as specified)

```
Step 1 — User → Bank (TLS):   { username }
         Bank identifies which K is associated with this username.

Step 2 — User:                presses OK on HardKey → selects "1.Login" → enters PIN
         HardKey verifies PIN internally (not forwarded to PC)
         Correct PIN → HardKey computes T = floor(unix_time / window)
                                          code = truncate(HMAC-SHA256(K, T), 8 digits)
         Code displayed on HardKey screen

Step 3 — User → Bank (TLS):   { code }    (manually typed from HardKey display)

Step 4 — Bank:                For each valid window T âˆˆ {current-1, current, current+1}:
                               candidate = truncate(HMAC-SHA256(K, T), 8 digits)
                               if code == candidate → authenticated; session established
                               if no match after 5 attempts → account blocked
```

### Part 4 — Vulnerability to Malware on the PC

**Malware cannot capture the PIN**: the PIN is entered on the HardKey's own dedicated keypad. The PC receives no PIN input whatsoever — a keylogger on the PC sees nothing.

**Malware cannot steal K**: K lives inside the HardKey's secure hardware. It is never transmitted to the PC, to any network interface, or to any storage accessible by the OS.

**Malware capturing the 8-digit code faces a narrow time window**: the code expires after a few minutes. Malware that batches and periodically exfiltrates captured data will almost certainly find the code already expired by the time it reaches the attacker. Opportunistic theft is much harder than with a static password.

**Residual risk — real-time man-in-the-browser:** a sophisticated malware that operates in real time can capture the 8-digit code the moment the user types it and immediately authenticate to the bank as the user — before the window expires. This attack succeeds within the validity window. The short window limits but does not eliminate this risk.

**Residual risk — real-time phishing proxy**: an attacker's fraudulent bank site collects the username and 8-digit code and relays them to the real bank in real time. From the real bank's perspective, a valid code was submitted. TLS server certificate verification (ch3.6 p.5) and user attention to the browser address bar prevent connecting to a fraudulent site. This is the same risk as case 6 (TOTP).

**Comparison with case 6 (TOTP on phone)**: case 25 and case 6 use the same underlying mechanism (HMAC-SHA256 of shared secret and time). The HardKey is slightly more resistant to phishing because the attacker cannot see which service the user is authenticating to (no username visible to an observer); the phone app shows the service name. Both are equally vulnerable to real-time relay.

**Comparison with case 21 (USB card reader)**: case 21 uses asymmetric challenge-response — the nonce changes per session and is bound to the specific server interaction. Case 25's time window is shared for the entire duration — any interaction within the validity window can use the code. Case 21 is strictly stronger for replay resistance; case 25 trades this strength for the ability to operate without any PC connection.

### Part 5 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Authentication mechanism | Time-based HMAC code — only option for disconnected device | ch3.1 p.3 | Challenge-response impossible (no incoming channel); counter-based needs sync; static code has no replay protection |
| MAC function | HMAC-SHA256(K, T) | ch2.2.3 p.63–66 | No length-extension vulnerability; SHA-256 hardware support; HMAC-SHA512 is overkill for 8-digit output |
| Secret key size | 256-bit K | ch2 PQCrypto p.16 | Long-lived device secret; Grover's reduces 128-bit to 64-bit effective — insufficient for years-long use |
| Output format | 8 decimal digits | ch3.7 p.85 | 10⁸ space; feasible for manual entry; factor 100 improvement over 6 digits |
| PIN protection | PIN on HardKey keypad, not PC keyboard | ch3.7 p.46 | Keylogger on PC cannot capture PIN; device useless to a thief without the PIN |
| Clock tolerance | Â±1 or Â±2 window tolerance at server | ch3.1 p.3 | Tolerates real-world clock drift; too wide a window reduces replay resistance |
| Brute-force protection | Account blocked after 5 wrong codes; HardKey blocked after 3 wrong PINs | ch3.7 p.85 | Two independent lockout mechanisms; online guessing infeasible |
| Transport | TLS 1.3, AES-128-GCM, ECDHE (as specified) | ch3.6 p.7–8, p.18, p.37 | Code confidential in transit; ECDHE forward secrecy; server certificate binds connection to real bank |

### Sources

- IS_UG_1_Introduction (p.10, p.15, p.22, p.42)
- IS_UG_2_2_2_SecM_AsymmEncr (p.7, p.13)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.63–66, p.85–87)
- IS_UG_2_2_SecM-adv-PQCrypto (p.16–17)
- IS_UG_3_1_Appl_Basics (p.3, p.7)
- IS_UG_3_2_Appl_AuthMeth (p.11)
- IS_UG_3_6_Appl_TLS (p.5, p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.46, p.85)

_Status: Complete_  
_Done by: William_
