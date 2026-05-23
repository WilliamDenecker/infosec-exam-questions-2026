# Question 38

A Merkle tree (aka hash tree) is used in the bitcoin Blockchain technology, but could also be seen as a normal hash function $H_{MT}$. In short, it consists of a tree of hashes in which the leaves are hashes of data blocks. This is illustrated for a (binary) Merkle tree of depth 2 in Fig. 4.

A more rigorous description for a binary Merkle tree of depth $n$ is:

- The data $T$ are divided in $2^n$ blocks $T_i$ of equal size (padding may be needed, but we shan't consider it here to keep things simple) (with $i \in 0..(2^n - 1)$), so that $T$ is the concatenation of all blocks $T_i$: $T = T_0\|T_1\|\ldots\|T_{2^n-1}$
- Each block $T_i$ is then hashed using a hash function $H$: $h_{0,i} = H(T_i)$ (with $i \in 0..(2^n - 1)$)
- The intermediate hash values are then combined and hashed to obtain the parent hash nodes: $h_{j+1,i} = H(h_{j,2 \cdot i}\|h_{j,2 \cdot i+1})$ (with $j \in 0..(n - 1)$ and $i \in 0..(2^{n-j-1} - 1)$)
- The final hash value is then $H_{MT}(T) = h_{n,0}$

The hash function $H$ is a traditional cryptographic hash function (MD5, SHA1, SHA2-256, etc.).

- **Compare the performance of the computation of $H_{MT}$ to that of a regular hash function $H$ (consider parallellisability).**
- **If the depth $n$ of the Merkle tree isn't given, this scheme doesn't exhibit weak collision resistance (aka second preimage resistance). Find some data $T'$ such that $T' \neq T$ and $H_{MT}(T') = H_{MT}(T)$.**

  *Extra: Can you adapt the basic scheme of the computation of the Merkle tree to avoid this issue (beyond using a fixed depth $n$)?*

- **For a given depth $n$, does $H_{MT}$ exhibit (strong) collision resistance if MD5 is chosen as a hash function $H$? Explain your answer.**

## Answer

### Part 1 — Performance: Merkle Tree vs Regular Hash Function (ch2.2.3 p.83–84)

**Regular hash function $H$ applied to all of $T$**:
A standard Merkle-Damgård hash function (MD5, SHA-1, SHA-256) processes the message block by block sequentially (ch2.2.3 p.30–34). For a message of $2^n$ blocks, the hash requires $2^n$ sequential applications of the compression function — **fully sequential, no parallelism possible** (each block's hash depends on the previous chained state).

**Merkle tree $H_{MT}$**:

The computation has $n + 1$ levels:
- Level 0 (leaf hashes): $2^n$ independent computations $h_{0,i} = H(T_i)$ — all independent, **fully parallelisable**
- Level 1: $2^{n-1}$ independent computations $h_{1,i} = H(h_{0,2i} \| h_{0,2i+1})$ — fully parallelisable (depend only on level 0, which is complete)
- Level $j$: $2^{n-j}$ independent computations — fully parallelisable within each level
- ...until the root $h_{n,0}$

**Total number of hash calls**: the Merkle tree requires:
$$\sum_{j=0}^{n} 2^{n-j} = 2^n + 2^{n-1} + \ldots + 1 = 2^{n+1} - 1$$

A direct hash of all blocks requires exactly $2^n$ compression function calls (one per block). So the Merkle tree requires approximately **twice as many hash calls** as a direct hash (roughly $2^{n+1}$ vs. $2^n$ calls), because it hashes each block and then hashes pairs of hashes recursively.

**Parallelism advantage**: with $P$ processors:
- Direct hash $H$: time = $2^n$ sequential hash calls → no speedup with more processors
- Merkle tree: all leaves computed in parallel (time = 1 hash call depth), then levels above proceed; total parallel depth = $n + 1$ levels. With $P = 2^n$ processors, the total time is $O(n)$ hash calls — **logarithmic in message size** vs. **linear** for direct hash.

**Summary**:

| Criterion | Direct $H(T)$ | Merkle tree $H_{MT}(T)$ |
|---|---|---|
| Total hash calls | $2^n$ | $2^{n+1} - 1 \approx 2 \times 2^n$ |
| Sequential depth | $2^n$ (fully sequential) | $n + 1$ (logarithmic) |
| Parallelisability | **None** (Merkle-Damgård chain) | **Full** (within each level) |
| Speedup with $P = 2^n$ processors | $1\times$ | $\approx 2^n / (n+1)$ |

The Merkle tree pays ~2× more total hash calls but achieves logarithmic parallel depth — valuable for large datasets and high-parallelism hardware.

---

### Part 2 — Second Preimage Attack When Depth $n$ Is Unknown (ch2.2.3 p.13, p.83–84)

**The vulnerability**: if the tree depth $n$ is not fixed or not included in the hash computation, the Merkle tree is vulnerable to a **tree-lifting attack** (also known as a second-preimage attack exploiting tree structure).

**Attack to find $T' \neq T$ with $H_{MT}(T') = H_{MT}(T)$**:

Consider the hash tree for $T = T_0 \| T_1 \| T_2 \| T_3$ at depth $n = 2$. The tree produces:
```
Level 0: h_{0,0} = H(T_0),  h_{0,1} = H(T_1),  h_{0,2} = H(T_2),  h_{0,3} = H(T_3)
Level 1: h_{1,0} = H(h_{0,0} || h_{0,1}),  h_{1,1} = H(h_{0,2} || h_{0,3})
Level 2: h_{2,0} = H(h_{1,0} || h_{1,1})  ← root
```

Now construct $T'$ as a **message of 2 blocks** at depth $n' = 1$:
$$T' = T_0' \| T_1' \quad \text{where} \quad T_0' = h_{1,0}, \quad T_1' = h_{1,1}$$

The Merkle tree of $T'$ at depth $n' = 1$:
- Level 0: $h_{0,0}' = H(T_0') = H(h_{1,0})$, $h_{0,1}' = H(T_1') = H(h_{1,1})$
- Wait — these are not the same as $h_{1,0}$ and $h_{1,1}$ (those are themselves hash values)

Let me use the simpler direct "lifting" attack:

Consider $T' = h_{1,0} \| h_{1,1}$ as a 2-block message (depth $n' = 1$):
- $h_{0,0}' = H(h_{1,0})$... this is not $h_{1,0}$

The correct attack is more direct: **set $T'$ to be a 2-block message where the two "blocks" are exactly the level-1 internal nodes**:

If we treat $h_{1,0}$ and $h_{1,1}$ as the direct leaf data blocks of a depth-1 Merkle tree:
- $T_0' = h_{1,0}$, $T_1' = h_{1,1}$ — these are the blocks of $T'$
- Level-0 leaf hashes: $H(h_{1,0})$ and $H(h_{1,1})$ — NOT the same as $h_{1,0}$ and $h_{1,1}$

This does not work directly. The simpler attack:

**Second preimage from tree node values**:
The depth-2 Merkle tree of $T$ includes internal nodes $h_{1,0}$ and $h_{1,1}$. If we define a depth-1 Merkle tree over $T' = T_0' \| T_1'$ where $T_0'$ is any block that hashes to $h_{1,0}$ and $T_1'$ is any block that hashes to $h_{1,1}$... this requires finding preimages.

**The actual simple attack**: define $T'$ as a **1-block message** at depth 0: $T' = h_{2,0}$ (the root hash value itself as a 1-block message). With depth $n' = 0$: $H_{MT}(T') = h_{0,0}' = H(h_{2,0}) = H(H_{MT}(T))$. This is not the same as $H_{MT}(T)$ unless $H_{MT}(T) = H(H_{MT}(T))$ — which is generally false.

**The correct tree-lifting attack** (standard result for Merkle trees without depth encoding):

Define $T'$ to be a message with $2^{n+1}$ blocks such that the first $2^n$ blocks of $T'$ are $h_{0,0}, h_{0,1}, \ldots, h_{0,2^n-1}$ (the leaf hash values of the original tree $T$), and the last $2^n$ blocks can be arbitrary. Wait — this doesn't work either.

**The true attack**: define $T'$ with **$2^{n-1}$ blocks** at depth $n' = n-1$, where the block $T_0' = h_{1,0}$ and $T_1' = h_{1,1}$ (for $n = 2$). But these are $|H|$-bit values (hash output length), while blocks are typically larger.

**Simplified correct attack for unknown $n$**:

Take $T' = h_{1,0} \| h_{1,1}$ as a 2-block message and compute $H_{MT}$ as if depth $= 1$:
- Actually, the tree-lifting attack sets $T_i' = h_{n-1, i}$ (the level-$n-1$ nodes of the original tree as the leaf "blocks" of a shallower tree)
- Leaf hashes at depth $n-1$ for $T'$: $H(T_i') = H(h_{n-1,i})$ — not equal to $h_{n-1,i}$ in general

The attack only works if we treat internal hash **values** directly as leaf **data** — i.e., if blocks and hash outputs have the same length (they do, for standard hash functions: SHA-256 produces 256-bit outputs matching typical block sizes — but "blocks" here are message blocks, not hash-output sized chunks).

In practice: for a Merkle tree without depth encoding, define $T' = (h_{0,0} \| h_{0,1} \| \ldots \| h_{0,2^n-1})$ — a message of $2^n$ blocks where block $i$ of $T'$ is the leaf hash of block $i$ of $T$. Then the Merkle tree of $T'$ at depth $n$ produces leaf hashes $H(h_{0,i})$ — which is level 1 of a depth-$(n+1)$ tree. This does not immediately give a second preimage.

**The standard result for the attack** (from Merkle tree security literature):
Take $T'$ as the concatenation of the $2^{n-1}$ level-1 nodes, treated as leaf blocks of a depth-$(n-1)$ tree:
$$T' = h_{1,0} \| h_{1,1} \| \ldots \| h_{1, 2^{n-1}-1}$$
Then $H_{MT}^{(n-1)}(T') = H(h_{1,0} \| h_{1,1}) $ ... wait, this uses depth $n-1$, which gives $2^{n-1}$ leaf blocks. The computation:
- Level 0 leaves: $H(h_{1,0})$, $H(h_{1,1})$, ..., $H(h_{1, 2^{n-1}-1})$ — NOT equal to $h_{1,i}$.

**This attack works if the internal node values ARE directly used as leaf data without re-hashing**. Such a Merkle variant exists but the standard recursive definition does hash leaves. The correct statement for the standard definition:

**The second preimage attack requires**: treat internal nodes at level 1 ($h_{1,0}, \ldots, h_{1, 2^{n-1}-1}$) as the raw leaf blocks of a depth-$(n-1)$ Merkle tree, where the leaf hashes are defined directly as $h_{1,i}$ without applying $H$ to them — i.e., bypass the leaf hashing step. This is the "length extension" style attack on Merkle trees: if the scheme doesn't distinguish leaf nodes from internal nodes, a subtree can be substituted.

**Simple valid attack**: For a Merkle tree where **leaves are hashed the same way as internal nodes**, and depth is unknown: let $T' = (h_{1,0}, h_{1,1})$ be a 2-block message (where each "block" is a hash-output-length string). Compute $H_{MT}^{(n'=1)}(T')$:
- Level 0: $h_{0,0}' = H(h_{1,0})$, $h_{0,1}' = H(h_{1,1})$ — NOT $h_{1,0}$ and $h_{1,1}$
This still doesn't give the same root unless $H$ is the identity, which it isn't.

**The attack only works if there is NO SEPARATION between leaf and internal node hashing**. The standard fix (described below) addresses this.

For the purposes of this answer: **the second preimage attack** on a Merkle tree without depth or node-type encoding exploits the ability to construct a shorter tree whose root equals an internal node of the original tree. Specifically, if we define $T'$ to consist of $2^{n-1}$ "blocks" where block $i$ is the **raw value** $h_{1,i}$ (treating level-1 nodes as data blocks, and computing leaves of $T'$ as $H(h_{1,i})$... no, same problem.

**Clean statement of the standard attack**: $T' = $ single-block message consisting of the concatenation of $h_{1,0}$ and $h_{1,1}$ at depth 0: $H_{MT}^{(0)}(T') = H(h_{1,0} \| h_{1,1}) = h_{2,0} = H_{MT}(T)$. **Yes! This works.**

- The original $T$ has $2^n = 4$ blocks at depth $n = 2$, root $h_{2,0} = H(h_{1,0} \| h_{1,1})$.
- Define $T'$ as a **single block** $T_0' = h_{1,0} \| h_{1,1}$ at depth $n' = 0$.
  - $H_{MT}^{(0)}(T') = h_{0,0}' = H(T_0') = H(h_{1,0} \| h_{1,1}) = h_{2,0} = H_{MT}(T)$. ✓

If depth is not given (so the receiver cannot tell whether the tree has depth 0 or depth 2), the sender can present $T' = (h_{1,0} \| h_{1,1})$ as a depth-0 (single-block) Merkle tree and claim it has the same hash as $T$. It does: $H_{MT}(T') = H_{MT}(T) = h_{2,0}$.

**$T' \neq T$** since $T'$ consists of one block equal to 256 bits (two concatenated SHA-256 hashes) while $T$ consists of four data blocks of arbitrary size.

**Extra — How to Fix It**:

Include the **depth** $n$ or the **number of leaf blocks** $2^n$ in the Merkle tree computation — either as a prefix hashed into the root, or by using **domain separation**: when hashing a leaf, prepend a leaf tag (e.g., $0x00$): $h_{0,i} = H(0x00 \| T_i)$; when hashing an internal node, prepend an internal tag (e.g., $0x01$): $h_{j+1,i} = H(0x01 \| h_{j,2i} \| h_{j,2i+1})$. This prevents a leaf value from being confused with an internal node value, defeating the attack.

---

### Part 3 — Collision Resistance with MD5 as $H$ (ch2.2.3 p.14)

**MD5** has practical collision attacks — two distinct messages $M \neq M'$ with $MD5(M) = MD5(M')$ can be found efficiently (ch2.2.3 p.14). A collision in $H$ (MD5) immediately implies a collision in $H_{MT}$:

If $MD5(T_i) = MD5(T_j')$ for some pair of blocks $T_i \neq T_j'$, then substituting $T_j'$ for $T_j$ in the leaf $T$ while keeping all other blocks the same gives a new message $T''$ with $H_{MT}(T'') = H_{MT}(T)$ (the leaf hash is the same, so all internal nodes and the root are unchanged).

**Conclusion**: **$H_{MT}$ does NOT exhibit collision resistance** when MD5 is used as the underlying $H$. Any collision attack against MD5 (practical with current techniques) directly produces a collision in the Merkle tree hash. The Merkle tree cannot compensate for a broken underlying hash function — it is only as collision-resistant as $H$.

Additionally, the tree structure gives the attacker more flexibility: collisions in any of the $2^n$ leaf positions suffice, and the structure of the tree (requiring collisions at specific positions) does not significantly increase the attacker's difficulty compared to finding a single MD5 collision.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.12–14: preimage resistance, second preimage resistance, collision resistance; p.14: MD5 collision attacks; p.30–34: Merkle-Damgård construction; p.83–84: Merkle trees in blockchain — structure and integrity properties)

_Status: Complete_  
_Done by: William_
