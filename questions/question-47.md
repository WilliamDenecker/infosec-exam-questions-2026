# Question 47

**Explain the role of the "seed" in RSA-OAEP. What are the potential vulnerabilities of "raw" RSA that are eliminated (or at least very strongly mitigated) by the use of this seed (+ explain how)? How can it be recovered from an encoded message? What is a reasonable minimal length for this seed (+ explain why)?**

## Answer

### Overview of RSA-OAEP (ch2.2.3 p.94–97; ch2.2.2 p.46)

RSA-OAEP (Optimal Asymmetric Encryption Padding, PKCS#1 v2.0) is a provably secure RSA encryption scheme (ch2.2.2 p.46). Under the assumption of ideal hash functions, recovering the plaintext from a ciphertext is provably as hard as the RSA problem itself. This is in contrast to "raw" RSA and PKCS#1 v1.5, which have no such proof and are vulnerable to specific attacks.

---

### OAEP Encoding Process (ch2.2.3 p.95–96)

**Parameters**:
- $k$: length (bytes) of the RSA modulus $n$
- $H$: hash function; $h\text{Len}$ = length (bytes) of $H$'s output
- $L$: optional label (default: empty string)
- $m\text{Len}$: length (bytes) of plaintext message $M$ (must satisfy $m\text{Len} \leq k - 2h\text{Len} - 2$)

**Encoding**:
1. Construct **DB** (Data Block): $\text{DB} = H(L) \| \underbrace{0 \cdots 0}_{\text{PS}} \| \texttt{0x01} \| M$
   - $H(L)$: hash of the label ($h\text{Len}$ bytes)
   - PS: zero-padding to fill the block to length $k - h\text{Len} - 1$ bytes
2. Generate a **random seed** of length $h\text{Len}$ bytes
3. $\text{dbMask} = \text{MGF}(\text{seed},\ k - h\text{Len} - 1)$
4. $\text{maskedDB} = \text{DB} \oplus \text{dbMask}$
5. $\text{seedMask} = \text{MGF}(\text{maskedDB},\ h\text{Len})$
6. $\text{maskedSeed} = \text{seed} \oplus \text{seedMask}$
7. $\text{EM} = \texttt{0x00} \| \text{maskedSeed} \| \text{maskedDB}$

The encoded message EM (of length $k$ bytes) is then converted to an integer and raised to the power $e$ modulo $n$ (standard RSA).

---

### Role of the Seed (ch2.2.3 p.96–97)

The seed is a random byte string of length $h\text{Len}$ bytes, generated freshly for each encryption. It plays three interlocking roles:

**1. Randomisation of the ciphertext** (ch2.2.3 p.97):
The seed drives the MGF to produce dbMask, which is XORed with DB to create maskedDB. Since the seed is chosen uniformly at random, maskedDB (and therefore the entire encoded message EM) is different for every encryption of the same plaintext, even with the same public key. This eliminates the **determinism** of raw RSA.

**2. Mutual masking between seed and data block**:
The seed masks DB (via dbMask), and maskedDB in turn masks the seed (via seedMask → maskedSeed). This bidirectional dependency means that the entire encoded message EM is uniformly distributed: neither maskedDB nor maskedSeed individually reveals anything about the plaintext or the seed without decrypting the entire RSA ciphertext.

**3. Structural integrity (indistinguishability)**:
The presence of $H(L)$ and the zero-padding PS in DB, combined with the seed's masking, creates a structured message that is computationally indistinguishable from random — this is the basis of the IND-CCA2 security proof.

---

### Vulnerabilities of "Raw" RSA Eliminated by the Seed (ch2.2.2 p.23–27; ch2.2.3 p.97)

**1. Small public exponent attack / limited message space (ch2.2.2 p.23–25; p.97)**:
- Raw RSA with $e = 3$ is vulnerable: if the same message $M$ is encrypted to three different public keys, CRT gives $M^3$ directly, from which $M$ is easily derived
- With OAEP seed: every encryption produces a different EM (fresh random seed → different maskedDB → different EM → different ciphertext). Two OAEP ciphertexts of the same $M$ under different keys encrypt different EM values — the CRT attack fails
- Also eliminates dictionary attacks: attacker cannot precompute $E_{K_{pub}}(M^*)$ for candidate messages and compare with the observed ciphertext, because the seed changes with every encryption

**2. Multiplicative property / signature forgery (ch2.2.2 p.27; p.97)**:
- Raw RSA: $E(m_1 \cdot m_2) = E(m_1) \cdot E(m_2) \bmod n$ — allows adaptive chosen ciphertext attacks
- OAEP: the structured encoding (first byte = 0x00, $H(L)$ fixed, PS of zero bytes, 0x01 separator) means that a product of two valid OAEP-encoded messages is overwhelmingly unlikely to decode to a valid OAEP structure — the attack produces only garbage

**3. "Million message attack" / Bleichenbacher's attack (ch2.2.2 p.45; p.97)**:
- PKCS#1 v1.5 is vulnerable to a chosen-ciphertext attack exploiting the error responses from decryption (does the decrypted result start with 0x00 0x02?)
- OAEP: decryption either succeeds completely or fails without leaking partial structural information — the IND-CCA2 proof guarantees that no adaptive chosen-ciphertext attack can recover the plaintext

---

### Recovering the Seed from an Encoded Message (ch2.2.3 p.96)

After RSA decryption, the recipient has EM = 0x00 || maskedSeed || maskedDB.

Recovery steps:
1. Parse EM: extract maskedSeed ($h\text{Len}$ bytes) and maskedDB ($k - h\text{Len} - 1$ bytes)
2. Compute $\text{seedMask} = \text{MGF}(\text{maskedDB},\ h\text{Len})$
3. Recover $\text{seed} = \text{maskedSeed} \oplus \text{seedMask}$
4. Compute $\text{dbMask} = \text{MGF}(\text{seed},\ k - h\text{Len} - 1)$
5. Recover $\text{DB} = \text{maskedDB} \oplus \text{dbMask}$
6. Verify DB structure: check DB starts with $H(L)$, followed by zero bytes, then 0x01, then extract $M$

The seed itself is not transmitted unmasked; it is masked by seedMask (derived from maskedDB) so that an attacker who sees EM cannot directly read the seed.

---

### Minimum Seed Length (ch2.2.3 p.94–95)

The seed must have length equal to $h\text{Len}$ — the output length of the hash function $H$. For SHA-256 this is 32 bytes (256 bits).

**Why this length is necessary**:

1. **Security of MGF**: the Mask Generation Function (MGF) is based on repeated hashing of the seed. The security of MGF as a pseudorandom generator depends on the seed being unguessable — the seed must have at least $h\text{Len}$ bits of entropy to provide full security (otherwise the attacker could enumerate seeds and check each)

2. **IND-CCA2 security reduction**: the formal security proof of OAEP in the random oracle model reduces breaking OAEP to solving the RSA problem. The reduction requires the seed space to be large enough that the adversary cannot enumerate it in polynomial time: $2^{8 \cdot h\text{Len}}$ seeds must be large enough to make exhaustive search infeasible

3. **Equal masking**: the seed and hash output are the same length ($h\text{Len}$) so that seedMask exactly covers maskedSeed and vice versa — any shorter seed would leave part of the maskedSeed unmasked or shorten the security parameter

For SHA-256 ($h\text{Len} = 32$ bytes): seed space = $2^{256}$ — infeasible to enumerate. Using a shorter seed (e.g., 16 bytes) would reduce the seed space to $2^{128}$ and weaken the security guarantee.

---

### Summary

| Aspect | Role |
|---|---|
| Randomisation | Fresh random seed → different ciphertext each encryption |
| Mask for DB | seed → MGF → dbMask → maskedDB (hides plaintext) |
| Mask for seed | maskedDB → MGF → seedMask → maskedSeed (hides seed) |
| Recovery | maskedSeed ⊕ seedMask = seed (after RSA decrypt) |
| Minimum length | $h\text{Len}$ bytes (= hash output length, e.g. 32 bytes for SHA-256) |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.94: RSA-OAEP objective — provable security; p.95: parameters; p.96: OAEP encoding scheme — DB, seed, MGF, maskedDB, maskedSeed, EM; p.97: security conclusions — attacks eliminated by seed)
- IS_UG_2_2_2_SecM_AsymmEncr (p.23–27: raw RSA vulnerabilities — small exponent, limited message space, multiplicative properties; p.35–46: PKCS#1 schemes; p.45: Bleichenbacher attack on PKCS#1 v1.5; p.46: RSA-OAEP provable security)

_Status: Complete_  
_Done by: William_
