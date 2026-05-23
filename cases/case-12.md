# Case 12

The most recent version of the Belgian eID (electronic identity card) uses an RSA key with a 2048 bit modulus. This is probably still secure for more than a few years.

**Which approach would you suggest if a digital signature with a very long validity is required (e.g. 30 years for a mortgage, possibly longer for a marriage)? You need to *guarantee* that the digital signature will remain valid over the entire time span.**

*Hint: You need to think whether simply a longer key or an alternative algorithm would allow you to guarantee that the digital signature will remain valid over the entire time span. You may assume however that no algorithm becomes insecure overnight (just as for MD5, SHA-1, RSA-768, etc. the warnings had been coming for years before their security was broken). You may also use a trusted third party.*

## Answer

### Part 1 — Why Longer Keys Cannot Provide a Guarantee

Switching to a longer RSA key (e.g. 4096-bit) **cannot guarantee** validity over 30 years. **Shor's algorithm** (ch2 PQCrypto p.12) solves integer factorisation and discrete logarithms in **polynomial time** — key length is irrelevant, because even RSA-4096 would be broken once a sufficiently large quantum computer exists. The same applies to ECDSA: it relies on the discrete logarithm problem, which Shor's algorithm also solves (ch2 PQCrypto p.18, p.19). No currently used asymmetric signature scheme (RSA, DSA, ECDSA, ECDH) survives a quantum computing breakthrough.

**Why not just use ECDSA with a longer curve?** Shor's runs in polynomial time regardless of the curve size. Doubling or quadrupling the key size does not help — the algorithm's complexity class changes entirely.

A guarantee of 30-year validity therefore cannot rest on any single existing asymmetric algorithm.

### Part 2 — Solution: Post-Quantum Signature Algorithm

For new long-validity signatures, use a **NIST-standardised post-quantum signature algorithm** (ch2 PQCrypto p.29). NIST finalised two categories in August 2024:

- **ML-DSA** (CRYSTALS-Dilithium, lattice-based, ch2 PQCrypto p.25, p.35): based on the hardness of lattice problems (Learning With Errors), believed resistant to both classical and quantum attacks.
- **SLH-DSA** (SPHINCS+, hash-based, ch2 PQCrypto p.27, p.29): constructed entirely from hash functions. Its security reduces to hash function collision resistance — not to any number-theoretic assumption.

**SLH-DSA is the more conservative choice for a 30-year document**: its security assumption (hash function collision resistance) is simpler and more widely analysed than lattice assumptions. Hash functions survive Grover's quantum algorithm with halved bit security — SHA-512 retains 256-bit preimage resistance post-quantum (ch2 PQCrypto p.17), which is more than sufficient.

**Why SLH-DSA over ML-DSA?** ML-DSA's security rests on lattice problem hardness, which is a newer and less battle-tested assumption. Hash-based security is better understood (ch2 PQCrypto p.43). For the most conservative long-term guarantee, the simpler assumption is preferable.

**Important caveat**: PQC quantum safety is still **conjectured, not proven** — surprises remain possible (ch2 PQCrypto p.43). This is precisely why Part 3 (timestamp chain renewal) is mandatory even when using post-quantum algorithms.

### Part 3 — Chained Timestamps from a Trusted Third Party

Even a post-quantum signature cannot be *guaranteed* secure for 30 years with certainty. The **trusted third-party timestamp server** (ch3.1 p.6) provides a mechanism that survives any single algorithm failure, as long as each renewal occurs before the previous algorithm is broken.

#### Step 1 — Initial Signing (Today)

The document (e.g. mortgage contract) is signed with **SLH-DSA** over a **SHA-512** hash of the document content (ch2.2.3 p.24–32). Immediately after signing, a TTP timestamp is applied:

```
Tâ‚ = TTP_signature_1{ SHA-512(document || SLH-DSA_signature) , time_t1 }
```

Tâ‚ proves the SLH-DSA signature existed at time tâ‚ while the algorithm was considered secure.

**Why SHA-512 over SHA-256?** SHA-256 has 128-bit preimage resistance post-quantum (ch2 PQCrypto p.17). SHA-512 has 256-bit preimage resistance post-quantum (ch2 PQCrypto p.17) — more appropriate for a 30-year commitment where hash collision attacks may become practical over time.

#### Step 2 — Renewal Before Any Algorithm Weakens

No algorithm breaks overnight — warnings appear years in advance (ch2 PQCrypto p.15; historical examples: MD5, SHA-1, RSA-768). Before SLH-DSA or SHA-512 is considered potentially weak, a new timestamp is applied using the then-current strong algorithm:

```
Tâ‚‚ = TTP_signature_2{ SHA-512(document || original_signature || Tâ‚) , time_t2 }
```

Tâ‚‚ covers all prior material (document + original signature + Tâ‚) and is signed by the TTP using whatever algorithm is then current and strong — possibly a second-generation post-quantum algorithm if SLH-DSA has by then been deprecated.

This creates a **chain of timestamps**. Each link in the chain is renewed before the previous link's algorithm weakens.

#### Why This Chain Provides a Guarantee

The validity argument at any future time t:
1. The SLH-DSA signature was valid at tâ‚ — proven by Tâ‚, issued while SLH-DSA was secure and unbroken.
2. Tâ‚ was covered by Tâ‚‚ before Tâ‚'s algorithm weakened — even if SLH-DSA is later broken, the existence of the signature at tâ‚ is anchored in Tâ‚.
3. Tâ‚‚ is covered by Tâ‚ƒ before Tâ‚‚'s algorithm weakens, and so on.

As long as each renewal happens in time, the chain of proof is unbroken regardless of which specific algorithms are eventually broken.

#### Practical Schedule

- Sign with SLH-DSA + SHA-512 immediately at document creation.
- Apply TTP timestamp immediately after signing.
- Monitor NIST and BSI algorithm recommendations; renew the timestamp chain every 5–10 years, well in advance of any published weakness warning.
- Store the complete archival package: original document + original SLH-DSA signature + all TTP timestamp tokens Tâ‚, Tâ‚‚, ...

### Part 4 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Signature algorithm | SLH-DSA (SPHINCS+, hash-based) | ch2 PQCrypto p.27, p.29 | Hash-based security assumption; simpler and more conservative than lattice (ML-DSA) |
| Document hash | SHA-512 | ch2.2.3 p.24–32; ch2 PQCrypto p.17 | 256-bit preimage resistance post-quantum; SHA-256 gives only 128-bit post-quantum |
| Long-term validity | TTP timestamp chain renewed every 5–10 years | ch3.1 p.6 | Survives any single algorithm failure; chain of proof unbroken as long as renewals are timely |
| Why not RSA-4096 | Broken by Shor's in polynomial time regardless of key size | ch2 PQCrypto p.12, p.18 | Key length irrelevant against Shor's algorithm |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.24–32)
- IS_UG_3_1_Appl_Basics (p.3, p.6)
- IS_UG_2_2_SecM-adv-PQCrypto (p.12, p.15, p.17, p.18, p.19, p.25, p.27, p.29, p.35, p.43)

_Status: Complete_  
_Done by: William_
