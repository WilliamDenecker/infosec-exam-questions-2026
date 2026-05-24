# Question 48

Quantum algorithms affect symmetric and asymmetric cryptography in fundamentally different ways.

**Explain the impact of Grover's algorithm and Shor's algorithm on:**

- **symmetric encryption,**
- **hash functions,**
- **digital signatures,**
- **key exchange mechanisms.**

**For each class, discuss:**

- **how security strength is affected,**
- **whether increasing key sizes is sufficient,**
- **which types of algorithms must be replaced entirely.**

## Answer

### Overview: Two Quantum Algorithms (ch2.2 PQCrypto p.11–12)

**Grover's algorithm** (ch2.2 PQCrypto p.11): a quantum algorithm for unstructured search. Searches an unsorted set of $n$ elements in $O(\sqrt{n})$ time instead of $O(n)$ classically. Applied to cryptography, this halves the effective security of symmetric primitives (search in key space or hash preimage space).

**Shor's algorithm** (ch2.2 PQCrypto p.12): a quantum algorithm for integer factorisation and discrete logarithm computation in polynomial time. This is catastrophic for all public-key algorithms based on these hard problems.

---

### Symmetric Encryption (ch2.2 PQCrypto p.11, p.16)

**Impact of Grover's algorithm**:
- AES-128: the key space is $2^{128}$; Grover's searches it in $O(2^{64})$ → reduces to effectively **64-bit security** → obsolete against a quantum adversary
- AES-256: key space $2^{256}$; Grover's reduces to $O(2^{128})$ → **128-bit security** → still acceptable (ch2.2 PQCrypto p.16)

**Impact of Shor's algorithm**: none — Shor's applies only to structured algebraic problems (factorisation, discrete logarithm). Symmetric encryption has no such structure.

**Is increasing key size sufficient?** Yes for symmetric encryption:
- Upgrade from AES-128 to AES-256 doubles the Grover-search cost from $2^{64}$ to $2^{128}$
- AES-256 remains secure in the post-quantum world — no algorithm replacement needed

**Conclusion**: symmetric encryption is **weakened but not broken**. Increasing key length to 256 bits is sufficient.

---

### Hash Functions (ch2.2 PQCrypto p.11, p.17)

**Impact of Grover's algorithm**:
- Preimage resistance: finding a preimage for an $n$-bit hash requires $O(2^n)$ classically; Grover's reduces to $O(2^{n/2})$
- SHA2-256 / SHA3-256: only **128-bit preimage resistance** post-quantum (ch2.2 PQCrypto p.17)
- SHA2-512 / SHA3-512: **256-bit preimage resistance** post-quantum — sufficient
- Collision resistance: Grover's reduces collision search from $O(2^{n/2})$ (birthday attack) to $O(2^{n/3})$ — a modest additional weakening

**Impact of Shor's algorithm**: none — hash functions have no algebraic structure Shor's can exploit.

**Is increasing output size sufficient?** Yes:
- Use SHA2-384 or SHA2-512 / SHA3-384 or SHA3-512 to maintain 192–256 bit preimage resistance post-quantum
- No algorithm replacement required

**Conclusion**: hash functions are **weakened but not broken**. Not the most critical element (ch2.2 PQCrypto p.17); using 384-bit or 512-bit hash outputs provides post-quantum adequate security.

---

### Digital Signatures (ch2.2 PQCrypto p.12, p.18)

**Impact of Shor's algorithm** (ch2.2 PQCrypto p.12, p.18):
- RSA signatures: rely on integer factorisation → broken in polynomial time by Shor's
- DSA: relies on discrete logarithm in $\mathbb{Z}_p^*$ → broken
- ECDSA: relies on elliptic curve discrete logarithm → broken

**All currently deployed digital signature algorithms are obsolete** against a large-scale quantum computer (ch2.2 PQCrypto p.18).

**Impact of Grover's algorithm**: negligible — does not attack the algebraic structure of signatures.

**Is increasing key size sufficient?** No:
- RSA-16384 is just as broken as RSA-2048 by Shor's algorithm — Shor's is polynomial and increasing modulus size doesn't help
- Only a fundamentally different mathematical problem can survive quantum attack

**What must be replaced**: all RSA, DSA, ECDSA-based signature schemes must be replaced with **post-quantum signature algorithms**:
- CRYSTALS-Dilithium (ML-DSA) — lattice-based (ch2.2 PQCrypto p.29)
- SPHINCS+ (SLH-DSA) — hash-based (ch2.2 PQCrypto p.29)
- Falcon — lattice-based, to be standardised (ch2.2 PQCrypto p.29)

---

### Key Exchange Mechanisms (ch2.2 PQCrypto p.12, p.19)

**Impact of Shor's algorithm** (ch2.2 PQCrypto p.12, p.19):
- RSA key encapsulation / RSA encryption: relies on factorisation → broken
- Diffie-Hellman (DH, DHE): relies on discrete logarithm in $\mathbb{Z}_p^*$ → broken
- Elliptic Curve DH (ECDH, ECDHE): relies on ECDLP → broken

**All currently deployed key exchange algorithms are obsolete** (ch2.2 PQCrypto p.19).

**Impact of Grover's algorithm**: none on key exchange — Grover's attacks symmetric key search, not DLP/factorisation.

**Is increasing key size sufficient?** No:
- DH with a 100,000-bit prime is still broken by Shor's in polynomial time — no key-size fix exists

**What must be replaced**: all DH, ECDH, RSA-based key exchange must be replaced with **post-quantum key encapsulation mechanisms (KEMs)**:
- CRYSTALS-Kyber (ML-KEM-512/768/1024) — lattice-based, NIST standardised (ch2.2 PQCrypto p.29)
- BIKE, Classic-McEliece, HQC — code-based, in 4th NIST round (ch2.2 PQCrypto p.30)

---

### Summary Table

| Primitive | Grover's impact | Shor's impact | Increasing size sufficient? | Must replace? |
|---|---|---|---|---|
| AES-128 | 64-bit security → obsolete | None | Yes (→ AES-256) | No |
| AES-256 | 128-bit security | None | Already sufficient | No |
| SHA2/3-256 | 128-bit preimage | None | Upgrade to 512-bit | No |
| SHA2/3-512 | 256-bit preimage | None | Already sufficient | No |
| RSA / DSA / ECDSA (signatures) | Negligible | Polynomial break | **No** | **Yes** |
| DH / ECDH / RSA-KEX (key exchange) | None | Polynomial break | **No** | **Yes** |

---

### Post-Quantum Note (ch2.2 PQCrypto p.43)

Post-quantum algorithms are **not yet as thoroughly tested** as classical cryptography. "Quantum safety is conjectured" — surprises remain possible (one NIST candidate was withdrawn after a successful attack was published, ch2.2 PQCrypto p.30). Deployment should use hybrid constructions combining classical and post-quantum algorithms wherever feasible.

### Sources

- IS_UG_2_2_SecM-adv-PQCrypto (p.11: Grover's algorithm — complexity O(√n), impact on symmetric/hash; p.12: Shor's algorithm — polynomial factorisation and DLP, impact on RSA/DH/ECDH/ECDSA/DSA; p.13: Shor's implementation challenges; p.16: AES-128 vs AES-256 post-quantum; p.17: SHA2/3 post-quantum; p.18: signature algorithms post-quantum — all obsolete; p.19: key exchange post-quantum — all obsolete; p.20: requirements post-quantum; p.29: NIST selected algorithms; p.30: 4th round submissions; p.43: remaining issues — quantum safety conjectured)

_Status: Complete_  
_Done by: William_
