# Question 46

Also see questions {5, 10, 25, 39}

**Explain why using the same key for CBC-encryption and for the computation of the CBC-MAC (on the same plaintext) is not secure.**

## Answer

### 1. Recap: What CBC-Encryption and CBC-MAC Compute (ch2.2.3 p.45–52, p.61–62)

**CBC-MAC** always uses $IV = 0$:
$$Y_i = E_K(P_i \oplus Y_{i-1}), \quad Y_0 = 0, \quad T = Y_n$$

**CBC encryption** uses some IV $C_0$:
$$C_i = E_K(P_i \oplus C_{i-1}), \quad C_0 = IV$$

Both operations apply $E_K$ with the **same key** to inputs of the form $P_i \oplus \text{(previous chaining value)}$. The only structural difference is the starting chaining value: $Y_0 = 0$ vs $C_0 = IV$.

---

### 2. Case 1: Encryption Also Uses IV = 0 (ch2.2.3 p.62)

When $C_0 = 0$, both chains start from the same value and use the same key, so they are **identical**:
$$C_1 = E_K(P_1 \oplus 0) = Y_1$$

By induction, $C_i = Y_i$ for every block $i$, and therefore:
$$T = Y_n = C_n$$

The MAC is simply the **last block of the ciphertext** — readable by any attacker who intercepts the message. Every intermediate chaining value $Y_i$ is also exposed as $C_i$.

#### Truncation attack

Because $Y_i = C_i$, the attacker knows not just $T = Y_n$ but also $Y_{n-1} = C_{n-1}$, $Y_{n-2} = C_{n-2}$, and so on — all for free from the ciphertext.

The attacker can present the **truncated message** $P_1 \| \ldots \| P_{n-1}$ with MAC $C_{n-1}$, and the receiver will accept it as valid:
$$\text{CBC-MAC}_K(P_1 \| \ldots \| P_{n-1}) = Y_{n-1} = C_{n-1}$$

The attacker forges a valid MAC for a shorter message without any computation — just by dropping the last ciphertext block. With different keys this is impossible: $C_i$ carries no information about $Y_i$.

#### Cut-and-paste (splice) attack

Since the attacker knows every intermediate chaining value $Y_i = C_i$, they are not limited to truncation. They can also **splice blocks from two different intercepted messages**.

Suppose the attacker intercepts two messages $(P, C)$ and $(P', C')$ and knows the chaining values at every point in both chains. They cut the first $j$ blocks from message $P$ and want to paste blocks from $P'$ onto the end. The CBC-MAC state after block $j$ of $P$ is $Y_j = C_j$. The attacker computes a **glue block**:
$$G = Y_j \oplus Y_0' = C_j \oplus C_0'$$

where $Y_0'$ is the MAC chaining value at the splice point in $P'$. Inserting $G$ as a transition block cancels the difference between the two chains, making the CBC-MAC continue as if it had been computing $P'$ from the start. The result is a forged "frankensteined" message with a mathematically valid MAC tag — assembled purely from intercepted ciphertexts, with no knowledge of $K$.

---

### 3. Case 2: Encryption Uses a Non-Zero IV (ch2.2.3 p.61–62)

Now suppose $C_0 = IV \neq 0$. The chains diverge at block 1:
$$C_1 = E_K(P_1 \oplus IV) \neq E_K(P_1) = Y_1$$

So $T \neq C_n$ in general. This seems safer, but key reuse still leaks MAC values. Two attacks apply, requiring progressively stronger attacker capabilities: the first needs only a known plaintext (no oracle); the second needs a chosen-plaintext oracle.

#### Known-plaintext 1-block existential forgery (no encryption oracle needed)

This is a **known-plaintext attack**: the attacker intercepts $(IV, C_1, \ldots, C_n)$ and also knows the corresponding plaintext $P_1$ — for example because the message has a fixed-format header, or the plaintext was previously disclosed. Knowing $P_1$ is what makes this attack possible; pure ciphertext interception alone is not enough.

From the intercepted ciphertext the attacker reads off:
$$C_1 = E_K(P_1 \oplus IV)$$

They now construct a brand new **1-block message**:
$$M_1 = P_1 \oplus IV$$

This is just XOR of two values the attacker already knows — no key, no oracle. The CBC-MAC of this single-block message is:
$$\text{CBC-MAC}_K(M_1) = E_K(M_1 \oplus 0) = E_K(P_1 \oplus IV) = C_1$$

The last step holds because $E_K(P_1 \oplus IV)$ is exactly what $C_1$ was defined as — the attacker never evaluates $E_K$ themselves, they just recognise that $C_1$ already equals the MAC they need.

The attacker presents $(M_1,\, C_1)$ as a valid authenticated message without querying any encryption oracle. What makes the same-key scheme uniquely vulnerable here is that knowing $(P_1, IV)$ from the encryption side directly gives you the MAC on the encryption side — with different keys, $C_1 = E_{K_E}(P_1 \oplus IV)$ tells you nothing about $E_{K_M}(M_1)$.

#### Re-alignment via a chosen plaintext (multi-block case)

For longer forgeries the attacker can additionally submit a chosen plaintext $P'$ and observe both the ciphertext and the MAC $T'$. They craft a second plaintext $P''$ where only the first block differs:
$$P_1'' = P_1' \oplus IV, \quad P_i'' = P_i' \text{ for } i \geq 2$$

The first encryption step for $P''$ gives:
$$C_1'' = E_K(P_1'' \oplus IV) = E_K(P_1' \oplus IV \oplus IV) = E_K(P_1') = Y_1'$$

The IV cancels out, and from block 1 onwards the encryption chain of $P''$ runs in lockstep with the MAC chain of $P'$:
$$C_i'' = Y_i' \quad \text{for all } i \geq 1$$

In particular $C_n'' = Y_n' = T'$, so the MAC of $P'$ leaks as a ciphertext block of $P''$. The truncation and splice attacks from Case 1 then apply in full. With **different keys**, tweaking $P_1''$ shifts the encryption chain but leaves the independently keyed MAC chain completely unaffected.

---

### 4. Side Note: Deterministic MAC Undermines Semantic Security (ch2.2.3 p.61–62)

CBC-MAC with $IV = 0$ is **deterministic**: the same plaintext always produces the same tag $T$, regardless of the random $IV$ used for encryption. In an Encrypt-and-MAC scheme the tag is sent alongside the ciphertext, so an eavesdropper who sees the same $T$ twice immediately knows the sender transmitted the same message — even though the ciphertext looks completely different due to the random IV. The MAC tag acts as a fingerprint, **undermining the semantic security** that the randomised IV was meant to provide.

---

### 5. Why CCM Can Use the Same Key Safely (ch2.2.3 p.67–68)

CCM mode also uses the same key $K$ for both CBC-MAC (authentication) and CTR mode (encryption). This is secure because CTR mode computes $E_K(\text{counter}_j)$ — the input to $E_K$ is a counter value, completely independent of the plaintext or MAC chaining state. An attacker cannot use a CTR encryption oracle to evaluate $E_K$ at any chosen input; the inputs are fixed counter values outside attacker control.

For **CBC encryption + CBC-MAC**, both operations share the same input structure: $E_K(\text{plaintext block} \oplus \text{previous chaining value})$. The attacker can craft an encryption request whose XOR input equals exactly the value needed to evaluate $E_K$ for a MAC forgery — which is precisely what all the attacks above exploit.

---

### 6. Conclusion

| Scenario | Consequence |
|---|---|
| Both use IV = 0 | $C_i = Y_i$ for all $i$ — MAC and all chaining values leak directly; truncation and splice forgeries require zero computation |
| Encryption uses IV ≠ 0 | Passive eavesdropping yields a 1-block existential forgery; one chosen-plaintext query re-aligns the full chain, restoring all Case 1 attacks |
| Different keys $K_E \neq K_M$ | Ciphertext blocks carry no information about MAC chaining values; all attacks fail |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.45–52: CBC mode, IV, block cipher chaining; p.61–62: CBC-MAC construction with IV=0, same-key insecurity; p.63–66: MAC requirements and keyed authentication; p.67–68: CCM mode — same key for CBC-MAC and CTR, why it is secure; p.70–75: GCM — secure authenticated encryption with single key)

_Status: Complete_  
_Done by: William_
