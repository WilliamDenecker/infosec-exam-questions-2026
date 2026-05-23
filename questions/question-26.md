# Question 26

It is possible to speed up the computation of an RSA digital signature using the Chinese Remainder Theorem (CRT). However, it is often advised to verify the correctness (simply by verifying the obtained signature) of the signature before publishing the signature.

**Why is it advised to verify the signature before publishing it? To what extent does it reduce the obtained speed-up of the digital signature (*calculate this*)?**

## Answer

### RSA-CRT Digital Signature (ch2.2.3 p.80–84)

**Standard RSA signing**: compute $\sigma = m^d \bmod n$, where $n = p \cdot q$ is the RSA modulus and $d$ is the private exponent. The exponent $d$ is approximately $|n|$ bits long (e.g., 2048 bits). This requires approximately $1.5 \cdot |n|$ modular multiplications using square-and-multiply.

**RSA-CRT signing** (ch2.2.3 p.80): use the Chinese Remainder Theorem to compute the signature modulo $p$ and $q$ separately, then combine:

1. Compute $\sigma_p = m^{d_p} \bmod p$, where $d_p = d \bmod (p-1)$
2. Compute $\sigma_q = m^{d_q} \bmod q$, where $d_q = d \bmod (q-1)$
3. Combine with CRT: $\sigma = CRT(\sigma_p, \sigma_q)$ — a single fast combination step

**Speed-up**: each of $p$ and $q$ is approximately $|n|/2$ bits (for $n = pq$ balanced). The exponents $d_p$ and $d_q$ are also $|n|/2$ bits long. Each modular exponentiation mod $p$ (or mod $q$) requires approximately $1.5 \cdot |n|/2$ modular multiplications with operands of size $|n|/2$ bits.

The cost of one modular multiplication modulo $p$ (a $|n|/2$-bit modulus) is approximately $(|n|/2)^2$ bit operations. The cost of one multiplication modulo $n$ (a $|n|$-bit modulus) is approximately $|n|^2$ bit operations — a factor of 4 more expensive.

**Total cost with CRT**:
- 2 exponentiations, each with $\approx 1.5 \cdot |n|/2$ multiplications, each costing $(|n|/2)^2$:
  $$\text{Cost}_{CRT} \approx 2 \cdot 1.5 \cdot \frac{|n|}{2} \cdot \left(\frac{|n|}{2}\right)^2 = 2 \cdot 1.5 \cdot \frac{|n|^3}{8} = \frac{1.5 \cdot |n|^3}{4}$$

**Total cost without CRT**:
- 1 exponentiation with $\approx 1.5 \cdot |n|$ multiplications, each costing $|n|^2$:
  $$\text{Cost}_{standard} \approx 1.5 \cdot |n| \cdot |n|^2 = 1.5 \cdot |n|^3$$

**Speed-up factor**:
$$\text{Speed-up} = \frac{\text{Cost}_{standard}}{\text{Cost}_{CRT}} = \frac{1.5 \cdot |n|^3}{\frac{1.5 \cdot |n|^3}{4}} = 4$$

RSA-CRT signing is approximately **4× faster** than standard RSA signing.

---

### Why Verification Before Publishing Is Advised

**Fault attacks on RSA-CRT** (ch2.2.3 p.84):

The CRT computation is vulnerable to **fault injection attacks**. If a computational error occurs in one of the CRT subcomputations — whether due to hardware fault, cosmic ray bit flip, temperature instability, or deliberate fault injection by an attacker — the resulting incorrect signature reveals the secret factor $p$ or $q$:

**Why a faulty CRT signature leaks the private key**:

Suppose an error occurs in the computation of $\sigma_p$ — the result is $\hat{\sigma}_p \neq m^{d_p} \bmod p$, but $\sigma_q$ is correct. The combined CRT signature $\hat{\sigma}$ is wrong:
- $\hat{\sigma} \equiv m^d \pmod{q}$ (correct part from $\sigma_q$)
- $\hat{\sigma} \not\equiv m^d \pmod{p}$ (wrong part from $\hat{\sigma}_p$)

The verifier can compute $m' = \hat{\sigma}^e \bmod n$ (verification) and will find $m' \neq m$.

Now, an attacker who observes the faulty signature $\hat{\sigma}$ alongside a known correct signature $\sigma$ (or by simply computing the verification themselves) can compute:
$$\gcd(\hat{\sigma}^e - m \bmod n, n) = p \quad \text{(or } q\text{)}$$

Because $\hat{\sigma}^e \equiv m \pmod{q}$ (correct), so $\hat{\sigma}^e - m \equiv 0 \pmod{q}$, meaning $q \mid (\hat{\sigma}^e - m)$. But $q \nmid n/p$... wait — $n = pq$, so $\gcd(\hat{\sigma}^e - m \bmod n, n)$ will be $q$ (the factor for which the signature is correct). This directly reveals $q = \gcd(\hat{\sigma}^e - m, n)$, and then $p = n/q$.

From $p$ and $q$, the attacker immediately computes $\phi(n) = (p-1)(q-1)$ and then $d = e^{-1} \bmod \phi(n)$ — the complete private key.

**This attack was discovered by Boneh, DeMillo, and Lipton (1997)** and is known as the **BDL fault attack**. It requires only one faulty signature to completely recover the RSA private key.

**Prevention — verify before publishing**:

Before publishing a signature $\sigma$ computed with CRT, the signer verifies it: compute $\sigma^e \bmod n$ and check that it equals the original message $m$ (or hash). This step:
1. Detects any faulty computation before it is exposed to an attacker
2. Prevents publication of the faulty signature that would enable the BDL attack
3. Has negligible security cost (verification does not reveal $d$)

If verification fails, the signature is discarded and recomputed — at the cost of the computation time, with no security loss.

---

### Cost of Verification and Reduced Speed-Up

**Verification** consists of computing $\sigma^e \bmod n$ — an RSA public key operation with exponent $e$.

The public exponent $e = 65537 = 2^{16} + 1$ has a very specific bit structure: in binary it is $1\underbrace{00\ldots0}_{15}1$ — exactly 2 bits set to 1. Using square-and-multiply:
- 16 squarings (for the 16-bit shift from the leading 1 to the trailing 1)
- 1 multiplication (for the trailing 1 bit)
- Total: **17 modular multiplications** modulo $n$ (full $|n|$-bit modulus)

This is negligible compared to either the standard or CRT signature computation ($\approx 1.5 \cdot |n|$ multiplications).

**Calculation of net speed-up**:

Total cost with CRT + verification:
$$\text{Cost}_{CRT+verify} = \frac{1.5 \cdot |n|^3}{4} + 17 \cdot |n|^2$$

For $|n| = 2048$ bits:
- CRT cost: $\frac{1.5 \times 2048^3}{4} \approx 3.2 \times 10^9$ unit operations
- Verification cost: $17 \times 2048^2 \approx 7.1 \times 10^7$ unit operations

The verification adds approximately $\frac{7.1 \times 10^7}{3.2 \times 10^9} \approx 2.2\%$ overhead.

**Net speed-up** compared to standard RSA signing (no CRT, no verify):
$$\text{Net speed-up} = \frac{1.5 \cdot |n|^3}{\frac{1.5 \cdot |n|^3}{4} + 17 \cdot |n|^2} = \frac{4}{1 + \frac{17 \times 4}{1.5 \cdot |n|}} = \frac{4}{1 + \frac{68}{1.5 \times 2048}} \approx \frac{4}{1 + 0.022} \approx 3.91$$

**Conclusion**: verification reduces the CRT speed-up from exactly $4\times$ to approximately $3.9\times$ — a negligible reduction (approximately 2%) that is strongly justified by the security protection it provides against private key recovery via fault attacks.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.80–84: RSA-CRT optimisation, fault attacks, key recovery from faulty signatures)

_Status: Complete_  
_Done by: William_
