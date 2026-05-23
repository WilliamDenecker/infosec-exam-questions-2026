# Question 25

**Explain why CBC-MAC is not secure for variable-length messages and describe the potential vulnerabilities that arise.**

**Why is this not an issue with CCM-mode encryption?**

## Answer

### CBC-MAC for Fixed-Length Messages (ch2.2.3 p.63–66)

For a **fixed, agreed-upon message length** $m$ (number of blocks), CBC-MAC is provably secure as a MAC. The tag is computed as:
$$T_0 = 0^{128}, \quad T_i = E_K(M_i \oplus T_{i-1}), \quad MAC = T_m$$

The security proof assumes the message length is fixed and known — the adversary cannot submit messages of different lengths. When this assumption is violated, two distinct vulnerabilities arise.

---

### Vulnerability 1 — Length Extension Attack

**Setup**: an attacker obtains a valid MAC $T_m = CBC\text{-}MAC_K(M_1 \| M_2 \| \ldots \| M_m)$.

**Attack**: the attacker constructs a new message:
$$M' = M_1 \| M_2 \| \ldots \| M_m \| M_{m+1}'$$
where $M_{m+1}' = T_m \oplus M_{m+1}$ for any chosen block $M_{m+1}$.

The CBC-MAC of $M'$ starting from $T_m$ is:
$$T_{m+1} = E_K(M_{m+1}' \oplus T_m) = E_K((T_m \oplus M_{m+1}) \oplus T_m) = E_K(M_{m+1})$$

But this is exactly $CBC\text{-}MAC_K(M_{m+1})$ — the MAC of the single-block message $M_{m+1}$.

**If the attacker also knows $CBC\text{-}MAC_K(M_{m+1})$** (e.g., obtained from a separate MAC query), they now know the MAC of $M'$ — a completely forged MAC for a new message — without knowing $K$.

**Generalisation**: any message that starts with a valid tagged message can be extended. Given $MAC_K(M)$ and $MAC_K(N)$ for single-block messages $M$ and $N$, the attacker can construct $MAC_K(M \| (T_M \oplus N))$ = $MAC_K(N)$ — a valid MAC for a two-block message.

---

### Vulnerability 2 — Prefix Attack (Different Length Messages)

**Setup**: CBC-MAC is used for messages of variable length with the same key $K$.

**Attack**: suppose the attacker obtains:
- $T_1 = CBC\text{-}MAC_K(X)$ for some 1-block message $X$
- $T_2 = CBC\text{-}MAC_K(X \| Y)$ for the 2-block message $X \| Y$

The computation of $T_2$ starts with $T_1 = E_K(X)$, then:
$$T_2 = E_K(Y \oplus T_1) = E_K(Y \oplus CBC\text{-}MAC_K(X))$$

Now the attacker constructs:
$$X' = X \oplus T_1$$

Note: $CBC\text{-}MAC_K(X') = E_K(X') = E_K(X \oplus T_1) = E_K(X \oplus E_K(X))$

More directly: the attacker chooses any 1-block message $A$, queries $T_A = CBC\text{-}MAC_K(A)$, and then computes:
$$CBC\text{-}MAC_K(A \| (T_A \oplus A)) = E_K((T_A \oplus A) \oplus T_A) = E_K(A) = T_A$$

Wait — this shows the MAC of the 2-block message $A \| (T_A \oplus A)$ equals $T_A$. So the attacker has a collision: $CBC\text{-}MAC_K(A) = CBC\text{-}MAC_K(A \| (T_A \oplus A)) = T_A$. Two messages of different lengths with the same MAC — a direct violation of MAC security.

---

### Root Cause

The root cause is that CBC-MAC's state after processing $m$ blocks is exactly $T_m$ — the MAC value. Any entity that knows $T_m$ and a subsequent block $M_{m+1}$ can continue the chain exactly as the legitimate MAC computation would. The MAC value exposes the internal state, enabling extension. For **fixed-length** messages, this is not an issue because the adversary cannot extend a message while remaining within the fixed-length constraint.

---

### Why CCM Mode Avoids This Problem (ch2.2.3 p.70–75)

**CCM (Counter with CBC-MAC)** is a mode of operation that combines CTR-mode encryption with CBC-MAC authentication. It is designed to be secure for variable-length messages and explicitly avoids the length extension vulnerability through a structural fix.

**How CCM prevents the length extension attack**:

CCM prepends a **formatted header block** to the message before computing the CBC-MAC. This header block encodes:
- A **flag byte** specifying whether there is associated data and the length of the authentication tag
- The **nonce**
- The **length of the message** (in bytes)

This header block is the very first block fed into the CBC-MAC:
$$T_0 = E_K(\text{formatted\_header} \oplus 0^{128})$$

**Critical security consequence**: the message length $\ell(M)$ is encoded in the first CBC-MAC block. The MAC computation is therefore keyed to the specific message length.

- A MAC computed for a message of length $m$ blocks includes $\ell = m \cdot 128$ (in bits) in the header block
- If an attacker tries to extend the message to $m+1$ blocks, the header encoding the length $m$ does not match the length $m+1$ — the extended message would have a different length, which would correspond to a different header, yielding a different initial $T_0$, and therefore a completely different MAC chain

The forged extended message cannot have the same MAC as the original because the length field in the header changes the very first block of the CBC-MAC computation, propagating the difference through the entire chain.

**Nonce inclusion**: CCM also uses a per-message nonce. The formatted header includes this nonce, ensuring that the CBC-MAC is unique per message instance. An attacker cannot reuse a MAC computed for one message in a different context.

**Summary**:

| Property | CBC-MAC alone | CCM mode |
|---|---|---|
| Variable-length messages | **Insecure** (length extension, prefix collision) | **Secure** |
| Message length binding | Not bound | Length encoded in first MAC block |
| Nonce | Not included | Nonce in first MAC block |
| Length extension attack | Possible | Not possible (length mismatch changes entire MAC) |
| Prefix collision attack | Possible | Not possible (nonce + length make each MAC unique) |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.63–66: CBC-MAC construction, security for fixed-length messages, length extension vulnerability; p.70–75: CCM and GCM modes — authenticated encryption solving variable-length MAC problem)

_Status: Complete_  
_Done by: William_
