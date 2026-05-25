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

**The core insight**: without knowing $n$, a verifier cannot tell whether a root hash came from a deep tree over many small blocks, or a shallow tree over fewer larger blocks. The attacker exploits this by presenting a *different* message that happens to produce the *same* root hash.

---

**Step 1 — Build the original tree for $T = T_0 \| T_1 \| T_2 \| T_3$ at depth $n = 2$**:

```
Level 0: h_{0,0} = H(T_0),  h_{0,1} = H(T_1),  h_{0,2} = H(T_2),  h_{0,3} = H(T_3)
Level 1: h_{1,0} = H(h_{0,0} || h_{0,1}),  h_{1,1} = H(h_{0,2} || h_{0,3})
Level 2: h_{2,0} = H(h_{1,0} || h_{1,1})   ← this is the root, H_MT(T)
```

The root is $h_{2,0}$. Notice its definition: it is $H$ applied to the two level-1 nodes concatenated together — $H(h_{1,0} \| h_{1,1})$.

---

**Step 2 — What does a depth-0 Merkle tree look like?**

A depth-0 tree has $2^0 = 1$ block. The definition says leaves are hashed as $h_{0,i} = H(T_i)$, and the root is $h_{0,0}$. With only one block there are no internal nodes at all — the root is simply:
$$H_{MT}(T') = H(T_0')$$
That is: hash the one block, done.

---

**Step 3 — Construct the forged message $T'$**

Choose $T'$ to be a single block whose *content* is the two level-1 nodes from the original tree pasted together:
$$T_0' = h_{1,0} \| h_{1,1}$$

This is just a string of bits — specifically the concatenation of two hash outputs that the attacker can read off from the original tree (they are not secret).

---

**Step 4 — Compute $H_{MT}(T')$ and watch why it equals $H_{MT}(T)$**

Since $T'$ is a depth-0 tree with one block $T_0'$:
$$H_{MT}(T') = H(T_0')$$

Substitute what $T_0'$ actually is (the choice we made in Step 3):
$$= H(h_{1,0} \| h_{1,1})$$

But look at the original tree in Step 1 — $h_{2,0}$ was defined as exactly this:
$$= h_{2,0}$$

And $h_{2,0}$ is the root of the original tree, i.e. $H_{MT}(T)$:
$$= H_{MT}(T) \checkmark$$

There is no cryptographic trick here. The equalities follow purely from definitions:
- The first equality is the definition of how a depth-0 tree works
- The second equality is just substituting the choice we made for $T_0'$
- The third equality is the definition of $h_{2,0}$ from the original tree

The root of the depth-2 tree is $H(\text{two level-1 nodes})$. A depth-0 tree over one block $B$ computes $H(B)$. Both are the same hash call — so choosing $B = h_{1,0} \| h_{1,1}$ makes them identical.

---

**$T' \neq T$**: $T'$ is a single block of two concatenated hash values ($\approx 512$ bits for SHA-256); $T$ consists of four arbitrary data blocks. Completely different messages, same root hash.

**Why depth must be known**: if the verifier knows $n = 2$, they immediately reject $T'$ (it has one block, not $2^2 = 4$). Without $n$, the root hash alone is identical — there is no way to tell which message was originally signed.

---

**Extra — How to Fix It**:

Use **domain separation** — hash leaves and internal nodes differently by prepending a tag:
$$h_{0,i} = H(\texttt{0x00} \| T_i) \quad \text{(leaf node)}$$
$$h_{j+1,i} = H(\texttt{0x01} \| h_{j,2i} \| h_{j,2i+1}) \quad \text{(internal node)}$$

The attack fails because the forged $T'$ would be processed as a leaf: $H(\texttt{0x00} \| h_{1,0} \| h_{1,1})$. But the original root was computed as an internal node: $H(\texttt{0x01} \| h_{1,0} \| h_{1,1})$. Different tag → different input → different hash output. An internal node value can never produce the same hash as a leaf containing the same bytes.

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
