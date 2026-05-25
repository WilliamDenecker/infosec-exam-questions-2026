# Question 26

It is possible to speed up the computation of an RSA digital signature using the Chinese Remainder Theorem (CRT). However, it is often advised to verify the correctness (simply by verifying the obtained signature) of the signature before publishing the signature.

**Why is it advised to verify the signature before publishing it? To what extent does it reduce the obtained speed-up of the digital signature (*calculate this*)?**

## Answer

### RSA-CRT Digital Signature (ch2.2.3 p.80–84)

#### How CRT Works

**Standard RSA signing** computes $\sigma = m^d \bmod n$ directly — one big exponentiation modulo the full $|n|$-bit modulus with a $|n|$-bit exponent.

**RSA-CRT signing** (ch2.2.3 p.80) exploits the factorisation $n = p \cdot q$. By Fermat's little theorem, for any prime $p$ and $\gcd(m, p) = 1$:
$$m^{p-1} \equiv 1 \pmod{p}$$
Therefore $m^d \equiv m^{d \bmod (p-1)} \pmod{p}$ — we only need the exponent reduced modulo $p-1$, not the full $d$. This gives:

1. $\sigma_p = m^{d_p} \bmod p$ where $d_p = d \bmod (p-1)$ — exponentiation modulo the **half-size** prime $p$
2. $\sigma_q = m^{d_q} \bmod q$ where $d_q = d \bmod (q-1)$ — exponentiation modulo the **half-size** prime $q$
3. **CRT recombination** (Garner's algorithm): find $\sigma$ such that $\sigma \equiv \sigma_p \pmod{p}$ and $\sigma \equiv \sigma_q \pmod{q}$:
$$\sigma = \sigma_q + q \cdot \bigl[q^{-1} \bmod p\bigr] \cdot (\sigma_p - \sigma_q) \bmod n$$
This is a handful of multiplications modulo $n$ — negligible cost.

The combined $\sigma$ is guaranteed to equal $m^d \bmod n$ by the Chinese Remainder Theorem (since $\sigma$ satisfies the correct residues mod both $p$ and $q$, and $\gcd(p,q)=1$).

---

#### Why CRT Gives Exactly 4× Speed-Up

Let $b = \log_2 n = |n|$ denote the **bit-length** of the modulus $n$. The cost of modular exponentiation scales as $O(b^3)$:
- The square-and-multiply algorithm performs $\approx 1.5\,b$ modular multiplications (one squaring per bit, one extra multiply per 1-bit on average)
- Each modular multiplication modulo a $b$-bit number costs $\propto b^2$ bit operations (schoolbook multiplication)
- **Total cost**: $\propto 1.5 \cdot b \cdot b^2 = 1.5\,b^3$

**What is the bit-length of $p$?** Since $n = p \cdot q$ with $p \approx q \approx \sqrt{n}$, the primes satisfy:

$$\log_2 p \approx \log_2 \sqrt{n} = \tfrac{1}{2}\log_2 n = \frac{b}{2}$$

So $p$ has bit-length $b/2$ — **half the bit-length of $n$**. Note this is NOT $\log_2(n/2) = b - 1$ (which would be halving the number itself and barely reduces cost). The CRT substitution replaces $b$ with $b/2$.

Substituting $b/2$ into the cost formula:

$$\text{Cost per sub-exp} = 1.5 \cdot \frac{b}{2} \cdot \left(\frac{b}{2}\right)^2 = 1.5 \cdot \frac{b}{2} \cdot \frac{b^2}{4} = \frac{1.5\,b^3}{8}$$

Each half-size exponentiation is $8\times$ cheaper. The factor of $8 = 2^3$ comes directly from the cubic $O(b^3)$ scaling: halving $b$ reduces cost by $2^3$.

However, CRT requires **both** $S_p$ and $S_q$, so you perform two such sub-exponentiations:

$$\text{Cost}_{CRT} = 2 \times \frac{1.5\,b^3}{8} = \frac{1.5\,b^3}{4}$$

$$\text{Speed-up} = \frac{1.5\,b^3}{\dfrac{1.5\,b^3}{4}} = 4$$

The theoretical $8\times$ per sub-exp is halved because two sub-exps are needed — giving a **4× net speedup**.

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

**Deriving the 17 multiplications for $e = 65537$**:

The square-and-multiply algorithm computes $x^e$ by scanning the bits of $e$ from left to right. The rule is simple:
- **Every bit**: square the running result (doubles the exponent)
- **If the bit is 1**: also multiply by $x$ (adds 1 to the exponent)

Why does this work? Squaring doubles the current exponent, and multiplying by $x$ increments it by 1 — so reading the binary representation left to right builds up $e$ bit by bit, exactly like shifting a binary number left and optionally setting the last bit.

Now apply this to $e = 65537 = 2^{16} + 1$, whose binary representation is:

$$e = \underbrace{1}_{}\underbrace{000000000000000}_{15\ \text{zeros}}\underbrace{1}_{}$$

- **Start**: result $= x$ (the leading 1-bit sets the initial value)
- **Next 15 bits are all 0**: square 15 times, no multiplications needed
  $$x \to x^2 \to x^4 \to \cdots \to x^{2^{15}} = x^{32768}$$
- **Final bit is 1**: square once more, then multiply by $x$
  $$x^{32768} \xrightarrow{\text{square}} x^{65536} \xrightarrow{\times\, x} x^{65537}$$

Total operations:
- 15 squarings (for the 15 zero bits)
- 1 squaring + 1 multiplication (for the final 1-bit)
- **Total: 16 squarings + 1 multiplication = 17 modular multiplications** modulo $n$

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
