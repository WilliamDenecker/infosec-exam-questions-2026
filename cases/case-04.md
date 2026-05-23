# Case 4

A company distributes software updates to millions of users via mirrors and content delivery networks (CDNs). Recent supply-chain attacks have made integrity and authenticity critical.

**Design a secure update system. Which security services are required? Which cryptographic mechanisms would you use to authenticate updates and protect against rollback attacks? How would you protect the signing keys? What remaining risks exist?**

## Answer

### Part 1 — Required Security Services (ch1 p.10)

| Service | Required? | Slide | Why it is needed | What breaks without it |
|---|---|---|---|---|
| **Data integrity** | Yes — critical | ch1 p.34 | An update must arrive exactly as the vendor produced it — nothing added, deleted, or modified in transit or on a mirror. | A tampered update installs malware on millions of devices. Even a single flipped byte could introduce a backdoor. |
| **Authentication / data-origin authentication** | Yes — critical | ch1 p.22 | The client must verify the update was produced by the legitimate vendor, not by a compromised mirror or attacker. | A rogue CDN node serves a malicious package; clients install it without warning. |
| **Non-repudiation** | Yes | ch1 p.40 | The vendor must not be able to deny having released a specific version. The digital signature is evidence of the release — critical for incident response. | A vendor can claim a malicious update "wasn't theirs" — no cryptographic proof otherwise. |
| **Confidentiality** | Partial | ch1 p.15 | Less critical for the package itself (its content is public), but required for the update channel to prevent traffic analysis revealing which vulnerable version a client runs. TLS provides this at no additional cost. | An eavesdropper learns which unpatched vulnerabilities each client has — targeted exploitation. |
| **Availability** | Yes | ch1 p.42 | Security patches must reach clients promptly. Mirror infrastructure must remain operational. | A DDoS on update servers prevents clients from patching a critical vulnerability — leaving millions exposed. |

### Part 2 — Update Signing

**Hash the package**: compute **SHA-256** (ch2.2.3 p.24–32) over the complete firmware binary. SHA-256 produces a 256-bit digest — any modification to the package changes the hash entirely.

**Sign the hash**: the vendor signs the hash using **ECDSA P-256** (ch2.2.3 p.85–87). ECDSA P-256 provides 128-bit security with compact 64-byte signatures efficiently verifiable on all clients including mobile and embedded devices.

**Why ECDSA P-256 over RSA-PSS?** RSA-PSS (ch2.2.3 p.88–93) achieves equivalent security with a 2048-bit key but produces 256-byte signatures. ECDSA P-256 produces 64-byte signatures — 4× smaller, faster to verify, especially relevant for embedded/IoT clients. RSA-PSS is an acceptable alternative for environments requiring RSA; the probabilistic padding (ch2.2.3 p.88–93) resists fault attacks.

**Why SHA-256 over SHA-1?** SHA-1 has known collision vulnerabilities — two different files can produce the same hash. SHA-256 (ch2.2.3 p.24–32) has no known practical collision attack.

The **signed manifest** (distributed separately from the binary, from the vendor's own server not mirrored):
```
manifest = {
    package_name,
    version_number,        // monotonically increasing integer
    SHA-256(package_binary),
    ECDSA_signature over above fields
}
```

The binary travels unsigned across CDN mirrors; the manifest is the trust anchor. Clients: (1) verify ECDSA signature on manifest, (2) download binary, (3) recompute SHA-256 and compare to manifest hash, (4) check version number > current installed version.

### Part 3 — Rollback Protection

The manifest includes a monotonically increasing **version number**. The client stores the highest version number it has successfully installed in write-protected OS storage. Any manifest with version ≤ stored value is rejected — even if its signature is valid. This prevents an attacker from rolling back to a known-vulnerable version.

**Why is a valid signature not sufficient?** Old versions have known vulnerabilities. An attacker who steals a legitimate old manifest can serve it via a rogue CDN. The version check is independent of the cryptographic validity check.

### Part 4 — Signing Key Protection (CA Hierarchy)

Use a **PKI-style hierarchy** (ch3.1 p.16–22):

1. **Root signing key** (offline, air-gapped): never connected to any network. Used only to sign the intermediate certificate. Stored on encrypted offline media in physical security.
2. **Intermediate signing key** (online, restricted server): used for daily release signing. Issued by the root key with a 1-year validity period.

This mirrors the offline root CA model (ch3.1 p.19): the root key is the offline trust anchor; the intermediate key handles operational signing. If the intermediate key is compromised, revoke it and re-issue from root. The root remains unaffected.

Clients embed the root public key at installation and verify the certificate chain: root → intermediate → manifest.

**Why an offline root key?** If the signing key is online (reachable from the internet), a server breach exposes it. An air-gapped offline root means an attacker who compromises the release server cannot forge manifests signed by the root (ch3.1 p.19).

### Part 5 — Distribution Channel

All client-to-server communications use **TLS 1.3** (ch3.6 p.7–8) with `TLS_AES_256_GCM_SHA384` and ECDHE (ch3.6 p.18) for forward secrecy (ch3.6 p.37). Manifests are fetched from the vendor's own HTTPS endpoint (not mirrored) — a compromised mirror cannot serve a forged manifest.

### Part 6 — System Security

**Packet filter on signing server** (ch3.7 p.51): no inbound internet connections. Signed manifests exported via a one-way mechanism. **Separation** (ch3.7 p.46): signing server, distribution server, and CDN infrastructure are independent systems. A CDN compromise cannot reach the signing key.

**IDS** (ch3.7 p.77, p.85): every signing operation logged with timestamp and operator identity. Alerts on signing outside scheduled release windows (threshold detection, ch3.7 p.85).

### Part 7 — Remaining Risks

- **Signing key compromise**: attacker can sign arbitrary malicious updates accepted by all clients. The offline root key limits blast radius to the intermediate; client revocation propagation must be rapid.
- **Build system compromise**: malicious code inserted before signing — the vendor signs the tampered binary in good faith. The signature is valid. No update-system cryptography can detect this; requires build system hardening.
- **Insider threat**: a developer with signing server access. Dual-control signing (two operators required) reduces this.
- **Version store tampering**: if the client's version store is writable by malware, rollback protection can be bypassed. Must be write-protected by the OS.

### Part 8 — Summary of Design Choices

| Component | Chosen Solution | Slide reference | Why better than alternatives |
|---|---|---|---|
| Package hash | SHA-256 | ch2.2.3 p.24–32 | No known collision attack; computationally efficient |
| Signature | ECDSA P-256 | ch2.2.3 p.85–87 | 64-byte compact signature; 128-bit security; verifiable on embedded clients |
| Rollback protection | Monotonic version number, client-verified | ch1 p.34 | Valid old signatures are still rejected; cryptographic validity alone is not sufficient |
| Key protection | Offline root CA hierarchy | ch3.1 p.16–22 | Online key compromise does not expose root; mirrors the PKI trust model |
| Distribution channel | TLS 1.3 for manifest; ECDHE for forward secrecy | ch3.6 p.7–8, p.18, p.37 | Traffic encrypted; even future key compromise does not expose past sessions |

### Sources

- IS_UG_1_Introduction (p.22, p.34, p.40, p.42)
- IS_UG_2_2_3_SecM_HashMac (p.24–32, p.85–87, p.88–93)
- IS_UG_3_1_Appl_Basics (p.16–22)
- IS_UG_3_6_Appl_TLS (p.7–8, p.18, p.37)
- IS_UG_3_7_Appl_System (p.46–47, p.51, p.77, p.83, p.85)

_Status: Complete_  
_Done by: William_
