# Question 53

In the original RSA encryption scheme the public exponent $e$ and the private exponent $d$ are chosen such that $e$ is coprime to $\phi(n)$ and that $e \cdot d = 1 \mod \phi(n)$, where $n = p \cdot q$.

This encryption scheme has meanwhile been somewhat modified and $e$ and $d$ are now chosen such that $e \cdot d = 1 \mod \lambda(n)$ (without restriction on e), where $\lambda(n)$ must remain secret and

$$\lambda(n) = \mathrm{lcm}(p-1, q-1) = \phi(n)/\gcd(p-1, q-1) \quad (Q53.11)$$

**Explain (with sufficient mathematical detail) how the generation and the verification of a digital signature still works with this modified version of RSA. What are the advantages of this modification? How does this modification affect the computation of a digital signature using the Chines remainder theorem? Why is it recommended that $\gcd(p-1, q-1)$ should be a small value?**

## Answer

### Background: φ(n) vs λ(n) (ch2.2.2 p.15–16, p.26)

In the original RSA, key generation computes $\phi(n) = (p-1)(q-1)$ and selects $d = e^{-1} \bmod \phi(n)$.

The modified version uses **Carmichael's function**:
$$\lambda(n) = \mathrm{lcm}(p-1, q-1) = \frac{(p-1)(q-1)}{\gcd(p-1, q-1)} = \frac{\phi(n)}{\gcd(p-1, q-1)}$$

Since $\lambda(n)$ divides $\phi(n)$ (i.e., $\phi(n) = k \cdot \lambda(n)$ for $k = \gcd(p-1, q-1)$), we have $\lambda(n) \leq \phi(n)$, with equality only when $\gcd(p-1, q-1) = 1$ (impossible for large primes since both $p-1$ and $q-1$ are even).

---

### Why Signing and Verification Still Work (ch2.2.2 p.15–16)

**Mathematical foundation — Carmichael's theorem**:

For any integer $m$ with $\gcd(m, n) = 1$:
$$m^{\lambda(n)} \equiv 1 \pmod{n}$$

This follows because $\lambda(n) = \mathrm{lcm}(p-1, q-1)$, so $\lambda(n)$ is a multiple of both $p-1$ and $q-1$. By Fermat's little theorem:
$$m^{p-1} \equiv 1 \pmod{p} \implies m^{\lambda(n)} \equiv 1 \pmod{p}$$
$$m^{q-1} \equiv 1 \pmod{q} \implies m^{\lambda(n)} \equiv 1 \pmod{q}$$

By CRT (since $\gcd(p,q)=1$): $m^{\lambda(n)} \equiv 1 \pmod{n}$.

**Key generation with $\lambda(n)$**: choose $e$ and $d$ such that:
$$e \cdot d \equiv 1 \pmod{\lambda(n)}$$
i.e., $e \cdot d = 1 + t \cdot \lambda(n)$ for some integer $t$.

**Signing**: $S = M^d \bmod n$

**Verification**:
$$S^e = M^{de} = M^{1 + t \cdot \lambda(n)} = M \cdot \left(M^{\lambda(n)}\right)^t \equiv M \cdot 1^t = M \pmod{n}$$

The last step uses Carmichael's theorem. The scheme works identically to original RSA — the mathematics holds with $\lambda(n)$ in place of $\phi(n)$ because $m^{\lambda(n)} \equiv 1 \pmod{n}$ just as $m^{\phi(n)} \equiv 1 \pmod{n}$.

**Note**: the original RSA proof uses Euler's theorem ($m^{\phi(n)} \equiv 1$). Carmichael's theorem gives a sharper result: $\lambda(n) \leq \phi(n)$, so the same step holds for any multiple of $\lambda(n)$, including $\phi(n)$.

---

### Advantages of This Modification (ch2.2.2 p.26)

**1. Smaller private exponent $d$**:

Since $e \cdot d \equiv 1 \pmod{\lambda(n)}$ and $\lambda(n) \leq \phi(n)$, the private exponent is:
$$d = e^{-1} \bmod \lambda(n) \approx \frac{\lambda(n)}{e}$$

whereas with $\phi(n)$: $d = e^{-1} \bmod \phi(n) \approx \frac{\phi(n)}{e} = \frac{k \cdot \lambda(n)}{e}$.

A smaller $d$ means the **private key operations** (signing, decryption) use a smaller exponent in the square-and-multiply algorithm → fewer multiplications → **faster private key operations**.

**2. Correct and valid keys for all messages**:
The scheme works for all $M$ with $\gcd(M, n) = 1$ (which holds with overwhelming probability for random $M$ given that $p$ and $q$ are large primes). If $\gcd(M, n) \neq 1$, $M$ shares a factor with $n$ — this immediately reveals $p$ or $q$, a catastrophic failure for a different reason.

---

### Impact on CRT Computation (ch2.2.2 p.22)

CRT-accelerated signing computes:
$$S_p = M^{d \bmod (p-1)} \bmod p, \quad S_q = M^{d \bmod (q-1)} \bmod q$$

then combines $S = \text{CRT}(S_p, S_q)$.

**With $\lambda(n)$**: since $e \cdot d \equiv 1 \pmod{\lambda(n)} = \mathrm{lcm}(p-1, q-1)$, this implies in particular:
$$e \cdot d \equiv 1 \pmod{p-1} \quad \text{and} \quad e \cdot d \equiv 1 \pmod{q-1}$$

because $\lambda(n)$ is a multiple of both $(p-1)$ and $(q-1)$. Therefore:
$$d_p = d \bmod (p-1) = e^{-1} \bmod (p-1), \quad d_q = d \bmod (q-1) = e^{-1} \bmod (q-1)$$

**The CRT exponents $d_p$ and $d_q$ are exactly the same** whether $d$ is computed modulo $\phi(n)$ or modulo $\lambda(n)$. The CRT computation is completely unaffected by the change from $\phi(n)$ to $\lambda(n)$.

In fact, the use of $\lambda(n)$ makes the connection between $d$ and the CRT exponents more transparent: $d \bmod (p-1)$ is directly the inverse of $e$ modulo $(p-1)$, which is what the CRT step needs.

---

### Why $\gcd(p-1, q-1)$ Should Be Small (ch2.2.2 p.26)

$\lambda(n) = \phi(n) / \gcd(p-1, q-1)$.

If $\gcd(p-1, q-1)$ is **large** (say $g$):
- $\lambda(n) = \phi(n) / g$ is much smaller than $\phi(n)$
- $d = e^{-1} \bmod \lambda(n)$ is approximately $\lambda(n)/e = \phi(n)/(g \cdot e)$
- For $g$ large, $d$ can become much smaller than $n^{1/4}$

The slide (ch2.2.2 p.26) explicitly warns: "specific faster factorisation algorithms exist when $\gcd(p-1, q-1)$ is small and $\lg(d) < \frac{1}{4}\lg(n)$." This is **Wiener's attack** (continued fraction attack on small $d$): if $d < n^{1/4}/3$, the private exponent can be recovered efficiently from the public key $(e, n)$ using the convergents of the continued fraction expansion of $e/n$.

**Requirement**: $d$ must satisfy $\lg(d) \geq \frac{1}{4}\lg(n)$ — i.e., $d \geq n^{1/4}$.

Since $d \approx \lambda(n)/e$ and $e = 65537 = 2^{16}+1$:
$$d \approx \frac{\phi(n)}{g \cdot e} \geq n^{1/4} \implies g \leq \frac{\phi(n)}{e \cdot n^{1/4}} \approx \frac{n}{e \cdot n^{1/4}} = \frac{n^{3/4}}{e}$$

For RSA-2048 ($n \approx 2^{2048}$, $e = 65537 \approx 2^{17}$): $g \leq 2^{2048 \times 3/4 - 17} = 2^{1519}$ — this bound is very loose. But the concern is more practical: for typical primes, $\gcd(p-1, q-1)$ should be kept small (ideally exactly 2, since $p$ and $q$ are both odd, so $p-1$ and $q-1$ are both even, making $\gcd \geq 2$ always). A value of $\gcd = 2$ gives $\lambda(n) = \phi(n)/2$, and $d$ remains large enough to resist Wiener's attack.

**Recommendation**: choose $p$ and $q$ such that $\gcd(p-1, q-1) = 2$ (or at most a small power of 2). This keeps $\lambda(n) \approx \phi(n)/2$, providing a modest speedup without dangerously reducing $d$.

---

### Summary

| Aspect | φ(n) version | λ(n) version |
|---|---|---|
| Key generation | $d = e^{-1} \bmod \phi(n)$ | $d = e^{-1} \bmod \lambda(n)$ |
| Mathematical basis | Euler's theorem | Carmichael's theorem |
| Signing/verification | Identical | Identical |
| Private exponent size | $\approx \phi(n)/e$ | $\approx \lambda(n)/e \leq \phi(n)/e$ |
| Private key operation speed | Baseline | Faster (smaller $d$) |
| CRT exponents ($d_p$, $d_q$) | Unchanged | Unchanged |
| Risk if $\gcd(p-1,q-1)$ is large | None | Wiener's attack if $d < n^{1/4}$ |

### Sources

- IS_UG_2_2_2_SecM_AsymmEncr (p.15–16: RSA key generation and Euler's theorem; p.16: $M^{ed} \equiv M \pmod{n}$; p.17: digital signature — $S = M^d \bmod n$, verification $S^e = M \bmod n$; p.22: CRT acceleration — $S_p = M^{d \bmod (p-1)} \bmod p$, $S_q = M^{d \bmod (q-1)} \bmod q$; p.26: small private exponent vulnerability — Wiener's attack, $\gcd(p-1,q-1)$ concern)
- IS_UG_2_1_SecM_MathCrypt (Carmichael's function and Carmichael's theorem — algebraic background)

_Status: Complete_  
_Done by: William_
