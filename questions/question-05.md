# Question 5

The CBC-MAC construction builds a MAC function from a block cipher by taking the last encrypted block of the CBC-mode encryption.

**Can a similar (secure) MAC function be derived from OFB-mode encryption? Explain how this derived function would work or why it would not be secure.**

## Answer

### How CBC-MAC Works (ch2.2.3 p.63–66)

In CBC-mode encryption, each block is XORed with the previous ciphertext block before encryption:
$$C_0 = IV, \quad C_i = E_K(M_i \oplus C_{i-1})$$

where **IV** (Initialisation Vector) is the fixed starting value that seeds the chain before the first block. For CBC-MAC, $IV = 0^{128}$ (all zeros) — a known, public constant. This ensures the MAC is deterministic and verifiable.

CBC-MAC takes the last encrypted block $C_m$ as the authentication tag. This is secure because the chaining means $C_m$ depends on every plaintext block $M_1, M_2, \ldots, M_m$: any change to any block propagates through all subsequent ciphertext blocks, changing $C_m$. The MAC is a one-way function of the complete message — an attacker who changes any $M_i$ cannot produce the correct $C_m$ without the key $K$.

---

### OFB Mode Operation

In OFB (Output Feedback) mode (ch2.2.3 p.52–55), the block cipher generates a **keystream** that is independent of the message:
$$S_1 = E_K(IV), \quad S_i = E_K(S_{i-1})$$
$$C_i = M_i \oplus S_i$$

The critical property: $S_i$ depends only on $K$ and $IV$, **not on any message block** $M_j$. The keystream $S_1, S_2, \ldots, S_m$ is entirely pre-determined by $K$ and $IV$ before any plaintext is seen.

---

### Can a MAC Be Derived from OFB? — Analysis

There are several natural candidates for an OFB-based MAC. All fail:

**Candidate 1: Take the last ciphertext block $C_m$**

$C_m = M_m \oplus S_m$. Since $S_m$ is fixed (depends only on $K$, $IV$), $C_m$ depends only on $M_m$ and $S_m$ — **not on $M_1, \ldots, M_{m-1}$**.

**Attack**: an attacker can freely modify any of $M_1, M_2, \ldots, M_{m-1}$ without changing $C_m$. This trivially breaks authentication for all but the last block. Completely insecure.

**Candidate 2: Take the last keystream block $S_m$**

$S_m$ depends only on $K$ and $IV$ — it does not depend on any message block at all. This is not a MAC in any sense; it authenticates nothing about the message content.

**Candidate 3: XOR all ciphertext blocks $C_1 \oplus C_2 \oplus \ldots \oplus C_m$**

$$\bigoplus_{i=1}^m C_i = \bigoplus_{i=1}^m (M_i \oplus S_i) = \left(\bigoplus_{i=1}^m M_i\right) \oplus \left(\bigoplus_{i=1}^m S_i\right)$$

The keystream XOR $\bigoplus S_i$ is a constant (fixed for given $K$, $IV$). So the MAC reduces to $(\bigoplus M_i) \oplus \text{const}$.

**Attack**: XOR is commutative and associative — swapping any two message blocks $M_i$ and $M_j$ leaves the XOR sum $\bigoplus M_i$ unchanged, and therefore leaves this "MAC" unchanged. An attacker can reorder all blocks freely without detection. Also, any pair of blocks $M_i, M_j$ can be replaced by $M_i' = M_i \oplus \Delta$ and $M_j' = M_j \oplus \Delta$ (for any $\Delta$) and the XOR sum is unchanged. Completely insecure.

**Candidate 4: Use OFB as a one-time pad and take a checksum of plaintext**

Any construction based on OFB ciphertext will fail because OFB ciphertext blocks $C_i = M_i \oplus S_i$ are computationally independent of each other — the OFB keystream creates no binding between blocks. This is the fundamental problem.

---

### Why OFB Cannot Produce a Secure MAC — Root Cause

The security of CBC-MAC comes from the **chaining**: $C_i = E_K(M_i \oplus C_{i-1})$. The MAC value $C_m$ depends on the plaintext through a cryptographic mixing chain — changing $M_i$ changes $C_i$, which changes $C_{i+1}$, ..., which changes $C_m$. Forgery requires finding a different sequence of message blocks that produces the same chain of encryptions — infeasible without the key.

OFB mode has **no such chaining through message content**. The keystream $S_i = E_K(S_{i-1})$ chains through encrypted keystream blocks, not through message content. The message enters only at the XOR step $C_i = M_i \oplus S_i$, and the XOR does not propagate message content to subsequent blocks.

Any MAC function derived from OFB will fail to bind all message blocks together through a cryptographic one-way chain. The construction cannot achieve the essential MAC property that any modification to any message block must, with overwhelming probability, change the tag value — because OFB provides no mechanism for early message blocks to influence the processing of later ones.

**Formal statement**: A secure MAC must be a pseudo-random function (PRF) of the message, meaning that for any two distinct messages $M \neq M'$, the tags $MAC(K, M)$ and $MAC(K, M')$ are computationally independent. OFB-based constructions cannot achieve this because the keystream is message-independent — any tag derived purely from the OFB keystream is trivially forgeable (independent of message content), and any tag that includes $C_i = M_i \oplus S_i$ depends on only one block at a time with no cross-block dependency.

---

### Contrast with GCM

GCM solves this problem correctly by **separating** the encryption function (CTR mode, which like OFB produces a message-independent keystream) from the authentication function (GHASH, which takes ciphertext blocks as inputs and accumulates them through a GF(2^{128}) polynomial) (ch2.2.3 p.70–75). GHASH chains ciphertext blocks multiplicatively, so each block influences all subsequent GHASH outputs. GCM does not use the CTR keystream for authentication — it uses a separate cryptographic mechanism (polynomial evaluation over GF(2^{128})). This is precisely why OFB cannot naively produce a MAC: the keystream computation alone is insufficient, and GCM demonstrates the correct separation between keystream generation and authentication.

### Conclusion

No secure MAC function can be derived from OFB-mode encryption by simply taking some output of the OFB encryption process. The OFB keystream is independent of message content; any function of the OFB ciphertext either ignores blocks entirely or allows block-substitution/reordering attacks. A secure MAC requires a chaining mechanism that cryptographically mixes all message blocks — such as CBC's feedback chain or GCM's GHASH polynomial — which OFB mode fundamentally lacks.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.52–55: OFB mode operation; p.63–66: CBC-MAC construction and security; p.70–75: GCM — separation of encryption and authentication)

_Status: Complete_  
_Done by: William_
