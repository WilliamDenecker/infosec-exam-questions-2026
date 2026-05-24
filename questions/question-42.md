# Question 42

Explain the "initial exchanges" in the Internet Key Exchange (IKE) in IPsec.

**What are the elements that guarantee the authenticity of the involved entities (Initiator (I) and Responder (R))? How is replay prevented? What cryptographic algorithms could be used (be sufficiently specific)?**

## Answer

### Overview of IKEv2 Initial Exchanges (ch3.4 p.34–44)

IKEv2 (Internet Key Exchange version 2, IETF RFC 7296) is the key management protocol for IPsec. Its **initial exchanges** consist of four messages in two request-response pairs — IKE_SA_INIT and IKE_AUTH — that establish a mutually authenticated, encrypted IKE Security Association (IKE SA) and the first pair of Child SAs (for actual IPsec traffic).

---

### Message Flow (ch3.4 p.39–40)

**Step 1 — IKE_SA_INIT: Initiator → Responder** (ch3.4 p.39)

```
I → R:  HDR, SAi1, KEi, Ni
```

| Field | Content |
|---|---|
| HDR | IKE header: initiator's SPI, responder's SPI (zero), exchange type, message ID, length |
| SAi1 | List of cryptographic suites supported by I for the IKE SA (cipher, PRF, integrity, DH group) |
| KEi | Initiator's Diffie-Hellman public value for the selected DH group |
| Ni | Initiator's nonce — fresh random value |

**Step 2 — IKE_SA_INIT: Responder → Initiator** (ch3.4 p.39)

```
R → I:  HDR, SAr1, KEr, Nr, [CERTREQ]
```

| Field | Content |
|---|---|
| SAr1 | The single cryptographic suite selected by R from SAi1 |
| KEr | Responder's DH public value |
| Nr | Responder's nonce |
| [CERTREQ] | Optional certificate request (which CA certificates R trusts) |

After these first two messages, both parties can independently derive (using their DH shared secret + both nonces):
- **SK_e** — encryption key for IKE messages in each direction
- **SK_a** — integrity/authentication key for IKE messages in each direction
- **SK_d** — key material for deriving Child SA keys

**Step 3 — IKE_AUTH: Initiator → Responder** (ch3.4 p.40)

```
I → R:  HDR, SK{ IDi, [CERT,] [CERTREQ,] [IDr,] AUTH, SAi2, TSi, TSr }
```

Everything inside `SK{·}` is encrypted and integrity-protected using the keys derived in Steps 1–2.

| Field | Content |
|---|---|
| IDi | Initiator's identity (typically an IP address or FQDN) |
| [CERT] | Optional X.509 certificate for I |
| [CERTREQ] | Optional request for R's certificate |
| [IDr] | Optional hint about which R identity I is communicating with |
| **AUTH** | Authentication payload — proves I's identity (see below) |
| SAi2 | Cryptographic suites supported by I for the Child SA (IPsec SA) |
| TSi, TSr | Traffic Selectors — IP address ranges and protocols for the Child SA |

**Step 4 — IKE_AUTH: Responder → Initiator** (ch3.4 p.40)

```
R → I:  HDR, SK{ IDr, [CERT,] AUTH, SAr2, TSi, TSr }
```

After Step 4, the first pair of **Child SAs** is established for normal IPsec traffic (AH or ESP).

---

### Authenticity of the Entities (ch3.4 p.38–44)

Authentication is achieved through the **AUTH payload** (ch3.4 p.44), which can use:

1. **Digital signature** (RSA or DSS/ECDSA): the AUTH payload contains a signature over essential parts of the first two messages (including nonces and the peer's identity). This proves ownership of the corresponding private key. Certificates (CERT payload) bind the public key to the identity (IDi or IDr).

2. **Pre-shared key (PSK)**: the AUTH payload is a keyed MAC computed over the same data using a symmetric key shared out-of-band. Authentication requires possession of the pre-shared key.

3. **Asymmetric encryption using private key**: alternative to signatures, less common.

The **identity payloads** (IDi, IDr) provide the claimed identity; the AUTH payload cryptographically binds that identity to the entity. The CERTREQ and CERT payloads provide the X.509 infrastructure to verify the binding between the public key and the identity.

---

### Replay Prevention (ch3.4 p.14, p.20, p.39)

Replay attacks are prevented at two levels:

**1. Nonces in IKE_SA_INIT (ch3.4 p.35, p.39)**:
- The initiator sends Ni (fresh random nonce) and the responder sends Nr
- The session keys (SK_e, SK_a, SK_d) are derived from a function of both nonces and the DH shared secret
- An attacker who replays a previous IKE_SA_INIT cannot produce a valid SK_e/SK_a for a new session (different nonces → different keys → MAC verification fails on IKE_AUTH)

**2. Message IDs in the IKE header (ch3.4 p.42)**:
- Each IKE message carries a unique monotonically increasing **Message ID** (32 bits)
- The responder verifies that each IKE request has a Message ID greater than the previous one
- A replayed IKE message with a stale Message ID is rejected

**3. Anti-replay window in the Child SA (ch3.4 p.14, p.20)**:
- Once Child SAs are established (AH or ESP), a sliding window mechanism (default size 64) tracks received sequence numbers
- Duplicate sequence numbers are rejected; packets outside the window are discarded

**4. Cookie mechanism (ch3.4 p.35–36)**:
- If a large number of half-open IKE SAs is detected (DoS defence), the responder can send a COOKIE notification instead of computing KEr
- The cookie is a hash of IP addresses, ports, and a locally generated secret — stateless for the responder
- Forces the initiator to prove it holds the source address before the responder performs costly DH computation

---

### Cryptographic Algorithms (ch3.4 p.24, p.37, p.21)

**Diffie-Hellman groups (ch3.4 p.37)**:
- **Elliptic curve groups (ECDH)**: RFC 5114, 5903 — NIST P-256 (256-bit), P-384, P-521, Curve25519
- **Modular exponentiation (classical DH)**: RFC 3526 — 2048-bit, 3072-bit, 4096-bit MODP groups
- Deprecated (RFC 9395): groups 1 and 2 (768/1024-bit MODP)

**Encryption algorithms (ch3.4 p.24)**:
- `AES-GCM` with 16-octet ICV (**MUST** per RFC 8221)
- `AES-128-CBC` (**MUST**)
- `DES`: MUST NOT (obsoleted)
- `3DES`: SHOULD NOT (obsoleted)

**Integrity/MAC algorithms (ch3.4 p.21)**:
- `HMAC_SHA2_256_128` (**MUST**)
- `HMAC_SHA2_512_256` (**MUST**)
- `HMAC_SHA1_96` (**MUST−**, deprecated in RFC 9395)

**Authentication of entities (ch3.4 p.38, p.44)**:
- RSA digital signature (over hash of AUTH data using SHA-256 or SHA-384)
- DSS / ECDSA digital signature
- Pre-shared key (HMAC-based AUTH)

---

### Summary

| Aspect | Mechanism |
|---|---|
| Key agreement | ECDH or classical DH (ephemeral, per-session) |
| Entity authentication | AUTH payload: RSA/ECDSA signature or PSK |
| Identity binding | X.509 certificates in CERT payload |
| Replay prevention | Nonces (Ni, Nr) + Message IDs + anti-replay window |
| DoS prevention | Cookie exchange before DH computation |
| Confidentiality of messages 3–4 | AES-GCM or AES-CBC with keys derived from DH + nonces |

### Sources

- IS_UG_3_4_Appl_IPSec (p.34–36: key management overview and Oakley/ISAKMP background; p.37: DH groups and ECDH; p.38: authentication mechanisms; p.39–40: IKE_SA_INIT and IKE_AUTH message flow; p.41: Child SA creation; p.42: IKE header format; p.43–44: SA, KE, ID, AUTH payload types; p.14: SA parameters and sequence number; p.20: anti-replay sliding window; p.21: HMAC algorithms; p.24: ESP encryption algorithms)

_Status: Complete_  
_Done by: William_
