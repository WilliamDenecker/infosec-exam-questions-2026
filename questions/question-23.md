# Question 23

**Describe how AES-GCM is used in TLS v1.3 to encrypt subsequent fragments.**

**How are the counter values determined? What "associated data" will be authenticated but not encrypted and why?**

## Answer

### AES-GCM in TLS 1.3 Record Protocol (ch3.6 p.7–8, ch2.2.3 p.70–75)

After the TLS 1.3 handshake completes, application data is protected using the negotiated AEAD cipher — in the case of `TLS_AES_256_GCM_SHA384`, this is AES-256-GCM. Each chunk of application data is wrapped in a **TLS record** before being encrypted.

---

### Structure of a TLS 1.3 Record

A plaintext TLS record (before encryption) consists of:
- **Content type** (1 byte): identifies the record type (application_data = 23)
- **Legacy record version** (2 bytes): always `0x0303` (TLS 1.2 for compatibility)
- **Length** (2 bytes): length of the encrypted payload
- **Plaintext data**: the actual application data fragment

After encryption, the record becomes:
- **Header** (5 bytes = content type byte `0x17` + version `0x0303` + length): the associated data — authenticated but not encrypted
- **Encrypted payload**: the encrypted record body, containing the actual content type and data inside
- **GCM authentication tag** (16 bytes): appended to the ciphertext

---

### How Counter Values Are Determined (ch2.2.3 p.70–75, ch3.6 p.7–8)

**GCM uses CTR mode internally for encryption**. The counter input to each AES call is a 128-bit block structured as:

$$J_i = (IV_{padded}) \oplus (i)$$

where:
- $IV_{padded}$ is the 12-byte (96-bit) GCM nonce padded to 16 bytes with a 32-bit counter appended (for 96-bit IVs, the nonce fills the first 96 bits and the counter starts at 1)
- $i$ is the block index (32-bit counter, starting at 1 for the first plaintext block; counter value 0 is reserved for the tag computation)

**In TLS 1.3, the GCM nonce (IV) is derived as follows** (ch3.6 p.7–8):

1. The handshake key schedule derives a **write IV** (12 bytes / 96 bits) for each traffic direction:
   - `client_write_iv` (client → server)
   - `server_write_iv` (server → client)

2. A **sequence number** is maintained separately for each direction, starting at 0 and incrementing by 1 for each record sent. The sequence number is a 64-bit big-endian integer.

3. For each record, the GCM nonce is computed by XOR-ing the write IV with the left-padded sequence number:
   $$nonce = write\_iv \oplus (seqnum\_padded\_to\_12\_bytes)$$

   Since the write IV is 12 bytes and the sequence number is 8 bytes (64 bits), the sequence number is zero-padded on the left to 12 bytes before XOR.

**Why XOR with the sequence number?**
- It ensures each record uses a unique, unpredictable nonce (ch2.2.3 p.70 — GCM nonce uniqueness requirement: never reuse a nonce with the same key)
- Nonce reuse in GCM is catastrophic: reusing a (key, nonce) pair allows recovery of both the plaintext and the GHASH key $H$, completely breaking authentication and confidentiality
- The XOR with the sequence number changes the nonce predictably but uniquely for each record, guaranteeing nonce uniqueness as long as the sequence number is unique (which it is, by definition)
- Using the XOR (instead of, say, concatenation) means even if the write IV has a predictable structure, the resulting nonce changes for every record

**Counter blocks within a record**: inside a single GCM encryption of one record:
- Counter block 0 ($J_0$): used to compute the authentication tag (last step)
- Counter blocks $J_1, J_2, \ldots, J_m$: used for CTR-mode encryption of plaintext blocks

The counter within a record is a 32-bit value incremented from 1 to $m$ (the number of 128-bit blocks in the record). It is appended to the nonce's first 96 bits as a 32-bit big-endian integer. Since the record is a single GCM invocation, this internal counter starts fresh at 1 for each record.

---

### Associated Data (Authenticated but Not Encrypted)

**What constitutes the associated data**: in TLS 1.3, the **TLS record header** is authenticated as associated data:
```
TLS record header (5 bytes):
  content_type: 0x17 (23 = application_data)
  legacy_version: 0x03 0x03
  length: 2-byte big-endian length of the ciphertext payload (including the 16-byte GCM tag)
```

**How GCM handles associated data** (ch2.2.3 p.70–75): GCM's GHASH authentication function takes both the ciphertext AND the associated data as input to compute the authentication tag. The associated data is included in the polynomial evaluation but is not encrypted. Any modification to the header (including the content type or length field) would cause the GHASH authentication tag to fail, and the receiver would reject the record.

**Why the header is authenticated but not encrypted**:

1. **Framing necessity**: the TLS record layer needs the content type and length fields to locate record boundaries in the TCP byte stream. If the length field were encrypted, the receiver could not determine where one record ends and the next begins without first decrypting — creating a bootstrapping problem. The length must be readable in plaintext.

2. **Integrity of framing**: by including the header as associated data, GCM's authentication tag covers it. An attacker cannot modify the record type or length without causing authentication failure. If these were outside GCM entirely (not even authenticated), an attacker could e.g. modify the length field to cause the receiver to misparse record boundaries.

3. **Traffic analysis**: the content type field `0x17` (application_data) is always the same for all TLS 1.3 data records — TLS 1.3 moved the real content type inside the encrypted payload (the last byte of the plaintext inside the record). The outer content type is a constant, providing no useful information to a passive eavesdropper. The length and record count do reveal some traffic metadata (record sizes, timing), which is an accepted limitation.

4. **Efficiency**: including a small header as associated data adds negligible cost to GCM authentication — the GHASH computation over 5 bytes is trivial compared to the ciphertext authentication.

---

### Summary

| Aspect | TLS 1.3 AES-256-GCM Implementation |
|---|---|
| Key | 256-bit write key, separately for each direction |
| Nonce construction | $write\_iv \oplus seqnum$ (both 96 bits); unique per record |
| Nonce reuse prevention | Sequence number increments by 1 per record |
| CTR counter within record | 32-bit counter, starts at 1 for first plaintext block; 0 reserved for tag |
| Associated data | 5-byte TLS record header (content type + version + length) |
| Authentication coverage | Header + full ciphertext in GHASH |
| Tag length | 128 bits (16 bytes) |

### Sources

- IS_UG_3_6_Appl_TLS (p.7–8: TLS 1.3 record protocol, key derivation, nonce construction)
- IS_UG_2_2_3_SecM_HashMac (p.70–75: GCM — counter construction, associated data, GHASH authentication)

_Status: Complete_  
_Done by: William_
