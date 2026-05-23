# Question 21

Compare the signature schemes PKCS #1 v1.5 and PKCS #1 v2.1 (RSA-PSS) for RSA.

**Explain how signature *verification* works for an RSA-PSS signed message. Which elements prevent the problems one may encounter using the "raw" RSA algorithm for digital signatures? Can you explain why RSA-PSS may be an improvement over the older PKCS #1 v1.5? Which elements are responsible for this improvement? Would the use of MD5 as a hash function in RSA-PSS be a security risk?**

## Answer

### Problems with "Raw" RSA for Digital Signatures (ch2.2.3 p.77–85)

The "raw" RSA signature scheme signs a message $M$ directly:
$$\sigma = m^d \bmod n \quad \text{where } m = M \text{ (or some direct encoding)}$$

**Problems with raw RSA**:

1. **Malleability**: given a valid signature $\sigma = m^d \bmod n$, an attacker can forge related signatures without knowing $d$. For example, given signatures $\sigma_1$ on $m_1$ and $\sigma_2$ on $m_2$, the attacker can compute a valid signature on $m_1 \cdot m_2 \bmod n$: $(\sigma_1 \cdot \sigma_2) \bmod n = (m_1 \cdot m_2)^d \bmod n$. This multiplicative homomorphism allows existential forgeries.

2. **Trivial forgery**: choose any $\sigma$; compute $m = \sigma^e \bmod n$; claim $\sigma$ is a valid signature on $m$. The verifier would accept this.

3. **No randomness**: the same message always produces the same signature — leaks information and allows offline verification before attack.

4. **No hash**: signing large messages directly is impractical (RSA only operates on values $< n$), and without a cryptographic hash there is no avalanche effect (small changes in $M$ produce related signatures).

Both PKCS #1 v1.5 and RSA-PSS address these problems by hashing the message first and applying structured padding before RSA signature computation.

---

### PKCS #1 v1.5 Signatures (ch2.2.3 p.77–79)

**Signing**:
1. Compute $h = H(M)$ (e.g., SHA-256)
2. Apply deterministic padding: construct $T = 0x00 \| 0x01 \| \underbrace{0xFF \cdots 0xFF}_{PS} \| 0x00 \| DigestInfo \| h$
   where DigestInfo encodes the hash algorithm OID and $PS$ is at least 8 bytes of 0xFF padding filling the remainder of the modulus
3. Signature: $\sigma = T^d \bmod n$

**Verification**: compute $T' = \sigma^e \bmod n$ and check that $T'$ has the correct padding structure and contains the expected $H(M)$.

**Problems with PKCS #1 v1.5**:
- The padding is **deterministic** — the same message always produces the same signature value $\sigma$ (no randomness)
- No tight proof of security — the scheme's security has not been formally reduced to the RSA hardness assumption under standard assumptions
- Historical Bleichenbacher-style attacks: in an encryption context, PKCS #1 v1.5 encryption padding is catastrophically vulnerable; for signatures the same structural weaknesses (determinism, rigid padding) create attack surface even if direct practical attacks on PKCS #1 v1.5 signatures are harder

---

### RSA-PSS Signature Verification (ch2.2.3 p.85)

**RSA-PSS signing** (to understand verification):
1. Hash the message: $mHash = H(M)$
2. Generate a random salt: $salt$ (random bytes, length $sLen$, e.g., 32 bytes for SHA-256)
3. Compute $M' = \underbrace{0x00 \cdots 0x00}_{8 \text{ bytes}} \| mHash \| salt$
4. $H' = H(M')$ (hash of the padded message with the random salt)
5. Generate a mask using a Mask Generation Function: $DB = \underbrace{0x00 \cdots 0x00}_{PS} \| 0x01 \| salt$
6. $maskedDB = DB \oplus MGF1(H', emLen - hLen - 1)$
7. Construct the encoded message: $EM = maskedDB \| H' \| 0xbc$
8. Sign: $\sigma = EM^d \bmod n$

**RSA-PSS Verification** (step by step):
1. Recover encoded message: $EM = \sigma^e \bmod n$
2. Check rightmost byte of $EM$ is $0xbc$ (constant trailer)
3. Extract: $maskedDB$ (leftmost $emLen - hLen - 1$ bytes) and $H'$ (next $hLen$ bytes)
4. Recompute: $DB = maskedDB \oplus MGF1(H', emLen - hLen - 1)$
5. Check the leftmost bits of $DB$ are zero (per spec)
6. Extract $salt$ (rightmost $sLen$ bytes of $DB$); verify the padding bytes ($0x00\cdots0x01$) before salt are correct
7. Compute: $mHash = H(M)$ (from the message being verified)
8. Reconstruct: $M' = \underbrace{0x00 \cdots 0x00}_{8} \| mHash \| salt$
9. Compute: $H'' = H(M')$
10. **Accept** if $H'' = H'$; **reject** otherwise

---

### Elements That Prevent Raw RSA Problems

**1. Message hashing ($mHash = H(M)$)**:
- Prevents existential forgery: the adversary would need to find a preimage of the hash to create a signature for a chosen message
- Enables signing of arbitrary-length messages
- The avalanche effect of the hash means any change to $M$ produces a completely different $mHash$

**2. Random salt**:
- Prevents the determinism of raw RSA and PKCS #1 v1.5: the same message signed twice produces different signatures (because the salt is different each time)
- Prevents offline dictionary/precomputation attacks on the signature
- Prevents the "same signature = same message" inference that deterministic schemes allow

**3. Mask Generation Function (MGF1)**:
- Spreads the hash $H'$ across the full width of the encoded message via a pseudo-random mask
- Ensures the signature depends on the entire encoded message block (not just the hash bytes), using the full RSA modulus width
- Prevents an adversary from manipulating individual bytes of the encoded message without affecting $H'$

**4. $H'$ covers both $mHash$ and $salt$**:
- Binds the hash of the message to the specific salt used — neither can be changed independently without invalidating $H'$
- The embedding of the salt in $DB$ and the hash of $(mHash \| salt)$ in $H'$ creates a two-way binding

---

### Why RSA-PSS Is an Improvement Over PKCS #1 v1.5

**Provable security**: RSA-PSS is provably secure in the random oracle model — its security reduces to the RSA One-Way Function (OWF) hardness assumption. If RSA-PSS is broken, RSA itself is broken. PKCS #1 v1.5 has no such tight reduction; its security is only heuristic. This is the fundamental improvement.

**Randomised signatures**: the random salt ensures that even with the same message and key, two PSS signatures are computationally independent. PKCS #1 v1.5 signatures are deterministic — given $(M, \sigma)$, the adversary knows exactly what $T$ looks like, enabling certain attacks on the deterministic structure.

**Full-width use of the modulus**: PSS uses MGF1 to spread entropy across the entire encoded message, using all $\lfloor \log_2 n \rfloor$ bits. PKCS #1 v1.5 padding leaves the signature value in a predictable form with many fixed bytes.

---

### Would MD5 in RSA-PSS Be a Security Risk?

**Yes**, using MD5 as the hash function $H$ in RSA-PSS would be a security risk, for the following reasons:

**1. MD5 collision resistance is broken** (ch2.2.3 p.14): practical collision attacks against MD5 have been known since 2004. An attacker can find two messages $M_1 \neq M_2$ with $MD5(M_1) = MD5(M_2)$.

**Impact on PSS specifically**: in PSS, $mHash = H(M)$ and $H' = H(mHash \| salt)$. If an attacker can find $M_1, M_2$ with $MD5(M_1) = MD5(M_2) = mHash$, then any PSS signature on $M_1$ is also a valid PSS signature on $M_2$ (since $mHash$ is identical, and the rest of the PSS construction is deterministic given the same salt). This completely breaks the security of the digital signature — **one valid signature covers two distinct messages**.

**2. Preimage attacks on MD5**: while not yet fully practical, MD5's preimage resistance is weaker than SHA-256. Any advance in MD5 preimage attacks would further weaken RSA-PSS.

**3. The security of PSS reduces to the security of the hash**: PSS's provable security holds only when $H$ is a secure hash function (modelled as a random oracle). MD5 is demonstrably not a secure hash function (collision resistance broken). Substituting MD5 voids the security proof.

**Conclusion**: the randomness and masking of PSS cannot compensate for a broken underlying hash function. RSA-PSS requires a collision-resistant hash function — SHA-256 or SHA-384 — to provide its provable security guarantees. MD5 must not be used.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.14: MD5 collision attacks; p.77–79: PKCS #1 v1.5; p.85: RSA-PSS construction and security)

_Status: Complete_  
_Done by: William_
