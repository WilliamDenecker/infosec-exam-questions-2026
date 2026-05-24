# Question 46

**Explain why using the same key for CBC-encryption and for the computation of the CBC-MAC (on the same plaintext) is not secure.**

## Answer

### Setup (ch2.2.3 p.61–62, p.67–68)

**CBC-MAC** (ch2.2.3 p.61–62): a message authentication code computed by encrypting the message in CBC mode with IV = 0 and using the last ciphertext block as the MAC tag:
$$T_i = E_K(P_i \oplus T_{i-1}), \quad T_0 = 0$$
$$\text{MAC} = T_N$$

**CBC encryption** (ch2.2.3 p.45–52): encrypts a message in CBC mode with a (typically random) initialisation vector IV:
$$C_i = E_K(P_i \oplus C_{i-1}), \quad C_0 = \text{IV}$$

Both use the same block cipher $E_K$ with the same key $K$.

---

### Why Using the Same Key Is Insecure

#### Attack 1 — CBC Encryption with IV = 0 Directly Reveals the MAC (ch2.2.3 p.62)

When CBC encryption is performed with IV = 0 (which is what CBC-MAC always uses), the encryption and MAC computation are **identical operations**:
$$C_i = E_K(P_i \oplus C_{i-1}), \quad C_0 = 0 = T_0$$

Therefore $C_i = T_i$ for all $i$, and in particular $C_N = T_N = \text{MAC}$.

**Consequence**: the last ciphertext block IS the MAC tag. An attacker who receives the ciphertext $C_1 \| \cdots \| C_N$ can immediately read off the MAC without knowing $K$. The MAC tag provides no authentication — it is fully public.

#### Attack 2 — MAC Forgery via the Encryption Oracle (ch2.2.3 p.61–62)

Even when CBC encryption uses IV ≠ 0, the same key $K$ allows the attacker to use the encryption function as an oracle to forge MACs.

**Concrete attack** (MAC extension forgery):

Given a valid pair $(M, T)$ where $M = P_1 \| P_2 \| \cdots \| P_N$ and $T = \text{CBCMAC}_K(M)$:

The attacker wants a valid MAC for the extended message $M' = M \| P_{N+1}$:
$$\text{CBCMAC}_K(M') = E_K(P_{N+1} \oplus T)$$

To compute $E_K(P_{N+1} \oplus T)$ without knowing $K$:
1. The attacker sets plaintext $P' = P_{N+1} \oplus T$
2. They request CBC encryption of the **single-block message** $P'$ with IV = 0 using the CBC encryption oracle (same key $K$):
   $$C' = E_K(P' \oplus 0) = E_K(P_{N+1} \oplus T)$$
3. This equals $\text{CBCMAC}_K(M')$ — the attacker has forged the MAC for $M'$

**Why this works**: with the same key $K$, a single-block CBC encryption with IV = 0 computes exactly one application of $E_K$ — which is precisely what the MAC extension requires. Without the same key, the encryption oracle ($E_{K_E}$) operates with a different key than the MAC ($E_{K_M}$), and the attacker cannot extract MAC-key evaluations from encryption oracle calls.

#### Why CCM Can Use the Same Key Safely (ch2.2.3 p.67–68)

CCM mode also uses the same key $K$ for both CBC-MAC (authentication) and CTR mode (encryption). This is secure because:
- **CTR mode** does not compute $E_K(P_i \oplus C_{i-1})$ — it computes $E_K(\text{counter}_j)$, which is independent of the plaintext content
- An attacker cannot use a CTR encryption oracle to compute $E_K(P_{N+1} \oplus T)$ because CTR mode inputs are counter values, not attacker-controlled data
- The CCM attacker has no way to query $E_K$ at the specific input $P_{N+1} \oplus T$ needed to forge the MAC

For **CBC encryption + CBC-MAC**, both operations share the same input structure: $E_K(\text{plaintext block} \oplus \text{previous block})$. The attacker can craft an encryption request whose plaintext XOR chaining equals exactly the value needed to evaluate $E_K$ at the MAC's required input.

---

### Root Cause

The root cause is **key reuse across two different but structurally related operations**. Both CBC encryption (with IV = 0) and CBC-MAC compute $E_K$ on plaintext-XOR-previous-block inputs. The attacker exploits this shared structure to use one oracle (CBC encryption) to evaluate the function needed to forge the output of the other (CBC-MAC).

The standard fix (ch2.2.3 p.67–68) is to:
1. Use **different keys** for encryption and MAC ($K_E \neq K_M$), or
2. Use an authenticated encryption mode (CCM, GCM) that was designed with key-reuse safety as an explicit security requirement

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.45–52: CBC mode, IV, block cipher chaining; p.61–62: CBC-MAC construction with IV=0; p.63–66: MAC requirements and keyed authentication; p.67–68: CCM mode — same key for CBC-MAC and CTR, why it is secure; p.70–75: GCM — secure authenticated encryption with single key)

_Status: Complete_  
_Done by: William_
