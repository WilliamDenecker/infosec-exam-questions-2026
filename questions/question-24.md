# Question 24

**Discuss the potential challenges in the implementation of the RSA algorithm for a solution requiring digital signatures and the public key encryption of symmetric keys.**

**Consider key generation, key management, and regular use (i.e. digital signatures and public key encryption).**

## Answer

### Overview: RSA for Signatures and Key Encipherment (ch2.2.3 p.77–85)

RSA is used for two distinct purposes:
1. **Digital signatures**: the private key $d$ signs a hash; the public key $e$ verifies
2. **Public key encryption of symmetric keys**: the public key $e$ encrypts a session key; the private key $d$ decrypts

Both require careful implementation. The challenges span key generation, key management, and runtime operations.

---

### Challenge 1 — Key Generation (ch2.2.3 p.77–80)

**1a. Prime generation**

RSA key generation requires two large prime numbers $p$ and $q$ of approximately equal size (each $\approx n/2$ bits). Generating cryptographically strong primes requires:
- A source of **high-quality randomness**: the primes must be generated from a cryptographically secure pseudorandom number generator (CSPRNG). Weak randomness is catastrophic — if $p$ or $q$ can be predicted or guessed, the private key $d$ can be recovered by factoring $n$. Historical incidents (Debian RNG bug, some embedded device RNG weaknesses) show that poor randomness in prime generation can be exploited at scale.
- **Primality testing**: large numbers must be tested for primality. Probabilistic tests (Miller-Rabin) are used; the probability of a composite number passing $k$ rounds of Miller-Rabin is $4^{-k}$. Sufficient rounds (typically $k = 40$) must be applied.
- **Timing**: generating a 2048-bit RSA key pair takes on the order of milliseconds to seconds on a typical CPU (many candidate primes must be tested). On constrained devices (smart cards, embedded hardware), key generation can take much longer and is a practical obstacle.

**1b. Key size selection** (ch2.2.3 p.80)

The key size determines security: RSA-2048 ≈ 112-bit classical security; RSA-3072 ≈ 128-bit; RSA-4096 ≈ 140-bit. Choosing too small a key size risks future compromise; too large a key wastes computation. The key must be sized for the required security lifetime (see Question 18, 20).

**1c. Parameter constraints**

- $p$ and $q$ must be of similar size (difference should not be too large, to prevent Fermat factorisation)
- $p - 1$ and $q - 1$ should have large prime factors (to prevent Pohlig-Hellman attacks on RSA)
- $(p-1)/2$ and $(q-1)/2$ should also be prime (safe primes) for strongest protection
- $gcd(e, (p-1)(q-1)) = 1$ must be verified for the chosen public exponent $e$ (typically 65537)

---

### Challenge 2 — Key Management (ch3.2 p.28–53)

**2a. Private key storage and protection**

The RSA private key $d$ must be stored securely. If $d$ is compromised:
- All past sessions encrypted under $n, e$ can be decrypted (no forward secrecy — see Question 19)
- Signatures can be forged in the holder's name

Private key protection mechanisms:
- **Hardware security modules (HSMs)**: the private key is generated inside and never leaves the HSM. All signing/decryption operations occur inside the secure hardware. Even physical theft of the device does not expose $d$.
- **Encrypted key storage**: $d$ is stored encrypted under a symmetric key derived from a passphrase or another key. Requires the encryption key to be available at use time.
- **Secure enclaves**: on modern CPUs (Intel SGX, ARM TrustZone), private key operations can be isolated in hardware-protected memory regions.

**2b. Certificate management and distribution** (ch3.2 p.28–53)

The RSA public key must be distributed in a trustworthy way — typically via X.509 certificates (ch3.2 p.28–29). This requires:
- Obtaining a certificate from a CA (verifying identity, paying fees, generating a CSR)
- Certificate lifecycle management: renewing before expiry, revoking if compromised
- CRL/OCSP checking at verification time (ch3.2 p.50–53)
- Different certificates for different key usages (signature key ≠ encryption key — see below)

**2c. Separate keys for signing vs. encryption**

Using the same RSA key pair for both digital signatures and key encipherment creates risks:
- A certificate granting both capabilities gives the holder more power than necessary (violates least privilege, ch3.7 p.46)
- Key escrow for encryption (some organisations store decryption keys for data recovery) conflicts with non-repudiation requirements for signing (an escrowed signing key could be used to forge signatures)
- Separate keys allow independent revocation: an encryption key can be escrowed; a signing key should never be escrowed

**Best practice**: maintain two separate RSA key pairs — one certified for signing only ($keyUsage: digitalSignature$) and one for key encipherment only ($keyUsage: keyEncipherment$). The X.509v3 key usage extension enforces this distinction (ch3.2 p.28–29).

---

### Challenge 3 — Regular Use: Digital Signatures (ch2.2.3 p.77–85)

**3a. Padding scheme selection**

Raw RSA signing is insecure (see Question 21). Two padding schemes:
- **PKCS #1 v1.5**: older, deterministic, no tight security proof. Still widely deployed but should be replaced.
- **RSA-PSS**: randomised, provably secure in the random oracle model (ch2.2.3 p.85). Requires correct implementation of the PSS encoding and the MGF1 mask generation function. The salt length must be chosen correctly (equal to the hash output length for maximum security).

**3b. Hash function selection**

The signed value is $H(M)$ (or PSS encoding of $H(M)$). The hash function must be collision-resistant (ch2.2.3 p.14). MD5 and SHA-1 are broken for this purpose; SHA-256 minimum, SHA-384 for higher security margins.

**3c. Performance — private key operations are slow**

RSA signature generation (computing $\sigma = H(M)^d \bmod n$) involves modular exponentiation with a large exponent $d$ (of the same bit length as $n$). For RSA-2048, this requires approximately 2048 modular multiplications. This is:
- Significantly slower than symmetric operations (thousands of times slower than AES)
- Feasible for low-volume signing (a few signatures per second is usually acceptable)
- Can become a bottleneck at scale (e.g., a CA signing millions of certificates, or a high-traffic HTTPS server if RSA signing is in the critical path)

**CRT optimisation**: use the Chinese Remainder Theorem to speed up private key operations by a factor of ~4 (see Question 26). The CRT-based computation works modulo $p$ and $q$ separately, then combines the results. Care must be taken to verify the signature after CRT computation (see Question 26).

---

### Challenge 4 — Regular Use: Public Key Encryption of Symmetric Keys (ch2.2.3 p.77–85)

**4a. Padding scheme for encryption**

RSA encryption of a session key (e.g., AES-256 key = 32 bytes) requires proper padding:
- **PKCS #1 v1.5 encryption**: vulnerable to Bleichenbacher's adaptive chosen-ciphertext attack (a landmark 1998 attack). Decryption oracles (timing differences between valid and invalid padding) allow complete key recovery in millions of queries. Requires very careful constant-time implementation to mitigate, but fundamentally insecure if any oracle exists.
- **OAEP (PKCS #1 v2.1 / RSA-OAEP)** (ch2.2.3 p.77–79): provably secure against adaptive chosen-ciphertext attacks in the random oracle model. Modern implementations must use OAEP for RSA key encipherment.

**4b. Message size limitation**

RSA can only encrypt messages smaller than the modulus $n$. For RSA-2048, the maximum plaintext size with OAEP-SHA256 is $256 - 2 \times 32 - 2 = 190$ bytes. This is sufficient for an AES-256 key (32 bytes) but not for arbitrary data. RSA is therefore used for hybrid encryption only: RSA encrypts the session key; a symmetric cipher encrypts the data.

**4c. No forward secrecy**

RSA key encipherment provides no forward secrecy (see Question 19). If $d$ is compromised, all past session keys encrypted under $n, e$ can be recovered. This is why TLS 1.3 eliminated RSA key exchange in favour of ECDHE.

**4d. Constant-time implementation requirements**

RSA private key operations must be implemented in **constant time** — execution time must not depend on the value of $d$ or the plaintext being decrypted. Timing variations can leak information about $d$ through side-channel attacks. This requires:
- Constant-time modular exponentiation (Montgomery multiplication with blinding)
- Exponent blinding: compute $m^{d + r\phi(n)} \bmod n$ for random $r$, eliminating timing dependency on specific bits of $d$
- Constant-time padding verification

---

### Summary of Challenges

| Phase | Key Challenges |
|---|---|
| Key generation | Strong RNG, primality testing, prime size/structure constraints, slow on constrained hardware |
| Key management | Secure private key storage (HSM preferred), certificate lifecycle, separate keys for signing vs. encryption |
| Digital signatures | Correct padding (RSA-PSS), collision-resistant hash, performance (CRT optimisation), constant-time implementation |
| Key encipherment | OAEP (not PKCS #1 v1.5), message size limit (hybrid encryption), no forward secrecy, constant-time decryption |

### Sources

- IS_UG_2_2_3_SecM_HashMac (p.77–85: RSA algorithm, key generation, padding schemes, OAEP, RSA-PSS, CRT)
- IS_UG_3_2_Appl_AuthMeth (p.28–29: X.509 certificates; p.50–53: revocation; p.11: key storage considerations)
- IS_UG_3_7_Appl_System (p.46: least privilege, attack surface minimisation)

_Status: Complete_  
_Done by: William_
