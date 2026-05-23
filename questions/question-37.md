# Question 37

There is a variant for RSA encryption (*multi-prime RSA*), where the modulus $n$ is a product of a certain number ($k > 2$) of different primes $p_i$ ($1 \le i \land i \le k$).

$$n = \prod_{i=1}^{k} p_i \quad (Q37.8)$$

You may assume that all prime factors $p_i$ are of the same order of magnitude.

The RSA-operations are completely similar: $C = M^e \mod n$ for the public key encryption of a message $M$ into a ciphertext $C$ using the public exponent $e$; $M = C^d \mod n$ for the decryption using the private exponent $d$, where $e \cdot d = 1 \mod \phi(n)$.

The modulus $n$ can, of course, be factorised using the general number field sieve (GNFS), but it can also be factorised using the elliptic curves method (ECM). The complexity for factoring using ECM is:

$$(\lg n)^2 \cdot L_p[1/2, \sqrt{2}] \quad (Q37.9)$$

where $p$ is a prime factor of $n$.

Consider the case of a 8192 bit RSA key. On the one hand you have a multi-prime RSA key, where the modulus is the product of four 2048 bit primes. On the other hand you have a traditional 8192 bit RSA key, where the modulus is the product of two 4096 bit primes.

**To what extent could a multi-prime RSA key be faster than a traditional RSA key? Consider key generation. Consider encryption (using the public key) and decryption (using the private key). Explain how the performance improvement is obtained.**

*Hint: Think of what you could do using the Chinese Remainder Theorem (CRT).*

*Note: Don't worry about the security of the multi-prime RSA key for the case considered here. There is no significant degradation of the security with respect to a traditional RSA key of same length.*

## Answer

### Setup (ch2.2.3 p.77–84)

**Traditional RSA-8192**: $n = p \cdot q$ where $p, q$ are 4096-bit primes. All RSA computations work modulo $n$ (8192 bits).

**Multi-prime RSA-8192 ($k = 4$)**: $n = p_1 \cdot p_2 \cdot p_3 \cdot p_4$ where all four primes are 2048-bit. Euler's totient: $\phi(n) = (p_1-1)(p_2-1)(p_3-1)(p_4-1)$. The private exponent $d$ satisfies $ed \equiv 1 \pmod{\phi(n)}$.

**Cost model**: the cost of one modular multiplication modulo an $b$-bit number scales as $O(b^2)$ bit operations (schoolbook multiplication; more precisely with advanced algorithms, but $b^2$ captures the key scaling). A modular exponentiation with an $b$-bit exponent requires $\approx 1.5b$ multiplications using square-and-multiply.

---

### Encryption (Public Key Operation)

**Encryption**: $C = M^e \bmod n$. The public exponent $e = 65537 = 2^{16} + 1$ is used for both variants. Encryption with $e$ requires 17 modular multiplications (square-and-multiply with 17-bit exponent having 2 set bits), all modulo $n$.

For both the traditional and multi-prime RSA with $n$ of the same bit length (8192 bits), the encryption operation is **identical** — both compute $M^e \bmod n$ with the same 8192-bit modulus $n$. The internal structure of $n$ (2 primes vs. 4 primes) is irrelevant for public key operations, since encryption only uses $n$, not the prime factors.

**Encryption: no difference between traditional and multi-prime RSA.**

---

### Decryption (Private Key Operation — CRT Speedup)

**Standard decryption (no CRT)**: $M = C^d \bmod n$. Exponent $d$ is approximately $|n| = 8192$ bits. Cost: $\approx 1.5 \times 8192$ multiplications, each of cost $(8192)^2$. Total: $\approx 1.5 \times 8192^3$ bit operations.

**CRT decryption** applies the **Generalized Chinese Remainder Theorem** (ch2.2.3 p.80–84) to speed up private key operations. For $k$ prime factors:

For each prime $p_i$, compute the partial private exponent: $d_i = d \bmod (p_i - 1)$.

Compute $k$ partial decryptions, each modulo $p_i$:
$$M_i = C^{d_i} \bmod p_i$$

Each $d_i$ is approximately $|p_i|$ bits, and the modulus $p_i$ is $|p_i|$ bits. The cost of one partial decryption is approximately $1.5 \times |p_i|$ multiplications of cost $|p_i|^2$, giving:
$$\text{Cost of one partial decryption} \approx 1.5 \times |p_i|^3$$

For $k$ prime factors: total cost = $k \times 1.5 \times |p_i|^3$.

Then combine the $k$ partial results using the CRT recombination step (Garner's algorithm or similar), which requires $O(k^2)$ multiplications modulo $n$ — this is small compared to the exponentiation steps.

**Speed-up calculation for traditional RSA ($k = 2$, 4096-bit primes)**:

$$\text{Speed-up}_{k=2} = \frac{1.5 \times 8192^3}{2 \times 1.5 \times 4096^3} = \frac{8192^3}{2 \times 4096^3} = \frac{2^{39}}{2 \times 2^{36}} = \frac{2^{39}}{2^{37}} = 4$$

Standard CRT with 2 primes gives a **4× speedup** (same as in Question 26).

**Speed-up calculation for multi-prime RSA ($k = 4$, 2048-bit primes)**:

$$\text{Speed-up}_{k=4} = \frac{1.5 \times 8192^3}{4 \times 1.5 \times 2048^3} = \frac{8192^3}{4 \times 2048^3} = \frac{2^{39}}{4 \times 2^{33}} = \frac{2^{39}}{2^{35}} = 16$$

Multi-prime RSA with 4 primes gives a **16× speedup** over standard RSA decryption.

**Alternatively**: compare multi-prime RSA decryption to traditional CRT RSA decryption:
$$\text{Ratio} = \frac{2 \times 1.5 \times 4096^3}{4 \times 1.5 \times 2048^3} = \frac{2 \times 4096^3}{4 \times 2048^3} = \frac{2 \times 2^{36}}{4 \times 2^{33}} = \frac{2^{37}}{2^{35}} = 4$$

Multi-prime RSA with 4 primes is **4× faster than traditional RSA with CRT (2 primes)**.

**General formula**: for $k$ equal prime factors, each of size $|n|/k$ bits, the CRT speedup over direct exponentiation is $k^2$ — the exponent size halves each time the number of primes doubles (because $d_i = d \bmod (p_i - 1) \approx |n|/k$ bits), and each multiplication cost also halves.

---

### Key Generation

**Traditional RSA**: generate two 4096-bit primes $p, q$. Primality testing of a 4096-bit candidate requires $(4096)^2 \cdot k_{rounds}$ bit operations per Miller-Rabin round.

**Multi-prime RSA**: generate four 2048-bit primes $p_1, p_2, p_3, p_4$. Each 2048-bit candidate primality test costs $(2048)^2 \cdot k_{rounds}$ — a factor of 4 cheaper per candidate.

Expected number of candidates to test for a random prime of $b$ bits $\approx b \cdot \ln 2$ (by the prime number theorem). For 4096-bit primes: $\approx 4096 \times 0.693 \approx 2839$ candidates. For 2048-bit primes: $\approx 2048 \times 0.693 \approx 1419$ candidates.

**Rough comparison (one prime of each size)**:
- Cost of generating one 4096-bit prime: $\propto 2839 \times 4096^2 \approx 4.76 \times 10^{10}$
- Cost of generating one 2048-bit prime: $\propto 1419 \times 2048^2 \approx 5.95 \times 10^9$
- Ratio: $4096^2 \times 2 \approx 8 \times 2048^2$ (approximately 8× more expensive per 4096-bit prime)

Traditional RSA generates 2 primes of 4096 bits; multi-prime generates 4 primes of 2048 bits:
- Traditional: $2 \times 4.76 \times 10^{10} \approx 9.5 \times 10^{10}$
- Multi-prime: $4 \times 5.95 \times 10^9 \approx 2.38 \times 10^{10}$

Multi-prime key generation is approximately **4× faster** than traditional 8192-bit RSA key generation, because generating four 2048-bit primes is cheaper than generating two 4096-bit primes (each 4096-bit prime costs $\approx 8\times$ a 2048-bit prime, but you only generate 4 instead of 2, giving net 4× speedup).

---

### Summary

| Operation | Traditional RSA-8192 (2×4096-bit primes) | Multi-prime RSA-8192 (4×2048-bit primes) | Speedup of multi-prime |
|---|---|---|---|
| Key generation | 2 × 4096-bit prime search | 4 × 2048-bit prime search | ~4× faster |
| Encryption ($M^e$) | Same (both use 8192-bit $n$) | Same (both use 8192-bit $n$) | **No difference** |
| Decryption (CRT) | 2 × exp mod 4096-bit | 4 × exp mod 2048-bit | **4× faster** (16× vs. no-CRT) |
| Decryption (no CRT) | exp mod 8192-bit | exp mod 8192-bit | **No difference** |

**How the speedup is obtained**: the CRT decomposes the large modular exponentiation (modulo $n$, expensive) into smaller exponentiations (modulo each $p_i$, cheap). The cost of modular exponentiation scales as $O(b^3)$ for $b$-bit modulus and exponent, so halving the modulus size reduces cost by $2^3 = 8\times$ per prime; combining this with the factor-of-$k$ increase in the number of exponentiations gives a net speedup of $k^3 / k = k^2$ — specifically $4^2 = 16\times$ for $k = 4$ compared to no CRT, and $4\times$ compared to traditional 2-prime CRT.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.77–84: RSA algorithm, CRT speedup, multi-prime generalisation; p.80: prime generation cost; the $k^2$ speedup formula)

_Status: Complete_  
_Done by: William_
