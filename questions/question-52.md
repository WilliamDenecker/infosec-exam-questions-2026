# Question 52

Rainbow tables provide a time-memory tradeoff (reasonably fast without too much storage) allowing to recover (crack) passwords from a file containing encode passwords.

The "crack.sh" initiative offers a rainbow table approach for the DES encryption of the plaintext "1122334455667788".

**Explain how such a rainbow table could help in cracking DES encryption. What might be the practical limitations? Estimate the effort required to set up this rainbow table. Estimate the storage requirements for this rainbow table. Estimate the time required to crack the encryption of "1122334455667788" with an unknown encryption key.**

## Answer

### Context: DES and Key Space (ch3.2 p.9)

DES (Data Encryption Standard) uses a **56-bit key**. The total key space has $2^{56} \approx 7.2 \times 10^{16}$ possible keys. A brute-force exhaustive search over all keys requires approximately $2^{56}$ DES operations — feasible for a modern GPU cluster but time-consuming.

The crack.sh initiative exploits the fact that the **plaintext is fixed** ("1122334455667788") and the **ciphertext** is observed. The goal is: find key $K$ such that $\text{DES}_K(\texttt{1122334455667788}) = C$ (observed ciphertext).

---

### How Rainbow Tables Help Crack DES (ch3.2 p.9; ch3.7 p.85)

A rainbow table is a **time-memory tradeoff** (TMTO) data structure. Instead of performing $2^{56}$ operations online (at crack time), a large fraction of the computation is done **offline** (precomputed and stored), reducing the online cracking time at the cost of storage.

**Basic principle**:

1. Define a **reduction function** $R_i: \{0,1\}^{64} \to \{0,1\}^{56}$ that maps a 64-bit DES ciphertext (output) to a 56-bit key (input domain). Each position $i$ in the table uses a different reduction function.

2. Build **chains** of alternating DES encryptions and reductions:
$$K_0 \xrightarrow{\text{DES}_K(\text{P})} C_1 \xrightarrow{R_1} K_1 \xrightarrow{\text{DES}_{K_1}(\text{P})} C_2 \xrightarrow{R_2} K_2 \xrightarrow{\cdots} K_t$$
where $\text{P}$ = "1122334455667788" (the fixed plaintext, always the same).

3. Store only the **start key** $K_0$ and **end key** $K_t$ for each chain — the intermediate values are discarded.

4. A **rainbow table** improves on basic Hellman tables by using a **different reduction function at each position** ($R_1, R_2, \ldots, R_t$). This dramatically reduces the number of "merge collisions" between chains (different chains colliding and becoming identical), improving coverage.

**Cracking a given ciphertext $C$**:

For each position $j$ from $t$ down to $1$:
1. Apply $R_j, \text{DES}(\text{P}), R_{j+1}, \text{DES}(\text{P}), \ldots$ to $C$ to obtain a candidate end key
2. Look up the candidate end key in the table (binary search on sorted end keys)
3. If found, regenerate the chain from the stored start key to find the key at position $j-1$ — that is the candidate key $K^*$
4. Verify: check $\text{DES}_{K^*}(\texttt{1122334455667788}) = C$ — if yes, done; if no, it was a false alarm (chain collision)
5. Repeat for all positions

---

### Estimating the Setup Effort (ch3.7 p.85)

**Parameters**:
- Key space size: $N = 2^{56} \approx 7.2 \times 10^{16}$
- Chain length: $t$ (typical values: 10,000 – 100,000)
- Number of chains: $m$ (chosen so $m \times t \approx N$ for good coverage)

For good coverage ($\approx 99\%$ of key space) with chain length $t = 10\,000$:
$$m \approx \frac{N}{t} = \frac{2^{56}}{10\,000} \approx 7.2 \times 10^{12} \text{ chains}$$

Each chain requires $t$ DES operations + $t$ reduction operations during precomputation.

**Total precomputation cost**:
$$m \times t = N \approx 2^{56} \approx 7.2 \times 10^{16} \text{ DES operations}$$

This is the same order of magnitude as brute force — the tradeoff is between precomputation time and lookup time. With modern GPUs performing $\sim 10^9$ DES operations per second:
$$\text{Precomputation time} \approx \frac{7.2 \times 10^{16}}{10^9} \approx 7.2 \times 10^7 \text{ seconds} \approx \mathbf{2.3 \text{ years}}$$

With a large GPU cluster ($\sim 1000$ GPUs): $\approx \mathbf{20 \text{ hours of GPU-cluster time}}$.

This cost is a **one-time investment** — the table is built once and can crack any DES ciphertext of the same fixed plaintext thereafter.

---

### Estimating Storage Requirements

Each chain entry stores (start key, end key): $7 \text{ bytes} + 7 \text{ bytes} = 14 \text{ bytes}$ per chain.

For $m = 7.2 \times 10^{12}$ chains:
$$\text{Storage} \approx 7.2 \times 10^{12} \times 14 \text{ bytes} \approx 1.0 \times 10^{14} \text{ bytes} = \mathbf{100 \text{ TB}}$$

In practice, multiple independent tables (different reduction function families) cover the key space with better coverage:
- crack.sh uses a distributed cluster of machines with a total of several hundred TB of RAID storage

---

### Estimating Time to Crack (Online Phase)

For a single target ciphertext $C$:
- For each position $j$ from $t$ to $1$: apply up to $t$ DES operations + 1 table lookup
- In the worst case (key not found until the last position): $t^2 / 2$ DES operations on average → for $t = 10\,000$: $5 \times 10^7$ operations
- At $10^9$ DES/second: $< 0.1$ seconds per table set

In practice, crack.sh claims cracking times of **seconds to a few minutes** using distributed GPU computation across multiple table sets.

---

### Practical Limitations (ch3.2 p.9–10)

1. **The attack is specific to the fixed plaintext**: every chain link is computed as $\text{DES}_K(\texttt{1122334455667788})$ — the same plaintext at every step. The table is therefore a precomputed structure for inverting the function $f(K) = \text{DES}_K(\texttt{1122334455667788})$ specifically. To use it, you must have a ciphertext $C = \text{DES}_K(\texttt{1122334455667788})$ — i.e., you must know that the target key was used to encrypt exactly that message. If a different plaintext $P'$ was encrypted under the same key, the cracking algorithm would still apply $\text{DES}(\texttt{1122334455667788})$ at each step (because that is what the chains were built with), producing a completely unrelated sequence of values with no connection to $C' = \text{DES}_K(P')$. A separate rainbow table built with plaintext $P'$ would be required. Note: once $K$ is recovered (from a ciphertext of the fixed plaintext), all other ciphertexts under the same key can be decrypted directly — but finding $K$ in the first place requires the matching plaintext-ciphertext pair.

2. **No salt in DES**: standard DES (not the crypt(3) modified version) uses no salt — the same plaintext always gives the same ciphertext under the same key. This is what makes the rainbow table viable. Unix crypt(3) uses a 12-bit salt, which would require $2^{12} = 4096$ separate rainbow tables (ch3.2 p.9 notes).

3. **Storage scale**: 100 TB is substantial but achievable for an organised initiative (crack.sh uses community-contributed hardware). Not feasible for an individual attacker with limited resources.

4. **DES is already obsolete** (ch3.4 p.24): DES is prohibited in modern IPsec (RFC 8221 MUST NOT). The 56-bit key is too short — brute force alone is feasible with a dedicated ASIC (EFF DES Cracker demonstrated this in 1998 in 22 hours). The rainbow table simply makes it faster for the fixed plaintext.

5. **Does not attack the key directly**: the rainbow table finds the key only for the specific plaintext-ciphertext pair. If the key is used for other (different plaintext) encryptions, those remain secure — but the exposed key is then known and all other encryptions can be decrypted directly.

---

### Summary

| Phase | Effort |
|---|---|
| Table setup | $\approx 2^{56}$ DES operations ($\approx$ 20 GPU-cluster-hours) — one time |
| Storage | $\approx$ 100 TB per table set |
| Cracking a single ciphertext | $< 1$ minute (online, using pre-built tables) |

### Sources

- IS_UG_3_2_Appl_AuthMeth (p.9: Unix/Linux passwords — DES-based crypt(3), 12-bit salt, rainbow tables as a threat; p.10: weaknesses — DES vulnerability, offline cracking with rainbow tables)
- IS_UG_3_7_Appl_System (p.85: threshold detection and statistical analysis — reference to precomputed attack tables)
- IS_UG_3_4_Appl_IPSec (p.24: DES obsoleted in ESP — RFC 8221 MUST NOT)

_Status: Complete_  
_Done by: William_
