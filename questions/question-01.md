# Question 1

OCB (Offset Codebook Mode) is an authenticated-encryption scheme (as is GCM). It offers both confidentiality and authentication using a symmetric encryption algorithm (block cipher, typically AES). It is also provably secure.

The (somewhat simplified) operation on a message $M$ using a symmetric encryption key $K$ and an encryption algorithm $E$ (AES) is as follows:

1. break the message $M$ into $m$ 128 bit blocks: $M_1, M_2, \ldots, M_m$
2. compute $L_* = E_K(0^{128})$ (where $0^{128}$ is a string of 128 null bits)
3. compute $L_\$ = x \cdot L_*$
4. compute $L[0] = x \cdot L_\$$ and $L[j] = x \cdot L[j-1]$ (for $j$ ranging from 1 to 127)
5. compute $Z_0 = Init(N)$, where $N$ is a nonce for the algorithm and $Init$ is an easily computed initialisation function (for which details are skipped here)
6. the offsets $Z_i$ ($i$ ranging from 1 to $m$) can be computed as $Z_i = Z_{i-1} \oplus L[ntz(i)]$, where $\oplus$ is the bitwise XOR operation and $ntz(i)$ is the number of trailing zeros in the binary representation of $i$ (e.g. $ntz(12) = 2$)
7. compute the ciphertext blocks $C_i = Z_i \oplus E_K(M_i \oplus Z_i)$
8. compute $Checksum = M_1 \oplus \ldots \oplus M_m$
9. compute the authentication tag $T = E_K(Checksum \oplus Z_m \oplus L_\$)$
10. the final outcome is $CT = C_1, C_2, \ldots, C_m, T$.

All multiplications ($x \cdot$) should be considered as a (polynomial) multiplication with the polynomial $x$ in $GF(2^{128})$ using the irreducible polynomial $x^{128} + x^7 + x^2 + x + 1$ (cf. GCM).

We have hereby assumed that the length (in bits) of message $M$ is a multiple of 128, that there are no associated data (that must be authenticated, but not encrypted).

**Compare the performance of OCB to the performance of GCM: consider parallellisability of encryption, complexity of operations, and potential to pre-compute some partial results.**

## Answer

### Parallelisability of Encryption

**OCB encryption** (step 7): $C_i = Z_i \oplus E_K(M_i \oplus Z_i)$. The offsets $Z_i$ form a sequential chain — $Z_i = Z_{i-1} \oplus L[ntz(i)]$ — but each step is a single XOR operation, negligibly fast. Once all $Z_i$ are known, the computation of each $C_i$ depends only on $M_i$ and $Z_i$, not on any other ciphertext block. The heavy operation — the AES block cipher evaluation $E_K(M_i \oplus Z_i)$ — is therefore **fully parallelisable**: all $m$ AES calls can be dispatched simultaneously to parallel hardware (multiple AES cores, or pipelined AES-NI instructions) (ch2.2.3 p.70).

**OCB authentication** (steps 8–9): $Checksum = M_1 \oplus \ldots \oplus M_m$. The XOR of all plaintext blocks is a simple **parallel reduction** (XOR tree) followed by a single AES call for the tag. Authentication therefore adds only $O(\log m)$ XOR depth plus one AES evaluation.

**GCM encryption** (ch2.2.3 p.70–75): uses CTR mode — $C_i = M_i \oplus E_K(IV \| i)$. The counter blocks $E_K(IV \| i)$ are independent of each other and of the ciphertext, so GCM encryption is also **fully parallelisable**.

**GCM authentication** (ch2.2.3 p.70–75): uses GHASH — a polynomial evaluation over $GF(2^{128})$ defined as:
$$GHASH = C_1 \cdot H^m \oplus C_2 \cdot H^{m-1} \oplus \ldots \oplus C_m \cdot H$$
Each term depends on its ciphertext block and the authentication key $H$. In the standard sequential formulation, GHASH is computed one block at a time: $G_i = (G_{i-1} \oplus C_i) \cdot H$, which is **sequential** — each step depends on the previous accumulator. Hardware implementations can partially pipeline this using Karatsuba multiplication, but the dependency chain is fundamentally longer than OCB's XOR checksum.

**Summary — parallelisability**: Both schemes parallelise their encryption passes equally well. OCB's authentication (XOR checksum + 1 AES call) is inherently more parallelisable than GCM's GHASH (sequential GF multiplication chain).

---

### Complexity of Operations

**Full operation count — OCB** ($m$ blocks):
- $m$ AES block-cipher invocations for encryption: $E_K(M_i \oplus Z_i)$
- $m$ XOR-with-offset operations: $C_i = Z_i \oplus E_K(\ldots)$
- $m$ polynomial multiplications over $GF(2^{128})$ to generate offsets: $Z_i = Z_{i-1} \oplus L[ntz(i)]$, where each $L[j] = x \cdot L[j-1]$ is a multiplication by the simple polynomial $x$
- 1 final AES call for the authentication tag $T = E_K(\text{Checksum} \oplus Z_m \oplus L\$)$

**Full operation count — GCM** ($m$ blocks, ch2.2.3 p.70–75):
- $m$ AES invocations for CTR-mode encryption: $C_i = M_i \oplus E_K(IV \| i)$
- $m$ multiplications in $GF(2^{128})$ for GHASH authentication: $Y_i = (Y_{i-1} \oplus C_i) \cdot H$
- 1 extra AES call for the initial hash key $H = E_K(0^{128})$
- 1 final AES call for the MAC tag

Both schemes thus perform $m$ AES block-cipher calls plus $m$ operations in $GF(2^{128})$. However, **the nature of the $GF(2^{128})$ operations differs fundamentally**:

- In **OCB**, the offset multiplications ($x \cdot L[j]$) are multiplications by the simple polynomial $x$. In $GF(2^{128})$ with irreducible polynomial $x^{128} + x^7 + x^2 + x + 1$, multiplying by $x$ reduces to a **single left-shift and a conditional XOR** with the irreducible polynomial (if the high bit was set). This is an extremely cheap operation — 2–3 instructions.

- In **GCM**, GHASH multiplications are **general field multiplications** by a fixed $H = E_K(0^{128})$. A general $GF(2^{128})$ multiplication involves 128 iterations of shift-and-XOR, or equivalently, carryless multiply (CLMUL) followed by modular reduction. This is computationally much heavier than a multiply-by-$x$.

**Per-block cost breakdown**:

| Operation | OCB | GCM |
|---|---|---|
| AES call | 1 × $E_K$ | 1 × $E_K$ |
| XOR | 3 (input mask, output mask, checksum) | 2 (CTR XOR + GHASH XOR) |
| $GF(2^{128})$ mult | 1 × **multiply-by-$x$** (shift + cond. XOR) | 1 × **full multiply-by-$H$** (CLMUL / 128 iterations) |

**Conclusion**: OCB's $GF(2^{128})$ operations are significantly cheaper than GCM's GHASH multiplications. OCB has a lower total computational complexity per block. On CPUs with CLMUL hardware instructions (Intel/AMD since ~2010), GCM's general multiplication is accelerated to ~a few cycles, narrowing the gap. On constrained hardware lacking CLMUL (microcontrollers, embedded systems), GCM's per-block GHASH multiplication requires software emulation (many cycles), whereas OCB's multiply-by-$x$ remains trivially cheap regardless.

**OCB setup operations** (key-dependent, one-time): $L_* = E_K(0^{128})$, then $L_\$$, $L[0], \ldots, L[127]$ via repeated multiply-by-$x$ operations. Done once per key — the more expensive setup pays off over many messages.

**GCM setup** (one-time): $H = E_K(0^{128})$ — one AES call. No precomputed table equivalent to OCB's $L$ table; GHASH multiplications by $H$ recur per block.

---

### Pre-computation Potential

**OCB pre-computation**:

1. **Key-dependent values** ($L_*$, $L_\$$, $L[0] \ldots L[127]$): depend only on $K$. These 130 values can be computed once when the key is established and cached. No per-message computation needed for the $L$ table.

2. **Nonce-dependent offsets** ($Z_0, Z_1, \ldots, Z_m$): once the nonce $N$ is known, $Z_0 = Init(N)$ and the entire offset sequence $Z_1 \ldots Z_m$ can be computed before the first message block is available. Since nonces are typically chosen at the start of message processing, all offset values are available before message data arrives. This means the XOR inputs for all $m$ block cipher calls are pre-computable.

3. **Block cipher calls**: $E_K(M_i \oplus Z_i)$ requires $M_i$, so these cannot be pre-computed before the message is known. However, the inputs $Z_i$ (one half of the XOR) are ready in advance.

**GCM pre-computation** (ch2.2.3 p.70–75):

1. **Key-dependent value** $H = E_K(0^{128})$: computed once at key setup. All GHASH multiplications use $H$ as a fixed multiplier — this can be combined into a lookup table (Shoup's method) for faster per-block GHASH.

2. **Nonce-dependent counter blocks**: $E_K(IV \| 0), E_K(IV \| 1), \ldots, E_K(IV \| m)$ depend only on $K$ and $IV$ (the nonce). If the nonce is known before message data arrives, all counter blocks can be pre-encrypted, making the XOR with message blocks trivial once data arrives.

**Comparison**: Both schemes allow meaningful pre-computation at the key level and at the nonce level. OCB's additional advantage is that the $L$ table (128 values, computed once per key) replaces GF multiplications entirely in the per-block computation, whereas GCM requires $H$-based GF multiplications per block even with pre-computation of $H$.

---

### Summary Table

| Criterion | OCB | GCM (ch2.2.3 p.70–75) |
|---|---|---|
| Encryption parallelisability | Full (AES calls independent once $Z_i$ known) | Full (CTR mode, independent counter blocks) |
| Authentication parallelisability | Full (XOR tree + 1 AES call) | Sequential (GHASH dependency chain) |
| Per-block operations | 1 AES + 3 XOR | 1 AES + 2 XOR + 1 GF(2^{128}) mult |
| GF(2^{128}) mult per block | **None** (only at key setup) | **Yes** (one per block in GHASH) |
| Key-level pre-computation | L table (128 values) | $H = E_K(0)$ |
| Nonce-level pre-computation | Full offset chain $Z_0 \ldots Z_m$ | Full counter block sequence |
| AES-D required? | Yes (for decryption path) | No (CTR mode uses only AES-E) |

**Conclusion**: OCB is generally more efficient per block (no GF multiplication per block) and more parallelisable in its authentication path. GCM's advantage is that it uses only AES-E operations (even for decryption via CTR mode) and benefits from hardware CLMUL support. GCM also avoids the patent issues that historically made OCB less deployable, which is the primary reason GCM became the de facto standard despite OCB's theoretical performance advantages.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.70–75: GCM authenticated encryption, parallelisability)

_Status: Complete_  
_Done by: William_
