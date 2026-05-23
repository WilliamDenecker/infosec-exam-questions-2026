# Question 22

**Is it possible efficiently to parallellise the encryption operation of a block cipher in CBC mode? If so, explain how. Otherwise, explain why it is not possible.**

**What about the decryption operation?**

## Answer

### CBC Mode — Encryption (ch2.2.3 p.45–52)

**CBC (Cipher Block Chaining) encryption** is defined as:
$$C_0 = IV, \quad C_i = E_K(M_i \oplus C_{i-1}) \quad \text{for } i = 1, 2, \ldots, m$$

**Can encryption be parallelised? No — encryption is strictly sequential.**

To compute $C_i$, the encryption function $E_K$ takes as input $(M_i \oplus C_{i-1})$. This requires $C_{i-1}$, which is the output of the previous step. The dependency chain is:
$$C_1 = E_K(M_1 \oplus IV)$$
$$C_2 = E_K(M_2 \oplus C_1) \quad \text{← requires } C_1$$
$$C_3 = E_K(M_3 \oplus C_2) \quad \text{← requires } C_2$$
$$\vdots$$

Each ciphertext block $C_i$ depends on $C_{i-1}$, which in turn depends on $C_{i-2}$, all the way back to $IV$. This is a **sequential dependency chain of depth $m$** — no block can be computed until all preceding blocks have been computed.

**Consequence**: no matter how many processors are available, CBC encryption produces one ciphertext block per step, sequentially. $P$ processors provide no speedup over 1 processor for CBC encryption.

**Contrast with CTR mode** (ch2.2.3 p.52–55): in CTR mode, $C_i = M_i \oplus E_K(IV + i)$. The input to $E_K$ for block $i$ depends only on $IV$ and $i$ (a fixed constant known before encryption begins) — not on any other block. All $m$ AES calls can proceed in parallel. This is precisely why CTR mode is preferred for high-throughput applications.

---

### CBC Mode — Decryption (ch2.2.3 p.45–52)

**CBC decryption** is defined as:
$$M_i = D_K(C_i) \oplus C_{i-1}, \quad C_0 = IV$$

**Can decryption be parallelised? Yes — decryption is fully parallelisable.**

For decryption, the input to $D_K$ for block $i$ is $C_i$ — a ciphertext block that was received and is known before decryption begins. The XOR with $C_{i-1}$ (also already known from the received ciphertext) is applied after the decryption call. Neither $D_K(C_i)$ nor $C_{i-1}$ depend on any other decryption step.

Each plaintext block is computed independently:
$$M_1 = D_K(C_1) \oplus IV \quad \text{(known: } C_1, IV)$$
$$M_2 = D_K(C_2) \oplus C_1 \quad \text{(known: } C_2, C_1)$$
$$M_3 = D_K(C_3) \oplus C_2 \quad \text{(known: } C_3, C_2)$$

All $m$ decryption operations can be dispatched simultaneously to $P$ processors, each computing $D_K(C_i)$ independently. After all $D_K$ calls complete, the XOR with $C_{i-1}$ is trivial.

**Speedup**: with $P$ processors, decryption time is approximately $m/P$ AES decryption calls (plus negligible XOR). Linear speedup in the number of processors, up to $P = m$.

**Why is decryption different from encryption?** In encryption, the ciphertext $C_{i-1}$ is not yet known when computing $C_i$ — it is being produced by the current computation. In decryption, all ciphertext blocks are already available (received as the input), so any block can be decrypted in any order. The direction of the data flow reverses, eliminating the sequential dependency.

---

### Summary

| Operation | Parallelisable? | Reason | Speedup with $P$ processors |
|---|---|---|---|
| CBC **encryption** | **No** | $C_i$ requires $C_{i-1}$; sequential chain of depth $m$ | None (time = $m$ AES calls regardless) |
| CBC **decryption** | **Yes** | $D_K(C_i)$ depends only on the already-received $C_i$; all inputs known upfront | $\approx P\times$ (time = $m/P$ AES calls) |

**Practical implication**: for applications where encryption throughput matters (e.g., bulk data encryption), CTR mode or GCM is preferred over CBC because both allow fully parallel encryption. CBC's sequential encryption is a fundamental performance limitation that cannot be overcome by adding hardware. Decryption (e.g., HTTPS content received from a server) benefits from CBC's parallelisable decryption, but encryption (e.g., server sending response) does not.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.45–52: CBC mode encryption and decryption; p.52–55: CTR mode for comparison)

_Status: Complete_  
_Done by: William_
