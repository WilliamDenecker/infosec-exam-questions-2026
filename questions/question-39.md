# Question 39

Consider the following security transformation on a message $M$:

$$E_K(M)\|H(M) \quad (Q39.10)$$

where $E_K$ is a symmetric encryption algorithm (with shared secret key $K$, at least 128 bits long), and $H$ is a hash function.

**How well does this implement the confidentiality and (partial) data-integrity functions? What could be possible issues? Does the quality of this security transformation depend on the encryption algorithm chosen? On the encryption mode chosen? On the hash function chosen?**

*Note: The shared secret key $K$ can be used for a (reasonably) large number of messages.*

## Answer

### What the Construction Does

The construction outputs $C = E_K(M) \| H(M)$: the message is encrypted with key $K$, and then the hash of the **plaintext** $M$ is appended in cleartext. The receiver receives both $E_K(M)$ and $H(M)$.

---

### Confidentiality Analysis (ch2.2.3 p.45–75)

**How well it provides confidentiality**: the encryption $E_K(M)$ provides confidentiality for the content of $M$ — a passive eavesdropper who does not know $K$ cannot recover $M$ from $E_K(M)$ alone (assuming a secure cipher and mode).

**Problem 1 — $H(M)$ leaks information about $M$**:

The hash $H(M)$ is transmitted in cleartext alongside the ciphertext. While $H$ is a one-way function (preimage resistant, ch2.2.3 p.12), it still leaks information:

- **Equality testing**: an attacker who knows or suspects a particular plaintext $M^*$ can compute $H(M^*)$ and compare it to the transmitted $H(M)$. If they match, the attacker knows $M = M^*$ — without ever decrypting $E_K(M)$. For short or predictable messages (binary decisions, fixed-format commands, small files from a known set), this is a devastating confidentiality failure.

- **Dictionary attacks**: if the space of possible messages $M$ is small (e.g., "yes" or "no", or one of a few hundred commands), the attacker precomputes $H(M^*)$ for all possible $M^*$ and matches against the observed $H(M)$.

- **Traffic analysis**: even without content recovery, the hash value correlates with the plaintext — the same message always produces the same hash (if $H$ is deterministic and no salt is used). This allows the attacker to detect when the same plaintext is sent twice (replay detection from the attacker's perspective), or to group messages by content.

**Quality depends on encryption algorithm?** — Yes (but not primarily due to $H(M)$ leakage). The encryption quality depends on the cipher, mode, and key length. But the $H(M)$ leakage is entirely separate from the encryption — regardless of how good $E_K$ is, $H(M)$ in cleartext undermines confidentiality.

---

### Data Integrity Analysis (ch2.2.3 p.63–66)

**What is provided**: the receiver can compute $H(M')$ on the decrypted $M'$ and check whether $H(M') = H(M)$ (from the received cleartext hash). If the ciphertext was modified in transit, then $E_K^{-1}(\text{modified ciphertext})$ will (with overwhelming probability for a good cipher) produce a modified $M'$, and $H(M') \neq H(M)$.

**Problem 2 — $H(M)$ is not a MAC; it is not keyed**:

$H$ is an unkeyed hash function. An attacker who can both modify the ciphertext and the cleartext hash can perform a **hash substitution attack**:
1. Attacker intercepts $E_K(M) \| H(M)$
2. Attacker modifies the ciphertext to $E_K(M)'$
3. Attacker decrypts $M' = E_K^{-1}(E_K(M)')$ — impossible without $K$... but wait:

In many block cipher modes (especially CBC or CTR), modifying a specific ciphertext byte predictably modifies a specific plaintext byte (in CTR mode: XOR bit-flip is exact; in CBC mode: bit-flip in block $i$ corrupts block $i$ and flips a bit in block $i+1$). The attacker may be able to craft a modified ciphertext $E'$ that decrypts to a chosen modified plaintext $M'$, then compute and substitute $H(M')$ for $H(M)$ in the cleartext hash.

**Key point**: because $H$ is unkeyed, the attacker can compute $H(M')$ for any $M'$ — they don't need $K$ to produce a valid hash for a modified message. If they can craft the ciphertext to decrypt to their chosen $M'$, they simply compute $H(M')$ and substitute it. The receiver accepts: $H(M') = H(M')$ ✓.

This is not just theoretical: in CTR mode, flipping bit $b$ of ciphertext block $i$ flips exactly bit $b$ of plaintext block $i$. The attacker can make precise, targeted modifications to the plaintext while still being able to compute the hash of the resulting modified plaintext. This completely breaks integrity.

**Problem 3 — the hash is of the plaintext, not the ciphertext**:

The construction is "Encrypt-then-Hash-of-Plaintext" — related to (but weaker than) the well-known "Encrypt-and-MAC" paradigm. In proper authenticated encryption (e.g., GCM, ch2.2.3 p.70–75), the MAC is computed over the ciphertext (or jointly with it via GHASH) using a key. Here:
- The hash is unkeyed
- The hash covers the plaintext, not the ciphertext
- An attacker who knows $M$ (from prior interactions or guessing) can verify this independently

**A proper MAC** (e.g., HMAC-SHA256 with a separate MAC key) over the ciphertext $E_K(M)$ would provide integrity — because the attacker cannot produce a valid MAC without knowing the MAC key.

---

### Does the Quality Depend on the Encryption Algorithm?

**Confidentiality**: yes — the chosen cipher ($E$) and its mode determine how much information $E_K(M)$ leaks about $M$ independently of $H$. A good cipher in a good mode (e.g., AES-256-GCM) provides ciphertext that is computationally indistinguishable from random. A weak cipher or bad mode (e.g., ECB mode) leaks structure from the plaintext even in the ciphertext.

**Integrity**: yes, partially. In ECB mode, individual blocks are encrypted independently — an attacker can reorder, substitute, or replay individual ciphertext blocks. The hash $H(M)$ would detect this (the reordered decryption is different from the original $M$), but the attacker could modify the ciphertext such that decryption yields a $M'$ they can compute $H(M')$ for, defeating integrity.

In GCM mode, $E_K$ is AES-CTR internally — bit-flip attacks are exact. The attacker who knows the plaintext positions can flip specific bits in the ciphertext, obtaining a precisely modified plaintext for which they compute the new hash.

---

### Does the Quality Depend on the Encryption Mode?

**Yes, significantly for integrity**:

- **ECB mode**: blocks are independent; attacker can reorder/swap ciphertext blocks. Hash detects it but attackers can substitute entire blocks.
- **CTR mode**: bit-flip attacks on ciphertext produce exact bit-flips in plaintext. Attacker with known plaintext can make targeted modifications and update hash.
- **CBC mode**: bit-flip in block $i$ corrupts block $i$ randomly and flips a specific bit in block $i+1$. Partially predictable — targeted attack is harder but not impossible.
- **GCM mode**: same CTR-based encryption as standalone AES-CTR, so same bit-flip vulnerability. But note: in GCM, the authentication tag is computed as a keyed function of the ciphertext — the construction $E_K(M) \| H(M)$ replaces GCM's keyed authentication with an unkeyed hash, losing all authentication integrity guarantees.

---

### Does the Quality Depend on the Hash Function?

**Integrity — yes**:
- If $H$ = MD5: practical collisions exist (ch2.2.3 p.14). An attacker can find $M'$ with $H(M') = H(M)$ efficiently. The attacker could find a fake message with the same hash — trivially defeating integrity without needing to modify the ciphertext at all.
- If $H$ = SHA-256: collision resistance holds currently. The unkeyed hash still allows hash substitution attacks (compute new hash for modified plaintext), but a brute-force collision is not currently feasible.

**Confidentiality — yes**: even with a collision-resistant hash, $H(M)$ in cleartext allows equality testing against candidate plaintexts. With MD5, the same weakness applies plus additional hash vulnerabilities.

---

### Summary

| Security Service | Provided? | Issues |
|---|---|---|
| Confidentiality | Partial | $H(M)$ in cleartext enables equality testing; same plaintext has same hash |
| Data integrity | Partial / Weak | $H$ is unkeyed — attacker can compute $H(M')$ for any $M'$; combined with bit-flip attacks on ciphertext (CTR/CBC), integrity is defeated |
| Authentication (data origin) | **No** | $H$ is not keyed; anyone can compute valid hash for any message |

**Correct alternative**: use AES-256-GCM (ch2.2.3 p.70–75) which provides confidentiality AND keyed integrity (GHASH authentication tag) in a single, standardised, secure construction. Or: Encrypt-then-MAC with separate keys — compute HMAC-SHA256 over the ciphertext.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.12–14: hash function properties; p.45–52: block cipher modes and ciphertext malleability; p.63–66: MAC and keyed integrity; p.70–75: GCM — correct authenticated encryption)

_Status: Complete_  
_Done by: William_
