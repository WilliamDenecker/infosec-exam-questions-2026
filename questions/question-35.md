# Question 35

Compare the encryption schemes PKCS #1 v1.5 and PKCS #1 v2.0 (RSA-OAEP) for RSA (for public key encryption).

**Explain how *decryption* works for an RSA-OAEP encrypted message. Which elements prevent the problems one may encounter using the "raw" RSA algorithm for public key encryption? Can you explain why RSA-OAEP may be an improvement over the older PKCS #1 v1.5? Which elements are responsible for this improvement? Would the use of MD5 as a hash function in RSA-OAEP be a security risk?**

## Answer

### Problems with "Raw" RSA for Public Key Encryption (ch2.2.3 p.77–85)

Raw RSA encryption: $c = m^e \bmod n$. This has fundamental problems:

1. **Deterministic**: $Encrypt(m) = m^e \bmod n$ is the same every time. An attacker can test candidate plaintexts by encrypting them and comparing to the ciphertext. For small message spaces (e.g., yes/no, accept/reject), all possible plaintexts can be encrypted and the correct one identified.

2. **Malleable**: given $c = m^e \bmod n$, an attacker can construct $c' = c \cdot r^e \bmod n = (m \cdot r)^e \bmod n$ — a valid encryption of $m \cdot r$ for any chosen $r$, without knowing $m$.

3. **No protection against chosen-ciphertext attacks**: the RSA group structure allows the attacker to adaptively choose ciphertexts and use decryption results to learn the plaintext.

4. **No integrity**: nothing prevents an attacker from submitting a modified ciphertext for decryption.

Both PKCS #1 v1.5 and OAEP add randomisation and structure to prevent these problems.

---

### PKCS #1 v1.5 Encryption (ch2.2.3 p.77–79)

**Encryption**: pad the message $m$ as: $P = 0x00 \| 0x02 \| PS \| 0x00 \| m$, where $PS$ is at least 8 random non-zero bytes (padding string). Encrypt: $c = P^e \bmod n$.

**Weaknesses**:
- The structured padding means that a decryption of a random ciphertext can be tested for "valid" padding — does it start with `0x00 0x02` and contain a zero byte after at least 8 non-zero bytes? Valid padding is rare (probability $\approx 2^{-16}$) but detectable.
- **Bleichenbacher's attack (1998)**: if the decryption oracle reveals whether the padding is valid (even through timing differences), an adaptive chosen-ciphertext attacker can recover any plaintext through approximately $10^6$–$10^7$ queries. This attack applies to SSL/TLS servers that revealed padding validity through timing. Many real-world implementations were vulnerable.
- The padding has some randomness ($PS$) but not full randomness covering the entire message block.

---

### RSA-OAEP Decryption Process (ch2.2.3 p.77–79)

**RSA-OAEP encryption** (to understand decryption):
1. Message $m$ with label $L$ (often empty). Let $hLen = |H(\cdot)|$ (e.g., 32 bytes for SHA-256).
2. Encode message as $DB = H(L) \| \underbrace{0x00\cdots0x00}_{PS} \| 0x01 \| m$ (data block)
3. Generate random seed $r$ of length $hLen$
4. Compute mask: $dbMask = MGF1(r,\ |DB|)$ (mask generation function, based on hash)
5. Compute: $maskedDB = DB \oplus dbMask$
6. Compute: $seedMask = MGF1(maskedDB,\ hLen)$
7. Compute: $maskedSeed = r \oplus seedMask$
8. Construct: $EM = 0x00 \| maskedSeed \| maskedDB$
9. Encrypt: $c = EM^e \bmod n$

**RSA-OAEP decryption** (step by step):
1. Compute: $EM = c^d \bmod n$ (RSA private key decryption)
2. Split $EM$: first byte must be $0x00$; then $maskedSeed$ ($hLen$ bytes); then $maskedDB$ (remainder)
3. Recover seed: $seedMask = MGF1(maskedDB,\ hLen)$; then $r = maskedSeed \oplus seedMask$
4. Recover data block: $dbMask = MGF1(r,\ |DB|)$; then $DB = maskedDB \oplus dbMask$
5. Parse $DB$: extract $H(L)'$ (first $hLen$ bytes); find the $0x01$ byte separator; extract $m$ (bytes after $0x01$)
6. Verify: $H(L)' = H(L)$ (the label hash in $DB$ matches the expected label hash)
7. Verify: there are only $0x00$ bytes between $H(L)'$ and the $0x01$ byte
8. If all checks pass: return $m$. If any check fails: return error (but in a constant-time, non-revealing way)

---

### Elements That Prevent Raw RSA Problems in OAEP

**1. Randomisation via random seed $r$**: the seed $r$ is generated fresh for each encryption. The same message $m$ encrypted twice produces different $c$ values (because different $r$ values). This defeats deterministic attacks — an attacker who guesses $m$ and encrypts it will get a different ciphertext than the one they're attacking.

**2. MGF1 — full-width randomisation**: MGF1 (Mask Generation Function, based on a hash) spreads the seed $r$ across the entire $DB$ and vice versa. The masking ensures that:
- Every bit of $maskedDB$ depends on every bit of $r$
- Every bit of $maskedSeed$ depends on every bit of $maskedDB$ (and thus on $m$)
- The encoded message $EM$ has no structure visible without decryption

**3. Cryptographic binding between seed and data**: the mutual masking ($dbMask = MGF1(maskedSeed, \ldots)$; $seedMask = MGF1(maskedDB, \ldots)$) means that modifying any bit of $c$ causes both recoveries to fail simultaneously, preventing malleability.

**4. Label hash $H(L)$**: the expected hash of the label $L$ is embedded in $DB$. If the decryption does not produce the correct $H(L)$, the entire decryption is rejected. This provides integrity for the label — the encryption is bound to the specific intended use context.

**5. Provable security**: OAEP is provably secure against adaptive chosen-ciphertext attacks (IND-CCA2 security) in the random oracle model — the best known security property for public-key encryption schemes.

---

### Why OAEP Is an Improvement Over PKCS #1 v1.5

**1. Provable security**: OAEP has a formal proof that breaking it requires breaking RSA (in the random oracle model). PKCS #1 v1.5 has no such proof — its security is only heuristic, and Bleichenbacher's attack demonstrates it can be broken given a decryption oracle.

**2. Full randomisation**: OAEP uses a random seed that fully randomises the entire encoded message via MGF1. PKCS #1 v1.5 only randomises the short padding string $PS$; the rest of the structure is deterministic, making it more fragile.

**3. No padding oracle**: OAEP validation (checking $H(L)' = H(L)$ and $0x01$ separator) can and should be implemented in **constant time** — returning only "success" or "failure" with no timing variation. PKCS #1 v1.5 was historically implemented with non-constant-time padding checks, creating the oracle that Bleichenbacher's attack exploits. While PKCS #1 v1.5 implementations can be fixed with careful constant-time coding, OAEP's structure makes it more naturally resistant because the failure conditions are hash comparisons rather than byte-by-byte format checks.

**4. Malleability prevention**: the mutual MGF1 dependency means any modification to the ciphertext produces unpredictable changes in the decrypted $EM$, causing validation to fail. PKCS #1 v1.5 ciphertexts can be multiplied by $r^e$ to produce a ciphertext of $m \cdot r$; OAEP does not have this algebraic structure.

---

### Would MD5 in RSA-OAEP Be a Security Risk?

**Yes**, using MD5 as the hash function $H$ in RSA-OAEP creates security risks:

**1. Collision attacks on the label hash**: the label $L$ is bound to the encryption via $H(L)$. With MD5, an attacker who can produce $L_1 \neq L_2$ with $MD5(L_1) = MD5(L_2)$ (ch2.2.3 p.14 — MD5 collisions are practical) could potentially use a ciphertext intended for one context in a different context with a colliding label. The label is often empty in practice, making this less relevant, but it is a structural weakness.

**2. MGF1 security**: MGF1 is instantiated as a hash-based XOF. Its security depends on the preimage resistance of the underlying hash. While MD5 preimage resistance is not yet broken in practice, MD5's security properties are significantly weakened compared to SHA-256, undermining the security proof.

**3. OAEP's security proof requires a random oracle**: the proof of IND-CCA2 security for OAEP models the hash function as a random oracle. MD5 is demonstrably not a random oracle — it has structural properties (algebraic collisions, related-key weaknesses) that violate the random oracle model. Substituting MD5 weakens or voids the security proof.

**Conclusion**: MD5 should not be used in RSA-OAEP. SHA-256 or SHA-384 are the appropriate choices, consistent with the provable security model and current standards.

| Criterion | PKCS #1 v1.5 | RSA-OAEP |
|---|---|---|
| Determinism | Partially randomised ($PS$) | Fully randomised (seed $r$ + MGF1) |
| Malleable | Yes (multiplicative) | No (MGF1 coupling breaks algebra) |
| Security proof | No (Bleichenbacher attack exists) | Yes (IND-CCA2 in ROM) |
| Padding oracle risk | High (historical) | Low (constant-time validation) |
| Hash function role | None (not used in padding) | Central (MGF1, label hash) |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.14: MD5 collision attacks; p.77–79: RSA-OAEP construction, PKCS #1 v1.5, Bleichenbacher attack; p.85: security proofs for RSA encryption padding)

_Status: Complete_  
_Done by: William_
