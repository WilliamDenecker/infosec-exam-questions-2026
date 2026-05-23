# Question 14

One of the fundamental challenges in digital currencies is the double-spending problem.

**Explain the double-spending problem in electronic currencies.**

**Why is this problem easy to solve with a trusted third party (TTP) but difficult in a fully decentralised system?**

**Explain how the blockchain structure and consensus mechanism address this problem without relying on a TTP.**

## Answer

### The Double-Spending Problem

In physical cash, a banknote is a physical object — it can only be in one place at a time. Once you hand a €20 note to a merchant, you no longer have it. Spending is physically exclusive.

Digital data is fundamentally different: copying is perfect, instantaneous, and free. A digital file representing currency value can be copied any number of times. If Alice holds a digital token worth €20 and sends it to Bob, she can simultaneously send an identical copy to Carol — the token itself is duplicated data. Both Bob and Carol receive what appears to be a valid €20 token, but Alice has spent the same €20 twice. This is **double-spending**: spending the same digital monetary unit more than once (ch3.1 p.6 — context of trusted third parties in financial systems).

Double-spending is not a cryptographic weakness — even perfectly signed, unforgeable tokens can be replayed by the legitimate owner. The problem is about **ordering and uniqueness of spending events**, not about token authenticity.

---

### Why TTP Solves the Problem Easily (ch3.1 p.6)

With a **Trusted Third Party** — a central bank or payment processor (ch3.1 p.6) — the solution is straightforward:

1. Alice holds a token representing €20, registered in the TTP's ledger as owned by Alice.
2. Alice sends a signed transaction to the TTP: "Transfer €20 to Bob."
3. The TTP checks its ledger: does Alice currently own €20? Yes. Is this token already spent? No.
4. The TTP marks Alice's token as spent and credits Bob.
5. If Alice simultaneously sends another transaction "Transfer €20 to Carol," the TTP checks again: Alice's token is now already marked spent. Transaction rejected.

**Why this is easy**: the TTP maintains a single authoritative, sequential ledger. It processes one transaction at a time, so there is no ambiguity about ordering. The TTP is the sole source of truth about token ownership. It can reject double-spends instantly and authoritatively.

This is exactly how VISA, PayPal, and traditional bank transfers work — they are all centralised ledgers.

**Why decentralised systems cannot use this approach directly**: a TTP introduces a single point of trust, control, and failure. The TTP can be compromised, can censor transactions, can fail operationally, or can collude with governments. The goal of cryptocurrencies is to eliminate this central authority — but this means there is no single entity that can maintain the authoritative ledger.

---

### Why Decentralisation Makes Double-Spending Hard

In a decentralised peer-to-peer network, transactions are broadcast to all participants simultaneously. The challenge:

1. **No single authority**: no single node can authoritatively declare "this transaction came first." Different nodes may receive transactions in different orders due to network latency.

2. **Sybil problem**: without a TTP, how do you prevent an attacker from creating thousands of fake identities (nodes) and flooding the network with fraudulent "this transaction came first" claims? A pure vote-by-number-of-nodes can be trivially manipulated.

3. **Simultaneous broadcast attack**: Alice creates two valid (signed) transactions spending the same coin — one to Bob and one to Carol — and broadcasts them simultaneously to different parts of the network. Without a TTP, which transaction should the network recognise?

4. **Network partition**: the network may temporarily split into segments that cannot communicate. Each segment might accept a different transaction. When the partition heals, which transaction is valid?

The core difficulty is achieving **global consensus on transaction ordering** among mutually distrusting peers with no central coordinator.

---

### How Blockchain Addresses Double-Spending (ch2.2.3 p.83–84)

Bitcoin's blockchain solves double-spending through a combination of three mechanisms:

**Mechanism 1 — A shared, append-only transaction ledger**

All confirmed transactions are recorded in a public blockchain — a chain of blocks where each block contains a batch of transactions and a cryptographic hash of the previous block (ch2.2.3 p.83–84). Each transaction references its unspent inputs (UTXO — Unspent Transaction Output). When a coin is spent, its UTXO is consumed and new UTXOs are created for the recipients.

A double-spend attempt creates two transactions consuming the same UTXO. Only one can be valid — the other attempts to spend a UTXO that has already been consumed.

**Mechanism 2 — Proof-of-work consensus**

Nodes (miners) compete to add the next block by solving a computational puzzle (proof-of-work — finding a nonce such that $SHA256(SHA256(block\_header)) < target$). This requires enormous computational effort.

**Key property**: adding a block costs real computational work. To rewrite history (change a past transaction), an attacker must redo all the proof-of-work for the modified block and all subsequent blocks — faster than the honest network produces new blocks.

**Mechanism 3 — Longest-chain rule and confirmations** (see Question 7)

All nodes adopt the longest chain as canonical. If Alice broadcasts both "pay Bob" and "pay Carol" simultaneously:
- Miners include one transaction in a block (they see both but can only include one per UTXO)
- The block is broadcast; the network adds it to the chain
- The transaction in the block is confirmed; the competing transaction is rejected (the UTXO is already spent)

If Alice mines a private chain to reverse the "pay Bob" transaction after Bob delivers goods:
- Alice would need to build a private chain (containing "pay Carol" instead) faster than the honest network builds on the "pay Bob" chain
- This requires Alice to control >50% of the network's hash power (51% attack)
- With <50% hash power, the probability that Alice can outpace the honest network decreases exponentially with each additional confirmation block (ch2.2.3 p.84)

**Confirmation protocol**: merchants wait for $k$ confirmation blocks (typically 6 in Bitcoin) before considering a transaction final. After 6 confirmations, the cost of reversing the transaction (requiring >50% hash power sustained over all 6 blocks) is considered prohibitively expensive for any rational attacker compared to the potential gain from double-spending.

**Why this eliminates the need for a TTP**:

| Role of TTP | Blockchain equivalent |
|---|---|
| Maintain authoritative ledger | Public blockchain replicated across all nodes |
| Order transactions | Block inclusion order; proof-of-work determines valid chain |
| Reject double-spends | UTXO model: spent UTXOs cannot be consumed again |
| Detect fraud attempts | Longest-chain rule: fraudulent forks lose unless >50% hash power |
| Trust in single authority | Distributed trust: no single node controls the ledger |

**Remaining limitation**: the system is only secure if honest miners hold >50% of hash power (see Question 7). If this assumption fails (51% attack), double-spending becomes possible. The blockchain does not eliminate double-spending risk — it makes it **computationally expensive** to the point of being economically irrational for most attack scenarios.

### Sources

- IS_UG_3_1_Appl_Basics (p.6: trusted third party in payment systems)
- IS_UG_2_2_3_SecM_HashMac (p.83–84: blockchain structure, hash chaining, proof-of-work, UTXO model, consensus)

_Status: Complete_  
_Done by: William_
