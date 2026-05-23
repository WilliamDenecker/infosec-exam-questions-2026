# Question 29

We have discussed the vulnerability of RSA to side-channel attacks.

**Discuss the vulnerability of elliptic curve cryptography (ECC) to side-channel attacks. Identify the types of side-channel attacks that can affect ECC implementations and describe the countermeasures that can be employed to mitigate these vulnerabilities.**

## Answer

### Why ECC Is Vulnerable to Side-Channel Attacks (ch2.2.3 p.85–87)

The core operation in ECC is **scalar multiplication**: given a point $G$ and a scalar $k$ (the private key), compute $k \cdot G$ (ch2.2.3 p.85). This is computed using a sequence of point additions and point doublings, typically via a double-and-add algorithm analogous to RSA's square-and-multiply. The sequence of operations depends on the bits of the private scalar $k$, creating opportunities for side-channel leakage.

Side-channel attacks exploit physical information leaked during cryptographic computation — timing, power consumption, electromagnetic emissions — rather than mathematical weaknesses in the algorithm itself.

---

### Type 1 — Timing Attacks

**Vulnerability**: in a naïve double-and-add implementation, a 1-bit in the scalar $k$ causes a point addition to be performed; a 0-bit causes only a doubling. These two operations take different amounts of time:
- Point doubling: $\approx 8$ field multiplications
- Point addition: $\approx 12$ field multiplications (different formula)

By measuring the total execution time of $k \cdot G$ with high precision, an attacker can infer the Hamming weight of $k$ (how many 1-bits it has) or, with careful analysis, recover individual bits.

**Attack variant**: if the same private key $k$ is used across many operations (e.g., a static ECDH private key on a server), an attacker can measure many timing samples, average out noise, and statistically recover $k$ one bit at a time.

**Countermeasure — constant-time implementation**: replace the variable-time double-and-add with a **Montgomery ladder** or **always double-and-add** algorithm that performs the same sequence of operations (both doubling and addition) for every bit of $k$, regardless of whether the bit is 0 or 1. The operation for bit 0 computes a dummy addition that is discarded — the point operation count (and thus timing) is constant across all possible $k$ values.

---

### Type 2 — Simple Power Analysis (SPA)

**Vulnerability**: in hardware implementations, point doubling and point addition consume different amounts of current due to different arithmetic patterns. A power trace (oscilloscope measurement of supply current over time) may directly show the sequence of doublings and additions, revealing the bit pattern of $k$ from a single execution.

**Countermeasure 1 — Unified point addition formulas**: use point addition formulas that work equally for both $P + Q$ (addition) and $P + P$ (doubling) — the same code path regardless. With unified formulas, the power trace no longer distinguishes doublings from additions.

**Countermeasure 2 — Montgomery ladder**: processes both a doubling and addition in every step, making the power trace uniform across all bits of the scalar.

**Countermeasure 3 — Scalar blinding** (randomisation): add a random multiple of the group order $n$ to $k$: $k' = k + r \cdot n$ for random $r$. Since $k' \cdot G = k \cdot G$ (the order of $G$ is $n$, so $r \cdot n \cdot G = 0$), the result is unchanged. But the scalar $k'$ is different in every execution, preventing the attacker from accumulating meaningful traces across runs.

---

### Type 3 — Differential Power Analysis (DPA) and Correlation Power Analysis (CPA)

**Vulnerability**: even when no individual trace reveals $k$, averaging many traces with a statistical hypothesis can extract key bits. DPA exploits the statistical correlation between hypothetical intermediate values (e.g., the value of a specific intermediate point coordinate) and power measurements at specific time points. If the correlation is non-zero for a particular key hypothesis, that hypothesis is correct.

**Why ECC is particularly susceptible**: unlike RSA, ECC private keys are typically small and used repeatedly (especially in ECDH). The same private key signs or decrypts hundreds or thousands of operations, providing the attacker with many traces.

**Countermeasures**:

**Point randomisation**: before each scalar multiplication, randomise the representation of the base point $G$ using a random projective factor: replace $(X:Y:1)$ with $(rX:rY:r)$ for random $r$. The scalar multiplication result is the same, but the intermediate values during computation differ unpredictably across executions, breaking the correlation that DPA exploits.

**Scalar splitting**: split the private scalar $k$ into two random parts $k_1, k_2$ such that $k = k_1 + k_2$, then compute $k \cdot G = k_1 \cdot G + k_2 \cdot G$ separately and add the results. Neither $k_1$ nor $k_2$ alone is $k$; the split changes unpredictably each time, preventing long-term correlation accumulation.

**Field element blinding**: randomise intermediate field elements (coordinates) during the computation so that power traces do not correlate with predictable coordinate values.

---

### Type 4 — Electromagnetic (EM) Analysis

**Vulnerability**: similar to power analysis but uses electromagnetic emissions from the chip rather than supply current. EM probes can be placed very close to specific circuit areas to focus on particular computation units. EM attacks can be more discriminating than power analysis: specific flip-flop transitions or ALU operations may be isolated spatially.

**Countermeasures**: the same techniques as for SPA/DPA — constant-time algorithms, scalar blinding, point randomisation — also reduce EM leakage. Physical shielding (Faraday cage around the chip package) provides additional protection for high-security hardware.

---

### Type 5 — Fault Injection Attacks

**Vulnerability**: by inducing a computational fault (via voltage glitch, clock glitch, laser, or focused ion beam) at a specific moment during the scalar multiplication, an attacker can corrupt one step. With a faulty and a correct result for the same input, it may be possible to recover $k$ — analogous to the RSA-CRT fault attack (see Question 26).

**Countermeasures**: 
- **Verify the result**: after computing $k \cdot G$, verify that the result is a valid point on the curve (satisfies the curve equation). A faulted intermediate result often produces an invalid point that can be detected.
- **Redundant computation**: compute $k \cdot G$ twice with different randomisation and compare results; a fault changes one but not the other.
- **Infective countermeasures**: design the algorithm so that a fault in any intermediate step corrupts the final result in a way that makes it useless to an attacker (rather than informative).

---

### Summary of ECC Side-Channel Vulnerabilities and Countermeasures

| Attack | What is leaked | Countermeasure |
|---|---|---|
| Timing attack | Execution time varies with bits of $k$ | Montgomery ladder / constant-time double-and-add |
| Simple Power Analysis (SPA) | Single trace reveals doubling vs. addition sequence | Unified formulas; Montgomery ladder |
| Differential Power Analysis (DPA) | Statistical correlation across many traces | Scalar blinding; point randomisation; field blinding |
| EM analysis | EM emissions reveal intermediate values | Same as DPA; physical shielding |
| Fault injection | Faulted $\hat{k} \cdot G$ reveals $k$ | Post-computation curve-point validity check; redundant computation |

**Common theme**: all countermeasures aim to make the physical observables (time, power, EM) independent of the private scalar $k$ — either by ensuring identical operation sequences for all $k$ values (constant-time) or by randomising intermediate values so that correlation with $k$ is destroyed.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.85–87: ECC operations, scalar multiplication, side-channel vulnerability context)
- IS_UG_2_2_SecM-adv-PQCrypto (p.5–7: comparison of classical vs. PQC side-channel resistance)

_Status: Complete_  
_Done by: William_
