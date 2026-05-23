# Question 17

Consider the following implementation of a hash function $H_T$

$$H_T(M) = H(M_1) \oplus \ldots \oplus H(M_n) \quad (Q17.3)$$

where $H$ is a traditional hash function (MD5, SHA1, SHA2-256, etc.), where $M$ is the message to be hashed, and $M$ is the concatenation of $n$ individual data blocks $M_i$ (with $i \in 1..n$), which are sufficiently small to be a single input data block for the hash function $H$:

$$M = M_1\|\ldots\|M_n \quad (Q17.4)$$

**What are possible advantages of $H_T$? What are possible security issues? Is it still a one-way function, does it still exhibit weak and strong collision resistance (consider the case where $H$ is MD5 and the case where $H$ is SHA2-256)?**

## Answer

### Overview

$H_T(M) = H(M_1) \oplus H(M_2) \oplus \ldots \oplus H(M_n)$ computes $H$ on each individual block of $M$ and XORs the results together. This is a parallelisable construction whose security properties differ significantly from those of a standard monolithic hash over the complete message.

---

### Possible Advantages

**1. Parallelisability**: each $H(M_i)$ is independent of all others. With $P$ processors, all $n$ hash computations can be dispatched simultaneously, and the results XORed. For large messages with many blocks, this gives an approximately $P$-fold speedup in the hash computation compared to a sequential Merkle-Damgård hash over the full message. This could be relevant for hashing large files across multi-core hardware.

**2. Incremental hashing**: if one block $M_i$ changes (e.g., a file is partially updated), the new $H_T$ can be computed by recomputing $H(M_i')$ for the changed block only and XORing it in: $H_T^{new} = H_T^{old} \oplus H(M_i) \oplus H(M_i')$. This is $O(1)$ hash calls per changed block, compared to $O(n)$ for a full re-hash.

**3. Streaming computation**: blocks can be hashed as they arrive, and the running XOR maintained. The final result is available immediately when the last block is processed, without needing to buffer the entire message.

---

### Security Issues

#### Issue 1 — No Ordering / Permutation Attack (ch2.2.3 p.14 — collision resistance)

XOR is **commutative and associative**. Therefore:
$$H_T(M_1 \| M_2 \| \ldots \| M_n) = H_T(M_{\pi(1)} \| M_{\pi(2)} \| \ldots \| M_{\pi(n)})$$
for any permutation $\pi$. Any reordering of the message blocks produces an identical hash.

**Consequence**: an attacker can freely **reorder** any blocks of the message without changing $H_T$. If $H_T$ is used to verify the integrity of a message, an adversary can shuffle blocks at will. This is a fundamental structural weakness — not dependent on the strength of $H$.

This is not a collision resistance property failure in the traditional sense; it is a property that $H$ alone does not have (standard $H$ would produce a different output for any permutation of input). $H_T$ introduces this weakness by design.

#### Issue 2 — Block Substitution Attack (ch2.2.3 p.14)

Given two messages $M$ and $M'$ with $n$ blocks each, if for some pair of blocks $M_i, M_j$ there exist $M_i', M_j'$ such that:
$$H(M_i') \oplus H(M_j') = H(M_i) \oplus H(M_j)$$

then $H_T(M_1 \| \ldots \| M_i' \| \ldots \| M_j' \| \ldots \| M_n) = H_T(M)$.

Finding such pairs requires finding $H(M_i') \oplus H(M_j') = \text{const}$, which is easier than finding a single collision in $H$ and dramatically easier than finding a second-preimage in $H_T$ for the full message.

#### Issue 3 — Output Size Unchanged But Effective Security Reduced

If $H = SHA256$ (256-bit output), $H_T$ also produces 256-bit output — one might assume similar security. However, the structural weaknesses above mean $H_T$ provides far weaker integrity guarantees than the underlying $H$ applied to the full message, even with the same output length.

---

### Is $H_T$ a One-Way Function? (ch2.2.3 p.12)

**One-wayness** (preimage resistance) requires: given $y = H_T(M)$, it is computationally infeasible to find any $M'$ such that $H_T(M') = y$.

**With $H$ = SHA256**: $H(M_i)$ individually is one-way (SHA-256 is preimage-resistant). However, $H_T(M) = \bigoplus_{i=1}^n H(M_i)$ is a XOR of $n$ independent SHA-256 outputs. To find a preimage, an attacker needs to find $M_1', \ldots, M_n'$ such that $\bigoplus H(M_i') = y$. For $n \geq 2$: choose $M_1', \ldots, M_{n-1}'$ freely (giving $H(M_1') \oplus \ldots \oplus H(M_{n-1}') = z$ for some $z$), then need $H(M_n') = y \oplus z$. This is a preimage attack on SHA-256 for a chosen target — still computationally infeasible if SHA-256 is preimage-resistant.

**Conclusion**: $H_T$ remains one-way when $H$ is preimage-resistant (SHA-256). Finding a preimage of $H_T$ requires finding a preimage of $H$ for an arbitrary target, which is as hard as a preimage attack on $H$.

With $H$ = MD5: MD5 is no longer considered preimage-resistant in practice (though direct preimage attacks are not as practical as collision attacks). The conclusion is weakened accordingly.

**Caution**: $H_T$ accepts any partition of a preimage into $n$ blocks. An attacker can choose $M_1' = M_1, \ldots, M_{n-1}' = M_{n-1}$ (same as original blocks) and only needs $H(M_n') = H(M_n)$ — a second-preimage attack on a single block. For the full message, one-wayness of $H_T$ reduces to second-preimage resistance of $H$ for individual blocks.

---

### Weak Collision Resistance (Second Preimage Resistance)? (ch2.2.3 p.13)

**Second preimage resistance** requires: given $M$, it is infeasible to find $M' \neq M$ such that $H_T(M') = H_T(M)$.

**Structural weakness — permutation**: any permutation of $M$'s blocks immediately gives $H_T(M') = H_T(M)$ (since XOR is commutative). For $n \geq 2$ blocks, the trivially constructed second preimage $M' = M_2 \| M_1 \| M_3 \| \ldots \| M_n$ (swap first two blocks) has $H_T(M') = H_T(M)$.

**Conclusion**: $H_T$ does **NOT** exhibit second preimage resistance for any message with more than one block ($n \geq 2$). This holds regardless of whether $H$ is MD5 or SHA-256. The XOR structure inherently allows block permutation, making $H_T$ trivially second-preimage-attackable by reordering.

---

### Strong Collision Resistance? (ch2.2.3 p.14)

**Collision resistance** requires: it is computationally infeasible to find any $M \neq M'$ such that $H_T(M) = H_T(M')$.

**Trivial collision from permutation**: for any $M$ with $n \geq 2$ distinct blocks, $M' = M_2 \| M_1 \| M_3 \| \ldots \| M_n$ is a different message with $H_T(M') = H_T(M)$. No computational effort required.

**Conclusion**: $H_T$ does **NOT** exhibit collision resistance for messages with $n \geq 2$ blocks. Trivial collisions exist by block reordering, regardless of the strength of $H$ (MD5 or SHA-256).

**With $H$ = MD5**: additionally, practical collision attacks exist for MD5 (ch2.2.3 p.14 — MD5 collision attacks), so collisions in $H_T$ can be found through MD5 collisions in individual blocks as well.

**With $H$ = SHA-256**: SHA-256 is collision-resistant, but the XOR structure of $H_T$ means collisions exist trivially from permutation. The security of the underlying $H$ is irrelevant for this vulnerability.

---

### Summary Table

| Property | $H_T$ with SHA-256 | $H_T$ with MD5 | Standard $H$ (SHA-256) |
|---|---|---|---|
| One-wayness (preimage) | Yes (reduces to SHA-256 preimage) | Weakened | Yes |
| Second preimage resistance | **No** (block swap is a free second preimage) | **No** | Yes |
| Collision resistance | **No** (block swap gives trivial collision) | **No** (worse: also MD5 collisions) | Yes |
| Permutation invariance | Yes (structural weakness) | Yes (structural weakness) | No |
| Parallelisable | Yes (advantage) | Yes (advantage) | No (sequential) |
| Incremental update | Yes (advantage) | Yes (advantage) | No |

**Conclusion**: $H_T$ is not a secure cryptographic hash function. The XOR combination of block hashes destroys both second-preimage resistance and collision resistance through trivial block reordering. It retains one-wayness (assuming $H$ is preimage-resistant) and offers parallelism advantages, but these advantages are far outweighed by the structural security failures. $H_T$ must not be used in applications requiring collision resistance or second-preimage resistance.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.12: preimage resistance; p.13: second preimage resistance; p.14: collision resistance and attacks; p.30–40: hash function construction properties)

_Status: Complete_  
_Done by: William_
