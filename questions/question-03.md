# Question 3

Most cryptocurrencies use a proof-of-work for the validation of blocks in the blockchain. This proof-of-work consists in computing some hash algorithm. In the case of Bitcoin the hash algorithm is SHA2-256. A major drawback of this kind of hash function is that traditional computer CPUs are at a significant disadvantage both in speed and in power consumption compared to Application-Specific ICs (ASICs), which can be optimised to perform this single task (i.c. computing SHA2-256 hashes).

**Can you suggest alternative hash algorithms that would reduce the advantage of ASICs with respect to CPUs? What properties would be desirable for such hash algorithms?**

*Note: Don't hesitate to think slightly out-of-the-box of traditional cryptographic hash functions, which are mainly intended for digital signatures and should therefore be very fast to compute.*

*Note: For those less familiar with hardware: ASICs are dedicated hardware for a single task. They're not reprogrammable (contrary to FPGAs, CPUs, or GPUs) and only have limited integrated random access memory (much less than the typical L3-cache of a CPU).*

## Answer

### Why ASICs Dominate SHA-256 Mining

SHA-256 has the following properties that make it ideal for ASIC implementation (ch2.2.3 p.30–40):
- Fixed, simple set of operations: bitwise AND, OR, XOR, NOT, 32-bit additions, bit rotations
- No large random-access memory requirement — all state fits in ~256 bits
- Highly parallelisable: many SHA-256 computations are independent of each other
- No branching: purely sequential, deterministic operations with no data-dependent control flow

An ASIC can implement SHA-256's fixed operations in dedicated silicon, achieving orders of magnitude more hashes per second per watt than a general-purpose CPU, which must also support the overhead of an instruction set, memory management, operating system, and many other operations.

A CPU's advantage lies in its large L3 cache (typically 8–64 MB), support for complex branching and memory access patterns, and general-purpose register files. An ASIC has very limited on-chip memory.

---

### Desirable Properties for an ASIC-Resistant Proof-of-Work Hash

To reduce ASIC advantage, the proof-of-work function should exploit precisely the resources where CPUs excel over ASICs:

**Property 1 — Memory-hardness** (the most important property)

The function should require a large amount of random-access memory during computation — far more than an ASIC can integrate on-chip. If the function requires, say, 1 GB of working memory with random access patterns, an ASIC cannot hold this on-chip (on-chip SRAM is extremely expensive in silicon area). The ASIC would need off-chip DRAM, which is slow (100s of nanoseconds per access) and power-hungry — eliminating the ASIC's speed and efficiency advantage. A modern CPU, by contrast, can use its L3 cache (fast, low latency) and system DRAM to satisfy the memory requirement efficiently.

This is directly analogous to the concept used in bcrypt (ch3.2 p.11) and related password-hashing schemes: deliberately increasing computation cost by requiring sequential or random memory accesses to prevent precomputation attacks. Applied to proof-of-work, memory-hardness achieves the same goal but targets ASIC hardware rather than GPU clusters.

**Property 2 — Sequential memory accesses (anti-pipelining)**

ASICs excel at pipelining: they can start the next computation before the previous one finishes, keeping all execution units busy. If the computation has sequential dependencies — where step $i+1$ depends on the output of step $i$, which in turn accesses a memory location determined by step $i$'s output — then pipelining is impossible. Each step must complete before the next can begin. This eliminates the throughput advantage that ASIC pipelining provides.

**Property 3 — Moderately expensive on CPUs, but uniformly so**

The function should be deliberately slower than SHA-256 on all hardware — the goal is not to be fast but to be the bottleneck. However, it must still be computable in reasonable time for legitimate participants (miners with standard hardware). The work factor should be tunable (like bcrypt's cost parameter).

**Property 4 — Asymmetric verification cost**

Ideally, verifying that a proof-of-work is correct should be much faster than computing it. In SHA-256 mining, verification is one SHA-256 call — trivial. The same should hold for any replacement function: the miner does the hard work; the network verifies it quickly. This ensures that verification (done by all nodes) is not a bottleneck even if mining is resource-intensive.

**Property 5 — Collision resistance and preimage resistance**

The function must remain a cryptographic hash function in the traditional sense — preimage resistance (ch2.2.3 p.12), second preimage resistance (ch2.2.3 p.13), and collision resistance (ch2.2.3 p.14) — so that the proof-of-work puzzle cannot be solved without performing the actual computation.

---

### Suggested Alternative: Memory-Hard Hash Function

A suitable alternative is a **memory-hard function** that works as follows (schematically):

1. **Initialisation**: use the block header and nonce to generate an initial seed, then fill a large memory array $A[0 \ldots N-1]$ (e.g., $N = 2^{20}$ 64-byte blocks = 64 MB) using a standard hash function iteratively: $A[i] = H(A[i-1])$. This fill phase is sequential — each entry depends on the previous — so it cannot be parallelised or pipelined.

2. **Random access phase**: perform many iterations where each step reads from a pseudo-random location in $A$ (the location is determined by the current state, which depends on the previous read's output): $state_j = H(state_{j-1} \oplus A[f(state_{j-1})])$. This creates a sequential dependency chain through random memory locations. An ASIC cannot predict which memory locations will be needed — it must wait for each result before knowing the next address, defeating pipeline speculation.

3. **Output**: the final $state$ value after all iterations is the proof-of-work output. The puzzle is to find a nonce such that the output is below a target threshold.

**Why this reduces ASIC advantage**:
- The large memory array $A$ cannot be stored on-chip in an ASIC without enormous silicon cost — ASIC manufacturers would face prohibitive chip area requirements
- The sequential access pattern eliminates pipelining — one memory access at a time, each depending on the previous
- CPUs have large, fast caches that can hold all or part of $A$ close to the processor, dramatically reducing memory access latency compared to ASIC off-chip DRAM

**Remaining limitations**:
- GPUs have some DRAM on-chip — memory-hard functions partly level the field between CPUs and GPUs but may not fully eliminate GPU farms
- Manufacturers will always try to find the most efficient hardware for whatever function is chosen; the goal is only to reduce the ASIC advantage, not eliminate it entirely
- Memory-hard functions trade hash rate for memory usage — they are inherently slower than SHA-256 per unit of hardware

---

### Summary of Desirable Properties

| Property | Why it counters ASICs | CPU advantage exploited |
|---|---|---|
| Memory-hardness (large RAM requirement) | ASICs have very limited on-chip RAM; off-chip DRAM is slow | Large L3 cache and system RAM, low latency |
| Sequential, data-dependent memory access | Eliminates ASIC pipelining; each step waits for previous | Out-of-order execution and speculative caching on CPUs |
| Tunable work factor | Allows difficulty adjustment without redesigning function | Same tunability on all hardware |
| Fast verification | Network nodes verify cheaply; miners bear full cost | Not hardware-specific |
| Standard cryptographic security | Puzzle cannot be shortcut | Not hardware-specific |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.12–14: preimage and collision resistance requirements; p.30–40: hash function construction principles)
- IS_UG_3_2_Appl_AuthMeth (p.11: bcrypt cost-parametrised hashing — same principle of deliberate slowness applied here)

_Status: Complete_  
_Done by: William_
