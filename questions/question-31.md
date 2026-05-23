# Question 31

A *ring signature* is a special type of digital signature. The signature can be performed by any member of a group of users. Anyone (given the ring signature and the public keys of all group members) can verify the authenticity of the signature, but not determine *which* member of the group has created the signature.

We consider a group with 4 members (this can be easily generalised to any number of members) using RSA key pairs $(KU_i, KR_i)$ ($1 \le i \le 4$) and assume the signer is user number 3.

The procedure uses a keyed function $C_{k,v}(y_1, y_2, y_3, y_4)$ (depending on a key value $k$ and a glue value $v$). The tuple $y_1, y_2, y_3, y_4$ will be chosen such that the ring equation holds:

$$C_{k,v}(y_1, y_2, y_3, y_4) = v \quad (Q31.6)$$

The practical implementation of the keyed function $C_{k,v}(y_1, y_2, y_3, y_4)$ is:

$$E_k(y_4 \oplus E_k(y_3 \oplus E_k(y_2 \oplus E_k(y_1 \oplus v)))) \quad (Q31.7)$$

where $E_k$ is the symmetric encryption (consider AES-256) using key value $k$ (the corresponding decryption could be written as $D_k$), and where $\oplus$ is the bitwise XOR operation.

The signature is generated (by user 3) as follows:

1. the key value $k$ is computed from the message using a (given and known) cryptographic hash function $H$ (e.g. SHA2-256): $k = H(M)$ (truncation to the required key length is allowed)
2. a random glue value $v$ is chosen
3. random values $x_i$ ($1 \le i \le 4 \land i \neq 3$) are chosen
4. the corresponding values $y_i = E_{KU_i}(x_i)$ are used (with $E_{KU_i}$ the RSA encryption with the public key $KU_i$) ($1 \le i \le 4 \land i \neq 3$; no PKCS #1 formatting is used)
5. the ring equation (Q4.1) is solved for $y_3$
6. the value of $x_3 = E_{KR_3}(y_3)$ is determined
7. the ring signature for message $M$ is the 9-tuple $(KU_1, KU_2, KU_3, KU_4, v, x_1, x_2, x_3, x_4)$

The verification of the ring signature is as follows:

1. compute the values $y_i = E_{KU_i}(x_i)$
2. calculate the key value $k = H(M)$
3. verify that the ring equation (Q4.1) holds

**How can the ring equation (Q4.1) be solved for $y_3$? Explain why someone outside the group (without knowledge of any of the private keys $KR_i$) can't generate a ring signature for the group. Explain why it isn't possible to identify which member of the group has generated the ring signature.**

## Answer

### How to Solve the Ring Equation for $y_3$ (ch2.2.3 p.77–85)

The ring equation is:
$$C_{k,v}(y_1, y_2, y_3, y_4) = v$$

Expanding via equation (Q31.7):
$$E_k(y_4 \oplus E_k(y_3 \oplus E_k(y_2 \oplus E_k(y_1 \oplus v)))) = v$$

**Working backwards through the chain**:

The signer (user 3) knows: $v$ (chosen randomly in step 2), $k = H(M)$, $y_1, y_2, y_4$ (computed in step 4 from the random $x_1, x_2, x_4$ and the public keys $KU_1, KU_2, KU_4$).

The signer needs to find $y_3$ such that the ring equation holds. Since $E_k$ (AES-256) is a permutation with a known inverse $D_k$, we can peel off the layers:

**Step 1**: Apply $D_k$ to both sides:
$$y_4 \oplus E_k(y_3 \oplus E_k(y_2 \oplus E_k(y_1 \oplus v))) = D_k(v)$$
$$\Rightarrow E_k(y_3 \oplus E_k(y_2 \oplus E_k(y_1 \oplus v))) = D_k(v) \oplus y_4$$

**Step 2**: Apply $D_k$ again:
$$y_3 \oplus E_k(y_2 \oplus E_k(y_1 \oplus v)) = D_k(D_k(v) \oplus y_4)$$

**Step 3**: The inner part $E_k(y_2 \oplus E_k(y_1 \oplus v))$ is fully computable (all values known: $k$, $v$, $y_1$, $y_2$). Compute:
$$z_2 = E_k(y_2 \oplus E_k(y_1 \oplus v))$$

**Step 4**: Solve for $y_3$:
$$y_3 = D_k(D_k(v) \oplus y_4) \oplus z_2$$

This is fully computable using only $k$ (known from hash), $v$ (chosen by signer), $y_1, y_2, y_4$ (computed from random $x_1, x_2, x_4$ and public keys), and $D_k$ (symmetric decryption — available to anyone who knows $k$).

**In summary**: the signer computes a partial forward evaluation of the chain up to the position before $y_3$, and uses a partial backward evaluation from the output $v$ to determine what $y_3$ must be. The ring equation can be solved for any one position if the values at all other positions are known — the key insight is that the chain is built from invertible operations.

After finding $y_3$, the signer can compute $x_3 = E_{KR_3}(y_3)$ — RSA encryption with the **private key** $KR_3$ (which is the RSA "signature" operation in raw RSA: $x = m^d \bmod n$).

---

### Why an Outsider Cannot Generate a Ring Signature

An outsider who does not know any $KR_i$ attempts to forge a ring signature:

1. They choose random $x_1, x_2, x_3, x_4$ freely.
2. They compute $y_i = E_{KU_i}(x_i) = x_i^e \bmod n_i$ for all $i$.
3. They check whether the ring equation $C_{k,v}(y_1, y_2, y_3, y_4) = v$ holds.

For the ring equation to hold with a random chosen $v$, the $y_i$ values must be related by the chain equation. With four random $x_i$ values (and thus four random $y_i$ values), the ring equation will almost certainly not hold — the probability is $\approx 1/2^{128}$ (random output of AES matching the fixed target $v$).

**The structural constraint**: to make the ring equation hold, the signer must be able to "close the ring" — adjust one position in the chain to make the output equal $v$. The calculation above shows how to find the required $y_3$. But then the signer needs $x_3$ such that $y_3 = E_{KU_3}(x_3) = x_3^e \bmod n_3$.

Finding $x_3$ from $y_3$ requires computing $x_3 = y_3^d \bmod n_3$ — the RSA private key operation, which requires knowledge of $KR_3 = d_3$. Without $KR_3$, this is the RSA inversion problem (equivalent to factoring $n_3$) — computationally infeasible (ch2.2.3 p.77–80).

**Key insight**: the ring equation can be solved for any $y_i$ using only symmetric operations (which anyone can perform). But converting the required $y_3$ back to a valid $x_3 = E_{KU_3}^{-1}(y_3)$ requires the private key $KR_3$. An outsider can find what $y_3$ must be, but cannot find the $x_3$ that produces it without the private key.

This is the fundamental security of the ring signature: the signer's contribution ($x_3, y_3$) is valid only because they can invert their own RSA encryption using their private key. Any member of the group can do this; an outsider cannot.

---

### Why It Is Impossible to Identify the Signer

The ring signature is the 9-tuple $(KU_1, KU_2, KU_3, KU_4, v, x_1, x_2, x_3, x_4)$.

A verifier computes $y_i = E_{KU_i}(x_i)$ for all $i = 1, 2, 3, 4$ and checks the ring equation. All four $y_i$ values appear symmetrically in the ring equation — no $y_i$ is distinguished from the others by its role in the verification.

**Why all positions look identical to an outsider**:

The relationship between $x_i$ and $y_i$ is $y_i = x_i^e \bmod n_i$ (RSA public key encryption). This is a publicly computable, deterministic operation. For positions $i = 1, 2, 4$: the signer chose $x_i$ randomly and then computed $y_i$ via the public key. For position $i = 3$: the signer computed $y_3$ to satisfy the ring equation, then computed $x_3 = y_3^{d_3} \bmod n_3$ using the private key.

An external verifier sees $(x_i, y_i)$ pairs satisfying $y_i = x_i^{e_i} \bmod n_i$ for all $i$. This relation holds for all four positions — it is the RSA public key relation, true by definition for any $x_i$ (whether chosen randomly as in positions 1,2,4 or computed using the private key as in position 3). **The verifier cannot distinguish the position where the private key was used.**

**Random glue value $v$** further prevents identification: $v$ is chosen randomly by the signer. Different signers (using different private keys) would choose different $v$ values and different random $x_i$ values for the non-signing positions. The resulting signature 9-tuples are computationally indistinguishable from signatures by any other group member — all valid ring signatures have the same statistical distribution from an observer's perspective.

**Information-theoretic argument**: the signer's private key $KR_3$ is used only to compute $x_3 = y_3^{d_3} \bmod n_3$ — a value that is publicly verifiable by checking $E_{KU_3}(x_3) = y_3$. But this verification tells an observer only that $x_3$ is the RSA preimage of $y_3$ under $KU_3$ — which is true for any member who could have signed (each member's private key would produce a preimage of the appropriate $y_i$). The signer's identity is hidden in the freedom to choose $v$ and the non-signing $x_i$ values arbitrarily.

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.77–85: RSA algorithm, raw RSA operations, public key encryption and private key decryption; p.45–52: chaining via XOR and encryption — analogous to CBC-MAC chain structure)

_Status: Complete_  
_Done by: William_
