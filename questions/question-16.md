# Question 16

**Explain why AES-GCM (Galois Counter Mode) can be more efficiently parallellised than a combination of AES-CTR (counter mode) for confidentiality with an AES-based CBC-MAC for authentication.**

**Show as an illustration how the AES-GCM algorithm could be efficiently parallellised over 4 processors.**

*Note: consider the simplified case for GCM with a 96 bit IV and with no additional authenticated data. The message to be encrypted is at least a few kilobytes, which means that a reasonably large number of blocks will have to be encrypted.*

## Answer

### The Parallelism Problem in CTR + CBC-MAC (ch2.2.3 p.52–75)

**AES-CTR (Counter Mode)** (ch2.2.3 p.52–55):

In CTR mode, ciphertext block $i$ is computed as:
$$C_i = M_i \oplus E_K(IV \| i)$$

The keystream block $E_K(IV \| i)$ depends only on the key $K$, the IV, and the block index $i$ — it is **fully independent** of all other blocks. Therefore, all $m$ keystream blocks can be computed **in parallel** with no dependencies. CTR mode is maximally parallelisable for encryption.

**AES-CBC-MAC** (ch2.2.3 p.63–66):

CBC-MAC computes the authentication tag by iterating:
$$T_0 = 0^{128}, \quad T_i = E_K(C_i \oplus T_{i-1})$$

The authentication tag for block $i$ requires $T_{i-1}$ (output of the previous step). $T_{i-1}$ requires $T_{i-2}$, and so on. This is a **strict sequential dependency chain**: no block can be processed before the previous one completes. CBC-MAC is **inherently sequential** — it cannot be parallelised.

**Combined CTR + CBC-MAC parallelism**:

- CTR encryption: $P$ processors each handle $m/P$ blocks → fully parallel → time proportional to $m/P$ AES calls
- CBC-MAC: strictly sequential → time proportional to $m$ AES calls **regardless of processor count**

The CBC-MAC becomes the bottleneck. Adding more processors provides no speedup for the authentication part. The combined scheme cannot be efficiently parallelised because one of its two components is fundamentally sequential.

---

### Why AES-GCM Is More Efficient to Parallelise (ch2.2.3 p.70–75)

**GCM combines**:
1. **CTR mode for encryption**: $C_i = M_i \oplus E_K(J_0 + i)$ (where $J_0$ is derived from the 96-bit IV; $J_0 + i$ is the counter block for record $i$). Fully parallel — same as standalone CTR.
2. **GHASH for authentication**: $G_i = (G_{i-1} \oplus C_i) \cdot H$ (where $H = E_K(0^{128})$). Standard formulation is sequential.

**The key insight — GHASH can be parallelised**:

Although the standard GHASH formulation is sequential, the polynomial structure allows reorganisation into a parallel computation. The GHASH over $m$ ciphertext blocks is:
$$G = C_1 \cdot H^m \oplus C_2 \cdot H^{m-1} \oplus \ldots \oplus C_m \cdot H^1$$

This is a sum of independent terms: each term $C_i \cdot H^{m+1-i}$ depends only on $C_i$, $H$, and $i$ — not on any other term. The powers $H^j$ are all computable from $H = E_K(0^{128})$ (computed once at key setup). Once $C_i$ is known, all terms can be computed in **parallel**, then XOR'd together.

**Contrast with CBC-MAC**: CBC-MAC is $E_K(C_m \oplus E_K(C_{m-1} \oplus \ldots))$ — a chain of nested function evaluations. This cannot be expressed as a sum of independent terms because each encryption call is applied to the output of the previous encryption. No algebraic reorganisation eliminates the sequential dependency.

**Summary of parallelism comparison**:

| Component | CTR + CBC-MAC | GCM |
|---|---|---|
| Encryption (CTR) | Fully parallel | Fully parallel |
| Authentication | Sequential (CBC-MAC, $O(m)$ sequential AES) | Parallelisable ($m$ independent GF multiplications + XOR tree) |
| Combined | Bottlenecked by CBC-MAC | Both components parallelisable |
| AES calls for auth | $m$ sequential AES encryptions | 1 AES call ($H = E_K(0)$, pre-computed) + $m$ GF multiplications |

Additionally, GCM's authentication uses GF(2^{128}) **multiplication** (exploiting hardware CLMUL instructions on modern CPUs) rather than AES block cipher calls. GF multiplication is faster per operation than AES encryption, further reducing the authentication bottleneck.

---

### Parallelisation of AES-GCM over 4 Processors

Setup (performed once, shared across all processors):
- Compute $H = E_K(0^{128})$ (one AES call, on any one processor)
- Precompute powers: $H^2, H^3, H^4$ (3 GF multiplications, fast)
- Derive counter base: $J_0 = IV \| \texttt{0x00000001}$ (for 96-bit IV in GCM)

**Message of $m$ blocks, divided into 4 equal groups** (assume $m = 4k$ for simplicity):

| Processor | CTR blocks | GHASH blocks | GHASH term structure |
|---|---|---|---|
| P1 | Blocks $1 \ldots k$ | Blocks $1 \ldots k$ | $C_i \cdot H^{m+1-i}$ for $i = 1 \ldots k$ |
| P2 | Blocks $k+1 \ldots 2k$ | Blocks $k+1 \ldots 2k$ | $C_i \cdot H^{m+1-i}$ for $i = k+1 \ldots 2k$ |
| P3 | Blocks $2k+1 \ldots 3k$ | Blocks $2k+1 \ldots 3k$ | $C_i \cdot H^{m+1-i}$ for $i = 2k+1 \ldots 3k$ |
| P4 | Blocks $3k+1 \ldots 4k$ | Blocks $3k+1 \ldots 4k$ | $C_i \cdot H^{m+1-i}$ for $i = 3k+1 \ldots 4k$ |

**On each processor $j$ (simultaneously)**:

1. Compute $k$ keystream blocks: $E_K(J_0 + (j-1)k + 1), \ldots, E_K(J_0 + jk)$ (these are independent AES evaluations — fully parallel with other processors)
2. XOR with plaintext: $C_i = M_i \oplus E_K(J_0 + i)$ for each block in the group
3. Compute partial GHASH: for each $C_i$ in the group, compute $C_i \cdot H^{m+1-i}$ (GF multiplication using precomputed $H^j$ values)
4. XOR the $k$ partial GHASH terms together to produce a partial sum $G_{partial,j}$

**Final combination** (on any one processor, or dedicated coordinator):

$$G_{final} = G_{partial,1} \oplus G_{partial,2} \oplus G_{partial,3} \oplus G_{partial,4}$$

This XOR tree takes 3 operations and is negligible.

**Authentication tag**:
$$T = E_K(J_0) \oplus G_{final}$$
(one additional AES call for the tag counter block, also pre-computable)

**Result**: the total time over 4 processors is approximately $m/4$ AES calls + $m/4$ GF multiplications (for the partial GHASH in each processor group) + a small constant for the final XOR and tag computation. Speedup is approximately $4\times$ compared to single-processor.

**Why CTR + CBC-MAC cannot achieve the same**:

With CTR + CBC-MAC over 4 processors: the CTR part runs in parallel (time = $m/4$ AES calls). The CBC-MAC runs sequentially on one processor (time = $m$ AES calls). The combined time is dominated by the sequential CBC-MAC: total = $m$ AES calls. Adding more processors provides **zero speedup** for the authentication — the bottleneck is fixed. AES-GCM over 4 processors completes in approximately $m/4$ steps; CTR + CBC-MAC completes in approximately $m$ steps regardless of processor count.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.52–55: CTR mode — parallel keystream generation; p.63–66: CBC-MAC — sequential dependency chain; p.70–75: GCM — GHASH polynomial structure, parallelisability, and combined AEAD operation)

_Status: Complete_  
_Done by: William_
