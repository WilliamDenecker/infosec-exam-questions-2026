# Question 12

**If you need a cryptographic hash function and you have to choose between SHA-2 and SHA-3, which algorithm would you choose and why?**

**What are the respective advantages and drawbacks of each algorithm?**

**What would determine your choice?**

## Answer

### Overview of SHA-2 and SHA-3 (ch2.2.3 p.30–45)

Both SHA-2 and SHA-3 are families of cryptographic hash functions that satisfy the core security properties — preimage resistance, second preimage resistance, and collision resistance (ch2.2.3 p.12–14). They differ fundamentally in design:

- **SHA-2** (SHA-256, SHA-384, SHA-512): based on the **Merkle-Damgård construction** (ch2.2.3 p.30–34). Iteratively applies a compression function to fixed-size blocks.
- **SHA-3** (SHA3-256, SHA3-512, SHAKE128, SHAKE256): based on the **Keccak sponge construction** (ch2.2.3 p.41–45). Absorbs input into a large internal state and "squeezes" output.

---

### SHA-2 — Advantages and Drawbacks

**Advantages**:

1. **Hardware acceleration**: modern CPUs include dedicated SHA-NI instructions (Intel SHA Extensions, ARM SHA2 instructions) that implement SHA-256 and SHA-512 natively in hardware. On these processors, SHA-256 operates at approximately 1–4 cycles per byte — extraordinarily fast. SHA-3 has no equivalent CPU-level hardware acceleration on most current processors.

2. **Deployment and compatibility**: SHA-256 has been in widespread use since the early 2000s. It is implemented in virtually every TLS library, PKI system, hardware security module, and secure element. Compatibility with legacy systems is guaranteed.

3. **Extensive cryptanalytic scrutiny**: SHA-256 has been intensively studied for over two decades without a successful attack on its security properties. No practical collision, preimage, or second-preimage attack exists.

4. **Performance in common cases**: on hardware with SHA-NI, SHA-256 is extremely fast — faster than SHA3-256 on the same hardware. For high-throughput applications (TLS, disk encryption, HMAC computation in bulk), SHA-2 is the performance winner.

**Drawbacks**:

1. **Length extension vulnerability** (ch2.2.3 p.34): the Merkle-Damgård construction leaks the internal state in the hash output. Given $H(M)$, an attacker can compute $H(M \| pad \| X)$ for any suffix $X$ without knowing $M$'s content beyond its length (see Question 10). This vulnerability requires HMAC to be used for MAC construction — $H(K \| M)$ directly is insecure. SHA-3 does not have this vulnerability.

2. **Same design lineage as SHA-1**: SHA-2 and SHA-1 share the same Merkle-Damgård structure. While SHA-1's compression function was attacked (practical collision attacks), SHA-2's compression function is significantly stronger. However, they are structurally similar — if a fundamental weakness in the Merkle-Damgård construction were discovered, SHA-2 would be affected alongside SHA-1. SHA-3's entirely different construction provides structural diversity (defence in depth).

3. **No extendable output**: SHA-2 produces fixed-length output (256 or 512 bits). It cannot be used as a variable-length pseudorandom generator (XOF — Extendable Output Function) without additional construction.

---

### SHA-3 — Advantages and Drawbacks

**Advantages**:

1. **No length extension vulnerability** (ch2.2.3 p.41–45): the sponge construction does not expose its internal state in the output. The rate and capacity are separate; the internal state after absorbing input is not recoverable from the output. $H(K \| M)$ is secure with SHA-3 (though using HMAC is still recommended practice). SHA-3 can be used directly for some MAC constructions without the HMAC wrapper.

2. **Structural independence from SHA-2**: SHA-3 (Keccak) was designed from scratch with no structural relationship to SHA-1 or SHA-2. If a class of attacks is developed against Merkle-Damgård constructions, SHA-3 remains unaffected. From a **defence-in-depth** perspective (ch3.7 p.46 — diversity), relying on two completely different hash constructions is stronger than relying on one.

3. **Extendable output functions (SHAKE128, SHAKE256)**: the sponge construction naturally supports variable-length output. SHAKE128 and SHAKE256 can produce any desired output length, making them suitable as pseudorandom number generators or key derivation functions without separate construction overhead.

4. **Built-in domain separation**: the sponge's rate/capacity parameters and output length are part of the function definition, providing natural domain separation between SHA3-256 and SHA3-512.

**Drawbacks**:

1. **No hardware acceleration on most current CPUs**: unlike SHA-2, there are no widely deployed CPU instructions for SHA-3. Software SHA-3 implementations must perform the Keccak permutation in general-purpose instructions, which is slower than hardware-accelerated SHA-256. On a typical CPU without specialised support, SHA3-256 is approximately 2–3× slower than SHA-256 with SHA-NI.

2. **Less deployment history**: SHA-3 was standardised by NIST in 2015 — significantly more recently than SHA-256 (2001). While it has been scrutinised through a decade of standardisation competition, its real-world deployment track record is shorter.

3. **Slightly larger state**: the Keccak sponge has a 1600-bit internal state — larger than SHA-256's 256-bit state — which has some memory implication in very constrained embedded systems.

---

### What Determines the Choice

**Use SHA-2 (SHA-256 or SHA-384) when**:

- **Performance is critical**: systems with SHA-NI instructions (modern servers, desktops, mobiles) should use SHA-256 for maximum throughput. This matters for TLS, disk encryption, software signing pipelines, and anywhere hashing is in the critical path.
- **Compatibility is required**: existing PKI, TLS stacks, hardware security modules, and legacy systems all support SHA-256. If the hash output must interoperate with existing systems, SHA-256 is the safe choice.
- **HMAC is used anyway**: if you are computing HMAC-SHA256 for MAC purposes, the length extension issue is already neutralised by the HMAC construction. SHA-2 with HMAC is secure and fast.
- **Digital signatures**: SHA-256 is specified in ECDSA, RSA-PSS, and all current TLS cipher suites — no reason to deviate from the standard.

**Use SHA-3 when**:

- **Length extension immunity matters without HMAC**: if the application computes $H(K \| M)$ directly (some embedded or resource-constrained contexts where HMAC overhead is undesirable), SHA-3's immunity to length extension attacks makes it intrinsically safer.
- **Algorithmic diversity is a goal**: for a new system that wants structural independence from the SHA-2 family (defence in depth), SHA-3 provides a different construction that would survive a hypothetical break of Merkle-Damgård structures.
- **Extendable output is needed**: SHAKE128 or SHAKE256 for variable-length pseudorandom output (key derivation, randomness expansion).
- **New protocol with no legacy constraints**: when designing a new protocol with no interoperability requirements, SHA-3 is a clean choice with better structural properties.

---

### My Recommendation

**For the vast majority of applications: SHA-256 (or SHA-384 for longer security margins).**

The practical benefits of SHA-2 — hardware acceleration, universal deployment, extensive analysis track record — outweigh the structural benefits of SHA-3 in typical use cases. When HMAC is used (the correct construction for MAC), SHA-2's length extension issue is not relevant.

**The exception**: use SHA-3 (SHAKE256) for key derivation or randomness expansion purposes where extendable output is needed, or for new cryptographic protocol designs where structural independence from SHA-2 is a design goal.

| Factor | Favours SHA-2 | Favours SHA-3 |
|---|---|---|
| CPU performance | Yes (SHA-NI acceleration) | No (software-only on most CPUs) |
| Length extension immunity | No (use HMAC to compensate) | Yes (intrinsic) |
| Structural diversity | No (same family as SHA-1) | Yes (completely different design) |
| Variable-length output | No (fixed output size) | Yes (SHAKE variants) |
| Compatibility / legacy | Yes | No |
| Deployment maturity | Yes (since 2001) | Less (since 2015) |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.30–45: Merkle-Damgård construction, SHA-2 properties, sponge construction, SHA-3; p.12–14: preimage and collision resistance)

_Status: Complete_  
_Done by: William_
