# Question 10

HMAC is the most commonly used message authentication code based on a hash function:

$$HMAC = H\left[(K^+ \oplus opad)\|H\left[(K^+ \oplus ipad)\|M\right]\right] \quad (Q10.1)$$

A possible simplification of HMAC might be:

$$SimpleHMAC = H\left[(K^+ \oplus ipad)\|M\right] \quad (Q10.2)$$

**Why is this SimpleHMAC (Q24.9) unsuitable as a message authentication code? Explain how, given a message $M$ and its SimpleHMAC, you could construct the SimpleHMAC of some related message $M'$ without knowing the key $K^+$.**

*Note: the hash function used in (Q24.9) may be either MD5, SHA-1, or SHA-2 (not SHA-3).*

## Answer

### Why SimpleHMAC Is Unsuitable (ch2.2.3 p.63–66)

SimpleHMAC = $H[(K^+ \oplus ipad) \| M]$ is vulnerable to the **length extension attack**, which exploits the internal structure of Merkle-Damgård hash functions.

---

### The Merkle-Damgård Construction (ch2.2.3 p.30–40)

MD5, SHA-1, and SHA-2 (SHA-256, SHA-512) are all built on the **Merkle-Damgård construction** (ch2.2.3 p.30–34). In this construction:

1. The input message is **padded** to a multiple of the block size (512 bits for MD5/SHA-1/SHA-256; 1024 bits for SHA-512). The padding consists of a `1` bit, followed by zeros, followed by a 64-bit (or 128-bit for SHA-512) encoding of the original message length. This padding is mandatory and deterministic — given a message of known length, the padding is uniquely determined.

2. The padded message is split into $t$ blocks $B_1, B_2, \ldots, B_t$.

3. The hash is computed iteratively: starting from a fixed initial value $IV$, apply the compression function $f$:
$$s_0 = IV, \quad s_i = f(s_{i-1}, B_i) \quad \text{for } i = 1, \ldots, t$$

4. The **final hash output** $H(M) = s_t$ is the internal state after processing all blocks.

**Critical property**: the hash output $H(M) = s_t$ is the internal state $s_t$ of the compression function. If an adversary knows $H(M)$, they know the exact internal state of the hash function after processing the padded message $M$. They can resume the computation from that state and process additional blocks.

---

### The Length Extension Attack on SimpleHMAC

**Setup**: the attacker knows:
- $M$: the original message
- $SimpleHMAC = H[(K^+ \oplus ipad) \| M]$: the MAC of $M$
- The length of $K^+$ (which is the block size of the hash function, typically 64 bytes for SHA-256 — this is not secret; $K^+$ is defined as the key padded to the block size)

**What the attacker does NOT know**: the actual key $K^+$.

**Attack to forge $SimpleHMAC$ of $M'$ without knowing $K^+$**:

Let $pad(x)$ denote the Merkle-Damgård padding applied to a string $x$. The hash function internally processes:
$$(K^+ \oplus ipad) \| pad(M)$$
and produces $SimpleHMAC = H[(K^+ \oplus ipad) \| M]$.

Now define:
$$M_{ext} = pad(M) \| \text{any attacker-chosen suffix } X$$

That is, the attacker appends the Merkle-Damgård padding that would have been applied to $M$, then appends arbitrary additional data $X$.

The attacker sets the internal state of the hash function to $SimpleHMAC$ (the known value) and continues hashing the suffix $X$:
$$SimpleHMAC' = ResumeHash(SimpleHMAC,\ X)$$

where $ResumeHash$ means: initialise the internal state to $SimpleHMAC$ instead of $IV$, and continue processing block(s) from $X$.

**Why this works**: $SimpleHMAC$ is exactly the internal state $s_t$ of the hash after processing $(K^+ \oplus ipad) \| pad(M)$. Resuming from $s_t$ and processing $X$ produces $H[(K^+ \oplus ipad) \| M_{ext} \| X]$, where the full input to the hash function is $(K^+ \oplus ipad) \| pad(M) \| X$.

Now define $M' = M_{ext} \| X$ = the message $M$ with its Merkle-Damgård padding appended, followed by the attacker-chosen data $X$.

The attacker has computed $SimpleHMAC'$ such that:
$$SimpleHMAC' = H[(K^+ \oplus ipad) \| M']$$

This is a valid SimpleHMAC for $M'$ — computed without knowing $K^+$.

**Concrete example with SHA-256** (block size = 512 bits = 64 bytes, output = 256 bits = 32 bytes):

Suppose $M$ = "Transfer $100 to Alice" (22 bytes). $K^+$ = 64-byte padded key.

The hash internally processes: $(K^+ \oplus ipad)$ (64 bytes) $\|$ $M$ (22 bytes) $\|$ $pad$ (42 bytes of padding + 8-byte length = 50 bytes) = 136 bytes = 2 SHA-256 blocks.

The attacker knows $SimpleHMAC$ (32 bytes = the SHA-256 output = the internal state after block 2).

The attacker sets state = $SimpleHMAC$ and processes a new block containing $X$ = "Transfer $1000 to Bob". They compute $SimpleHMAC'$ without knowing $K^+$.

The resulting $M'$ = "Transfer $100 to Alice" $\|$ (SHA-256 padding for 22-byte message) $\|$ "Transfer $1000 to Bob" — a message that looks garbled but has a valid MAC.

---

### Why Full HMAC Prevents This

Real HMAC = $H[(K^+ \oplus opad) \| H[(K^+ \oplus ipad) \| M]]$ (ch2.2.3 p.63–66).

The outer hash takes as input $(K^+ \oplus opad) \| inner$, where $inner = H[(K^+ \oplus ipad) \| M]$.

If an attacker applies the length extension to the inner hash, they produce $H[(K^+ \oplus ipad) \| M']$ — a new inner hash. But to produce a valid full HMAC for $M'$, they now need:
$$HMAC' = H[(K^+ \oplus opad) \| H[(K^+ \oplus ipad) \| M']]$$

The outer hash requires computing $H[(K^+ \oplus opad) \| \text{something}]$. The attacker knows the new inner hash value but does not know $(K^+ \oplus opad)$. They cannot produce the outer hash without this outer key material. The double-hash structure specifically breaks the length extension by placing the key material in the outer layer — an attacker who extends the inner hash cannot compute the outer hash without knowing $K^+$.

**This is the design rationale for the nested HMAC structure**: the inner key ($ipad$) protects against key-recovery; the outer key ($opad$) protects against length extension. Both layers use the same key $K^+$ with different padding constants ($ipad = 0x36...36$, $opad = 0x5C...5C$), ensuring the outer layer cannot be bypassed even when the inner hash output is known.

---

### Summary

| Property | SimpleHMAC $H[(K^+ \oplus ipad) \| M]$ | Full HMAC $H[(K^+ \oplus opad) \| H[(K^+ \oplus ipad) \| M]]$ |
|---|---|---|
| Length extension attack | **Vulnerable**: output is the raw MD internal state | **Immune**: output is hashed again with a different key material |
| Applies to | MD5, SHA-1, SHA-2 (Merkle-Damgård) | MD5, SHA-1, SHA-2 — all secure with HMAC wrapper |
| Forging without key | Possible for $M' = M \| pad \| X$ (any suffix) | Not possible — outer hash requires $K^+$ |

*Note: SHA-3 (Keccak sponge construction) is NOT vulnerable to length extension by design (ch2.2.3 p.41 — different internal structure), which is why the question specifically excludes SHA-3.*

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.30–40: Merkle-Damgård hash construction; p.63–66: HMAC design and security rationale)

_Status: Complete_  
_Done by: William_
