# Question 30

Learning With Errors (LWE) is a fundamental hard problem underlying several post-quantum schemes.

**Explain the following things**

- **what is meant by *quantum safety*, and discuss the limitations of this notion;**
- **why LWE is believed to be a quantum safe algorithm;**
- **how LWE enables public key encryption;**
- **the role of noise in LWE;**
- **how LWE is related to lattice problems.**

## Answer

### What Is Meant by Quantum Safety — and Its Limitations (ch2 PQCrypto p.1–10)

**Quantum safety (quantum hardness)** means that a computational problem is believed to require exponential time on both classical and quantum computers — no polynomial-time algorithm (classical or quantum) is known for it.

Classical public-key cryptography (RSA, ECC) rests on problems that are quantum-unsafe: Shor's algorithm solves the integer factorisation problem and the discrete logarithm problem in polynomial time on a quantum computer (ch2 PQCrypto p.5–7). A cryptographically relevant quantum computer (CRQC) would therefore break all RSA and ECC-based systems.

A quantum-safe (post-quantum) cryptographic scheme is based on a problem for which no polynomial-time quantum algorithm is known. The best known quantum attack is Grover's algorithm, which provides only a quadratic speedup (square root of the brute force search space) — not the exponential speedup of Shor's (ch2 PQCrypto p.16). Against LWE-based schemes, Grover's algorithm reduces the effective security by a factor of 2 in bit terms (e.g., 256-bit classical security → 128-bit post-quantum security), which is why post-quantum schemes use larger parameters.

**Limitations of "quantum safety"** (ch2 PQCrypto p.8):

1. **No proof of quantum hardness**: "quantum safe" means "no efficient quantum algorithm is known." It does not mean "provably hard for quantum computers." No mathematical theorem rules out the existence of a polynomial-time quantum algorithm for LWE, SVP, or other PQC hard problems. The claim is a conjecture based on current knowledge.

2. **New quantum algorithms may be discovered**: quantum algorithmics is a young field. Future breakthroughs (analogous to how Shor's algorithm surprised the cryptographic community in 1994) could target PQC problems.

3. **Algorithm-specific**: quantum safety is a property of specific mathematical problems, not of "quantum computers in general." A scheme may be safe against Shor's and Grover's but vulnerable to a future quantum algorithm designed for its specific algebraic structure.

4. **Parameter uncertainty**: the specific parameter choices (lattice dimension, error distribution) that achieve claimed security levels are based on the best current classical and quantum cryptanalysis. New attacks could require larger parameters — or reveal that current parameters are insufficient.

---

### Why LWE Is Believed to Be Quantum Safe (ch2 PQCrypto p.10–12)

**Learning With Errors (LWE)** was introduced by Regev (2005). The hardness of LWE is:

1. **No polynomial-time classical algorithm known**: despite intensive study, the best known classical algorithm for solving LWE runs in exponential time in the lattice dimension $n$. This places it in a different category from factoring (broken by Shor's) or discrete log.

2. **No polynomial-time quantum algorithm known**: Shor's algorithm exploits the hidden subgroup problem structure in groups — a structure not present in LWE. Grover's algorithm provides at most a quadratic speedup, not a polynomial-time solution. No quantum algorithm achieves polynomial-time LWE decoding.

3. **Reduction from hard lattice problems**: LWE has a remarkable property — Regev proved that solving LWE (on average, for random instances) is at least as hard as solving certain lattice problems (Shortest Vector Problem, SVP; Bounded Distance Decoding, BDD) in the worst case. This means breaking LWE in practice would imply breaking these worst-case lattice problems, which would be a fundamental breakthrough in computational geometry. Lattice problems are believed to be hard even for quantum computers.

4. **NIST standardisation (2024)**: after years of competition and cryptanalysis by the global community, NIST standardised CRYSTALS-Kyber (key encapsulation) and CRYSTALS-Dilithium (signatures) — both based on LWE variants — as primary post-quantum standards. This reflects broad expert consensus on their security.

---

### How LWE Enables Public Key Encryption (ch2 PQCrypto p.10–12)

**The LWE problem**: given $m$ samples of the form $(a_i, b_i)$ where $a_i \in \mathbb{Z}_q^n$ (random vector) and $b_i = \langle a_i, s \rangle + e_i \bmod q$ (inner product with secret vector $s$ plus small error $e_i$), find $s$.

Without the error: $b_i = \langle a_i, s \rangle \bmod q$ is just a system of linear equations, solvable efficiently by Gaussian elimination. The error $e_i$ is what makes LWE hard.

**LWE-based public key encryption** (simplified construction):

**Key generation**:
- Choose a secret vector $s \in \mathbb{Z}_q^n$ (the private key)
- Sample random matrix $A \in \mathbb{Z}_q^{m \times n}$ and small error vector $e \in \mathbb{Z}_q^m$
- Compute $b = As + e \bmod q$
- **Public key**: $(A, b)$; **Private key**: $s$

**Encryption** (of a bit $\mu \in \{0, 1\}$):
- Sample random $r \in \{0,1\}^m$ (a random binary vector selecting a subset of rows)
- Compute: $u = r^T A \bmod q$ and $v = r^T b + \lfloor q/2 \rfloor \cdot \mu \bmod q$
- **Ciphertext**: $(u, v)$

**Decryption**:
- Compute: $v - u \cdot s = r^T b + \lfloor q/2 \rfloor \mu - r^T A s \bmod q$
  $= r^T (As + e) + \lfloor q/2 \rfloor \mu - r^T A s$
  $= r^T e + \lfloor q/2 \rfloor \mu$
- If the noise $r^T e$ is small compared to $q/2$, then:
  - If $\mu = 0$: result is close to 0 → round to 0
  - If $\mu = 1$: result is close to $q/2$ → round to 1
- The decoded bit is $\mu$

**Security**: an attacker who sees $(A, b)$ cannot determine $s$ (LWE hardness). Seeing the ciphertext $(u, v)$ does not help without $s$.

---

### The Role of Noise in LWE (ch2 PQCrypto p.10–12)

The noise $e_i$ is **essential for LWE security**:

**Without noise**: $b = As \bmod q$ is a system of linear equations. Given public key $(A, b)$, the private key $s$ is the solution — recoverable by Gaussian elimination in polynomial time. The encryption scheme would be trivially broken.

**With noise**: $b = As + e \bmod q$ is no longer a simple linear system. The small error destroys the algebraic structure that would allow efficient recovery of $s$. Recovering $s$ requires solving the Bounded Distance Decoding (BDD) problem — finding the lattice vector closest to $b$ — which is believed to be computationally hard.

**Noise parameters matter critically**:
- Noise too small: the errors are negligible; Gaussian elimination still approximately recovers $s$
- Noise too large: decryption fails — the term $r^T e$ may be larger than $q/4$, causing incorrect rounding
- The noise distribution (typically Gaussian or discrete Gaussian with standard deviation $\sigma$) must be chosen to balance security (large enough to prevent recovery of $s$) with correctness (small enough for successful decryption)

The noise is always drawn from a small distribution — in practice, $|e_i| \ll q$ (e.g., $\sigma \approx 3$ for $q \approx 3329$ in Kyber). The noise is computationally imperceptible to an attacker who does not know $s$, but decodable by the private key holder through the $v - u \cdot s$ computation.

---

### How LWE Is Related to Lattice Problems (ch2 PQCrypto p.8–12)

**A lattice** is a discrete additive subgroup of $\mathbb{R}^n$ — the set of all integer linear combinations of $n$ basis vectors. Lattice problems ask about the geometric structure of lattices:

- **Shortest Vector Problem (SVP)**: find the shortest non-zero vector in a lattice
- **Closest Vector Problem (CVP)**: given a target point, find the nearest lattice point
- **Bounded Distance Decoding (BDD)**: a variant of CVP where the target is promised to be within a small distance of some lattice point

**Connection**: the public key $(A, b)$ in LWE defines a lattice — the set of all vectors of the form $As' \bmod q$ for $s' \in \mathbb{Z}_q^n$. The vector $b = As + e$ is a point close to the lattice (at distance $\|e\|$ from the nearest lattice point $As$). Recovering $s$ from $(A, b)$ is equivalent to finding the lattice vector closest to $b$ — this is exactly the BDD problem.

**Regev's worst-case to average-case reduction**: Regev proved that solving LWE on random instances (as required to break the encryption scheme) is at least as hard as solving certain lattice problems in the **worst case** over all lattice instances. This is a much stronger guarantee than typical cryptographic hard problems, which only have average-case hardness. It means:
- There exist no "easy" LWE instances in a random draw — all instances are approximately equally hard
- Breaking LWE would imply a polynomial-time algorithm for SVP or related problems in the worst case — a result that would overturn decades of computational geometry

This reduction from worst-case lattice hardness is the primary theoretical justification for LWE's security and is one of the strongest security arguments in all of cryptography.

### Sources

- IS_UG_2_2_SecM-adv-PQCrypto (p.1–16: post-quantum threat model; Shor's and Grover's algorithms; LWE problem definition; lattice problems; NIST PQC standardisation; quantum safety limitations)

_Status: Complete_  
_Done by: William_
