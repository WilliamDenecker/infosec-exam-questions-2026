# Question 7

Blockchain consensus relies on the assumption that the longest available chain is the correct one.

**Explain the longest-chain rule used in blockchains such as Bitcoin.**

**How do forks arise, and how are they resolved?**

**Discuss under which assumptions this rule is secure, and what happens if these assumptions no longer hold.**

## Answer

### The Longest-Chain Rule

A blockchain (ch2.2.3 p.84 — Merkle trees and hash chaining) is a sequence of blocks where each block contains:
- A set of transactions
- A hash of the previous block (creating the chain)
- A nonce and a proof-of-work value

The **longest-chain rule**: when a node receives multiple competing chains (which can occur due to forks), it considers the chain with the **most accumulated proof-of-work** — in practice, the longest chain — as the canonical, authoritative chain. All nodes follow this rule independently and autonomously.

**Why the longest chain is trusted**: producing a valid block requires solving a proof-of-work puzzle (finding a nonce such that $SHA256(SHA256(block\_header)) < target$). This requires enormous computational effort. A longer chain represents more cumulative computational work — more blocks solved, more energy expended. An attacker who wants to present a fraudulent longer chain must therefore outperform the entire honest network in computational power.

**Transaction finality**: a transaction included in block $B$ is considered final when $k$ additional blocks have been added on top of it ($k$ confirmations). With each additional block, the probability that the block containing the transaction will be reversed decreases exponentially. Bitcoin conventionally uses 6 confirmations for high-value transactions.

---

### How Forks Arise

**Type 1 — Accidental forks (temporary forks)**:

Two or more miners solve the proof-of-work puzzle at approximately the same time and broadcast their valid blocks to the network simultaneously. Due to network propagation delays, different parts of the network receive different blocks first. For a brief period, two valid competing chains of equal length coexist:

```
... ← Block_N ← Block_A   (found by Miner A, received first by some nodes)
... ← Block_N ← Block_B   (found by Miner B, received first by other nodes)
```

The next miner who successfully mines a new block adds it to whichever chain they received first. This makes one chain longer:

```
... ← Block_N ← Block_A ← Block_A+1
... ← Block_N ← Block_B             ← orphaned
```

All nodes adopt the longer chain. Block_B becomes an **orphaned block** — it was a valid block but is no longer part of the canonical chain. Transactions in Block_B that were not also in Block_A re-enter the memory pool and will be included in a future block.

**Type 2 — Protocol forks (intentional forks)**:

- **Soft fork**: a backward-compatible protocol upgrade. New rules are strictly more restrictive than old rules — blocks valid under new rules are also valid under old rules. Old nodes follow the new longer chain without realising it is under new rules.
- **Hard fork**: an incompatible protocol upgrade. New blocks are not valid under old rules. Old nodes reject new blocks and vice versa. The network permanently splits into two separate chains with separate coins (e.g., Bitcoin vs Bitcoin Cash). Requires coordinated upgrade across all participants.

---

### How Forks Are Resolved

**Accidental forks** resolve automatically through the longest-chain rule:
1. Each node mines on the chain tip it received first
2. The next successfully mined block makes one chain longer
3. All nodes switch to the longer chain (they received it and verified it has more proof-of-work)
4. The shorter chain's tip becomes an orphaned block
5. Resolution is typically complete within one or two block intervals (~10 minutes per block in Bitcoin)

**Protocol hard forks** do not resolve — they result in permanent chain divergence. Both chains continue independently with separate communities, miners, and coin values.

---

### Security Assumptions (and What Happens When They Fail)

**Assumption 1 — Honest majority of hash power (>50%)**:

The longest-chain rule is secure under the assumption that honest miners collectively control more than 50% of the total network hash power (ch2.2.3 p.83–84). This ensures that the honest chain grows faster than any attacker's private chain.

**Mathematical basis**: if the attacker controls fraction $q$ of hash power and honest miners control $p = 1-q > 0.5$, the probability that an attacker who starts $k$ blocks behind can ever catch up decreases exponentially with $k$. Transactions with enough confirmations ($k$ large enough) are secure with overwhelming probability.

**What happens if assumption fails — 51% attack**:

If an attacker controls >50% of hash power:
1. Attacker mines a private chain, starting from block $N$
2. On the public chain, the victim's transaction is included in block $N+1$, $N+2$, ..., $N+k$ (confirmed)
3. The victim delivers goods or services, considering the transaction final
4. The attacker continues mining their private chain. Since they have >50% hash power, their private chain grows faster than the public chain
5. The attacker releases their private chain (which does not include the victim's transaction, or includes a conflicting transaction sending the same coins to the attacker)
6. All nodes switch to the attacker's chain (it is now the longest)
7. The victim's transaction is reversed — the goods/services are gone; the coins are back in the attacker's private chain (**double spend**)

The attacker can continue this indefinitely as long as they maintain majority hash power, reversing transactions at will.

**Additional consequences of a successful 51% attack**:
- The attacker can exclude specific transactions from the chain (censorship)
- The attacker can prevent any other miner from successfully mining (selfish mining, monopolisation)
- The attacker cannot steal coins from addresses whose private keys they do not hold, nor create coins from thin air (the proof-of-work rules still constrain valid blocks; the attacker can only control the chain ordering)

**Assumption 2 — Network connectivity and propagation**:

The longest-chain rule also assumes that all honest nodes receive the longest chain promptly. If the network is partitioned, different partitions may build on different chains and accumulate significant work before reconnecting. Upon reconnection, all work on the shorter partition's chain is discarded (including the transactions in those blocks). This is not typically exploitable by external attackers but can cause disruption during network partitions.

**Assumption 3 — Chain length = most work**:

In Bitcoin, longest chain means most cumulative proof-of-work (most blocks, since difficulty is adjusted globally). In networks where difficulty adjusts per block (some altcoins), chain length can be gamed — adding many low-difficulty blocks. The correct metric is cumulative proof-of-work, not block count.

---

### Summary

| Aspect | Description |
|---|---|
| Longest-chain rule | Adopt the chain representing the most cumulative proof-of-work |
| Accidental fork cause | Simultaneous block discovery; network propagation delay |
| Accidental fork resolution | Next block makes one chain longer; shorter chain orphaned |
| Security assumption | Honest nodes hold >50% of hash power |
| 51% attack consequence | Attacker can reverse confirmed transactions (double spend), censor transactions |
| Finality with $k$ confirmations | Probability of reversal decreases exponentially with $k$ |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.83–84: blockchain structure, Merkle trees, hash chaining, proof-of-work consensus)

_Status: Complete_  
_Done by: William_
