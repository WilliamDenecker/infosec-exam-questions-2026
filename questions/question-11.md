# Question 11

A basic password storage mechanism is to encode a combination of the password and a salt using a one-way function.

A commonly used one-way function is MD5. An alternative one-way function is bcrypt, in which a lengthy (and configurable) setup process may increase the required computation time for the output of the one-way function up to 100 ms on a regular PC.

**What is the advantage (consider performance and security) of using bcrypt instead of MD5 for password storage? Are there also possible drawbacks (consider again performance and security) in using bcrypt instead of MD5? Consider the case of a smartphone, the case of a PC, and the case of a server.**

*Note: the precise working of bcrypt is not relevant for answering this question.*

## Answer

### Context: Password Storage with One-Way Functions (ch3.2 p.11)

When storing passwords, the server stores not the password itself but $H(salt \| password)$ (ch3.2 p.11). At login, the server recomputes the hash and compares. The salt prevents precomputed dictionary attacks (rainbow tables): an attacker who obtains the hash database must crack each hash individually rather than looking up precomputed hashes.

The security of this approach depends critically on how hard it is to compute the hash function. If hash computation takes 1 nanosecond (MD5 on modern hardware), an attacker can try billions of passwords per second. If it takes 100 ms (bcrypt), the attacker can try at most 10 per second per core.

---

### MD5 Performance Characteristics

MD5 was designed for speed (ch2.2.3 p.30). On a modern CPU, a single MD5 computation takes on the order of microseconds. On a modern GPU (or custom hardware), billions of MD5 hashes can be computed per second. This makes MD5 completely unsuitable for password storage in any modern security context, regardless of the device.

---

### Advantages of bcrypt over MD5

**Security advantage — brute-force resistance (all devices)**:

bcrypt's configurable work factor means each password attempt takes ~100 ms on a PC. An attacker with a stolen hash database can attempt:
- With MD5: ~10 billion attempts per second (GPU-accelerated) → a dictionary of 100 million common passwords is exhausted in ~10 milliseconds
- With bcrypt: ~10 attempts per second per core → the same dictionary takes ~10 million seconds (~115 days) per core

This is not a marginal difference — it is a factor of approximately 10^9. Even a dedicated cluster of GPUs gains very little against bcrypt because bcrypt is specifically designed to resist GPU parallelism (its memory access patterns are not efficiently parallelisable by GPU hardware).

**Security advantage — configurable cost (all devices)**:

As hardware becomes faster, the bcrypt work factor can be increased to maintain constant computation time. MD5's speed is fixed by its algorithm — it can only get faster as hardware improves.

**Security advantage — resistance to dedicated hardware (ASICs)**:

bcrypt uses the Blowfish cipher internally, which has a large key schedule that must be computed during setup. This requires significant memory (contrary to Question 3's ASIC concern, bcrypt is more resistant to ASICs than MD5 though less so than fully memory-hard functions).

---

### Drawbacks of bcrypt vs MD5

#### Case 1 — Smartphone

**Advantage of bcrypt on smartphones**: the same brute-force resistance as on PC. If a smartphone stores hashed passwords locally (e.g., a local app requiring login), bcrypt protects against offline attacks if the device is seized.

**Drawback of bcrypt on smartphones**:
- **Battery consumption**: bcrypt's 100 ms computation requires sustained CPU activity. On a smartphone, 100 ms of active CPU at full clock speed consumes significant battery — not a problem for a single login, but problematic for applications that re-authenticate frequently (e.g., screen lock, per-transaction PIN verification in a payment app). MD5 would consume negligible power by comparison.
- **Performance perception**: if the app authenticates on every screen unlock, a 100 ms delay is barely perceptible. If it re-authenticates for every action, the accumulated latency becomes annoying and drains battery.
- **CPU throttling**: some mobile chipsets throttle CPU performance during sustained load or when battery is low — bcrypt may take significantly longer than 100 ms under throttled conditions, creating variable (sometimes very long) login times.

**Net assessment for smartphone**: bcrypt is the correct choice for login — security far outweighs the ~100 ms overhead. Drawbacks become significant only for high-frequency re-authentication patterns.

#### Case 2 — PC

**Advantage of bcrypt on PCs**: full brute-force resistance. Even if an attacker obtains the password hash file (from a local breach or backup), cracking a bcrypt-hashed password is computationally infeasible within any reasonable timeframe.

**Drawback of bcrypt on PCs**:
- **Essentially no practical drawback for authentication**: 100 ms is imperceptible to a user entering a password. PCs have abundant power and CPU resources.
- **Security drawback (subtle)**: bcrypt has a maximum input length of 72 bytes. Passwords longer than 72 bytes are silently truncated — two passwords with the same first 72 characters produce the same hash. This is an implementation-specific limitation not related to MD5, but a potential subtle security issue if users set very long passwords.

**Net assessment for PC**: bcrypt is straightforwardly the correct choice. The only concern (72-byte truncation) can be mitigated by pre-hashing the password before passing it to bcrypt.

#### Case 3 — Server (the most nuanced case)

**Advantage of bcrypt on servers**: same security benefit — attacker who obtains the database cannot crack passwords efficiently.

**Drawback of bcrypt on servers**:

1. **Throughput bottleneck**: if a server handles $N$ simultaneous login requests, each requiring 100 ms of CPU time, it can serve at most approximately $10 \times \text{CPU cores}$ login requests per second. For a high-traffic authentication service handling thousands of logins per second, bcrypt creates a **severe CPU bottleneck**. MD5 imposes no such constraint (millions of logins per second per core). Bcrypt effectively limits login throughput to $\sim10/\text{core}$ per second.

2. **Denial-of-service vulnerability**: an attacker can deliberately trigger many failed login attempts with crafted usernames. Each attempt forces the server to compute bcrypt (100 ms of CPU). An attacker sending 1000 simultaneous fake login attempts forces 100 seconds of cumulative CPU time — exhausting server resources without requiring any special hardware. This is a **CPU exhaustion DoS** attack that MD5 (negligible computation) would not enable. Rate limiting and CAPTCHAs mitigate this, but they add complexity.

3. **Scaling cost**: for high-traffic systems, the CPU cost of bcrypt authentication increases cloud infrastructure costs proportionally. In a system where authentication is in the critical path (e.g., API gateway), this has direct financial cost.

**Mitigations on servers**: use a separate authentication microservice with bcrypt, rate-limit login attempts (ch3.7 p.85 — IDS threshold detection for brute-force), apply bcrypt work factor to the minimum sufficient for security rather than maximising it, and cache successful authentication tokens (so users don't re-authenticate on every request).

**Net assessment for server**: bcrypt is still the correct choice for security — the alternatives (fast MD5) are catastrophically insecure. But the throughput and DoS implications must be explicitly engineered around. The drawbacks are real operational concerns, not theoretical ones.

---

### Summary Table

| Context | Advantage of bcrypt over MD5 | Drawback of bcrypt |
|---|---|---|
| All | 10^9× harder to brute-force offline | Computationally more expensive |
| Smartphone | Protects against offline attacks on seized device | Battery drain for frequent re-auth; CPU throttling risk |
| PC | Full brute-force protection; negligible user impact | 72-byte truncation (minor); essentially no performance drawback |
| Server | Offline breach protection for password database | Login throughput limited (~10/core/sec); CPU exhaustion DoS risk |

### Sources

- IS_UG_3_2_Appl_AuthMeth (p.11: password storage with salted hash; cost-adjustable one-way functions for password hashing)
- IS_UG_2_2_3_SecM_HashMac (p.30: MD5 and SHA speed characteristics)
- IS_UG_3_7_Appl_System (p.85: brute-force detection and rate limiting)

_Status: Complete_  
_Done by: William_
