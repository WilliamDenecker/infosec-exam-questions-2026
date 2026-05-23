# Question 40

Blinding is sometimes used to prevent timing attacks against private key operations in RSA.

**Design a similar blinding operation for elliptic curve cryptography (for a given elliptic curve $EC$ with given generator $G$ of given order $n$).**

## Answer

### RSA Blinding — Context (ch2.2.3 p.80–84)

In RSA, **blinding** prevents timing attacks on the private key exponentiation $m^d \bmod n$ (ch2.2.3 p.84). Without blinding, the execution time of the square-and-multiply algorithm depends on the bits of $d$, allowing timing side-channel recovery of $d$.

RSA blinding works as follows:
1. Choose a random blinding factor $r \in \mathbb{Z}_n^*$
2. Blind the input: $m' = m \cdot r^e \bmod n$ (multiply by $r^e$ — the RSA encryption of $r$)
3. Perform the private key operation on the blinded input: $c' = (m')^d \bmod n = (m \cdot r^e)^d = m^d \cdot r^{ed} = m^d \cdot r \bmod n$
4. Unblind the result: $c = c' \cdot r^{-1} \bmod n = m^d \bmod n$ (the correct answer)

The attacker observes the timing of step 3, which operates on random $m'$ rather than the actual input $m$. Since $m'$ changes for every operation (fresh random $r$ each time), the attacker cannot build a statistical model correlating execution time with specific $m$ values. The blinding randomises the intermediate values, destroying timing correlation.

---

### ECC Blinding — Goal

In ECC, the core private key operation is **scalar multiplication**: $Q = k \cdot P$ (given point $P$ and private scalar $k$, compute $k \cdot P$). The double-and-add algorithm's execution time depends on the bits of $k$ (see Question 29). Blinding must prevent timing correlation between execution time and the bits of $k$.

Two complementary blinding techniques for ECC:

---

### Technique 1 — Scalar Blinding (Randomising the Private Key $k$)

**Goal**: make the scalar value processed during computation different for every execution, even when the same private key $k$ is used.

**Method**: choose a random integer $r$ uniformly from $\{0, 1, \ldots, n-1\}$ before each scalar multiplication. Compute:
$$k' = k + r \cdot n$$

The modified scalar $k'$ satisfies:
$$k' \cdot P = (k + r \cdot n) \cdot P = k \cdot P + r \cdot (n \cdot P) = k \cdot P + r \cdot \mathcal{O} = k \cdot P$$

since $n \cdot P = \mathcal{O}$ (the point at infinity, the additive identity of the elliptic curve group — this follows from the fact that the order of generator $G$, and all points on the curve, divides $n$, so $n \cdot P = \mathcal{O}$).

The result $k' \cdot P = k \cdot P$ is unchanged, but the scalar $k'$ processed by the double-and-add algorithm is different for every execution (fresh random $r$). Since the bit pattern of $k'$ is unpredictable, the timing signature of the double-and-add is different each time — no correlation builds up between timing and the bits of $k$.

**Implementation**:
1. Generate fresh random $r \in \{0, \ldots, 2^t - 1\}$ for some $t$ (e.g., $t = 32$ bits)
2. Compute $k' = k + r \cdot n$ (a larger scalar, approximately $|k| + t$ bits)
3. Compute $Q = k' \cdot P$ using any scalar multiplication algorithm
4. Return $Q$ (no unblinding needed — the result is automatically correct)

**Effect on timing**: the scalar $k'$ has $|k| + t$ bits (approximately), so scalar multiplication takes slightly longer on average. The additional cost is proportional to the blinding width $t$, typically $< 5\%$ overhead for $t = 32$ bits.

---

### Technique 2 — Point Blinding (Randomising the Input Point $P$)

**Goal**: make the intermediate point values during scalar multiplication different for every execution.

**Method**: instead of computing $k \cdot P$ directly, add a random multiple of the generator $G$ to the input, compute the blinded result, then subtract the corresponding correction:

1. Choose random $r \in \{0, \ldots, n-1\}$ (uniform random)
2. Compute the blinded input: $P' = P + r \cdot G$ (a random "offset" point added to $P$)
3. Compute the blinded output: $Q' = k \cdot P' = k \cdot P + k \cdot r \cdot G = k \cdot P + (kr \bmod n) \cdot G$
4. Unblind: $Q = Q' - (kr \bmod n) \cdot G$

Note: in step 4, computing $(kr \bmod n) \cdot G$ requires knowing both $k$ and $r$ — which the legitimate holder does. But this requires an additional scalar multiplication for unblinding, doubling the computation. A more efficient variant:

**Alternative — projective coordinate randomisation** (the standard technique):

Instead of blinding the affine point $P = (x, y)$, represent it in randomised projective coordinates. The affine point $(x, y)$ is equivalent to the projective point $(x\lambda : y\lambda : \lambda)$ for any non-zero $\lambda$ (since projective equivalence: $(X:Y:Z) \equiv (X/Z, Y/Z)$ in affine). 

1. Choose random $\lambda \in \mathbb{F}_q^*$ (random element of the base field)
2. Represent $P$ as $(x\lambda, y\lambda, \lambda)$ in projective coordinates
3. Perform all scalar multiplication operations in projective coordinates
4. Convert the result back to affine coordinates at the end

The projective representation of $P$ is randomised by $\lambda$ — different for every execution. The intermediate point values during the scalar multiplication are all scaled by $\lambda$ (or powers thereof), making them unpredictable to a side-channel observer. The final affine result is independent of $\lambda$.

**Why this works**: power analysis and EM analysis attack the predictable bit patterns in field element computations. Randomising the projective representation means the field elements being processed ($x\lambda$, $y\lambda$, $\lambda$) are different each time, even for the same input point $P$. Statistical correlation across traces is destroyed.

---

### Complete ECC Blinding Protocol

For maximum protection, combine both techniques:

1. **Scalar blinding**: replace $k$ with $k' = k + r_1 \cdot n$ (fresh random $r_1$)
2. **Projective coordinate randomisation**: represent $P$ as $(x\lambda : y\lambda : \lambda)$ with fresh random $\lambda$
3. Compute $Q' = k' \cdot (x\lambda : y\lambda : \lambda)$ in projective coordinates
4. Convert $Q'$ to affine: $Q = (X/Z, Y/Z)$ from final projective result
5. Return $Q$ (the correct $k \cdot P$, since blinding factors cancel)

The combination ensures that neither the scalar nor the point representation reveals information about $k$ through timing or power side channels.

**Verification after blinding**: as with RSA-CRT, it is advisable to verify the result: check that $Q = k \cdot P$ is a valid point on the curve (satisfies the curve equation). This detects any fault injection that might be used to bypass the blinding and cause a faulty result that leaks $k$ (see Question 29 on ECC fault attacks).

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.80–84: RSA blinding, timing attack prevention; p.85–87: ECC scalar multiplication, side-channel vulnerabilities)
- IS_UG_2_2_SecM-adv-PQCrypto (p.5–7: side-channel considerations for ECC vs. post-quantum alternatives)

_Status: Complete_  
_Done by: William_
