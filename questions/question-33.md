# Question 33

**Given an RSA private and public key pair (private exponent $d$, public exponent $e$, and public modulus $n$), explain the process of factoring the RSA modulus $n$. Provide a step-by-step explanation of how one can derive the prime factors $p$ and $q$ of $n$ using the private key information.**

## Answer

### 1. The Mathematical Foundation

In RSA, the private key $d$ is the modular inverse of $e$ modulo $\phi(n)$.

This means:

$$
ed \equiv 1 \pmod{\phi(n)} \tag{1}
$$

Therefore, $ed - 1$ must be a multiple of $\phi(n)$. We can write this as:

$$
ed - 1 = k \cdot \phi(n) \tag{2}
$$

Because $p$ and $q$ are odd primes, both $(p-1)$ and $(q-1)$ are even numbers. Therefore

$$
\phi(n) = (p-1)(q-1) \tag{3}
$$

$\phi(n)$ is always an even number.

Because (2) and  $k \cdot \phi(n)$ is even, $ed - 1$ is also even. Any even number can be factored into a power of 2 multiplied by an odd number. So, we can rewrite $ed - 1$ as:

$$
ed - 1 = 2^s t \tag{4}
$$

where $s \ge 1$ and $t$ is an odd integer.

---

### 2. The Factoring Algorithm (Step-by-Step)

#### Step 1

Choose a random integer $a$ from the set $\mathbb{Z}_n^*$ (meaning $1 < a < n$ and $\gcd(a,n)=1$).

#### Step 2

Compute $a^t \pmod n$ and then repeatedly square the result to form a sequence:

$$
a^t, a^{2t}, a^{4t}, \dots, a^{2^s t} \pmod n \tag{5}
$$

#### Step 3

According to Euler's Theorem, the final term in this sequence will always be $1 \pmod n$ because using (4):

$$
a^{2^s t} = a^{ed-1} = a^{k\phi(n)} \equiv 1 \pmod n \tag{6}
$$

#### Step 4

Walk backwards through your sequence of squares. You are looking for a specific step index $i$ (where $1 \le i \le s$) such that squaring a number gives you $1$, but the number itself is not $1$ and not $-1 \pmod n$.

Mathematically, you want to find an

$$
x = a^{2^{i-1}t} \tag{7}
$$

where:

$$
x \not\equiv \pm 1 \pmod n
$$

and

$$
x^2 \equiv 1 \pmod n \tag{8}
$$

#### Step 5

If you find such an $x$, you have found a non-trivial square root of $1$ modulo $n$.

We can rewrite (8) as:

$$
x^2 - 1 \equiv 0 \pmod n \tag{9}
$$

$$
(x - 1)(x + 1) \equiv 0 \pmod n \tag{10}
$$

This means $n$ perfectly divides the product $(x - 1)(x + 1)$.

#### Step 6

Since

$$
x \not\equiv 1 \pmod n \tag{11}
$$

and

$$
x \not\equiv -1 \pmod n \tag{12}
$$

we know that $n$ does not evenly divide $(x-1)$ or $(x+1)$ individually.

Instead, one of $n$'s prime factors ($p$) is hidden in $(x-1)$ and the other ($q$) is hidden in $(x+1)$.

#### Step 7

You can extract these prime factors by computing the Greatest Common Divisor (GCD):

$$
p = \gcd(x - 1, n) \tag{13}
$$

$$
q = \gcd(x + 1, n) \tag{14}
$$

(Or only compute one of them and then divide $\frac n p $ to obtain the other one, this is faster.)

---

### 3. Efficiency of the Attack

This attack is highly efficient. For at least half of the randomly chosen values of

$$
a \in \mathbb{Z}_n^* \tag{15}
$$

the sequence will reveal a non-trivial factor.

This means the expected number of random guesses needed to successfully factor $n$ is only about 2 tries.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.77–85: RSA algorithm, $\phi(n)$, private key definition, Euler's theorem; p.18: factoring from private key — mentioned in lecture)

_Status: Complete_  
_Done by: William_
