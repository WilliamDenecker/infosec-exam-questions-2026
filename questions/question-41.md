# Question 41

Classical system security relies heavily on the concept of a network perimeter.

**Explain why firewalls alone are insufficient to protect modern systems.**

**Discuss at least three concrete limitations of perimeter security in the context of:**

- **stolen credentials,**
- **malware,**
- **mobile and cloud-based users.**

**Explain how these limitations influence modern system security design.**

## Answer

### The Perimeter Security Model (ch3.7 p.46–51)

The classical perimeter security model treats the network as having a clearly defined inside (trusted) and outside (untrusted), with a **packet filter firewall** (ch3.7 p.51) as the gatekeeper. Traffic from the internet is blocked or permitted by firewall rules; traffic inside the network is trusted. The model assumes: "if you're inside the perimeter, you're authorised."

This model worked reasonably well when:
- All users and resources were inside a physical building
- Network topology was simple and static
- Attacks came exclusively from external adversaries
- Credentials and devices were under organisational control

Modern networks violate all these assumptions. The perimeter has effectively dissolved.

---

### Limitation 1 — Stolen Credentials (ch3.2 p.11; ch3.7 p.20)

**The problem**: a firewall authenticates connections by IP address, port, and protocol — not by user identity. Once a legitimate user authenticates to a VPN or internal system, they are "inside the perimeter." If an attacker steals that user's credentials (password, session token, VPN certificate), the attacker's traffic looks identical to the legitimate user's traffic. The firewall passes it without challenge.

**How credentials are stolen**:
- Phishing attacks: user enters credentials on a fake login page
- Password database breaches: compromised servers expose password hashes
- Malware (keyloggers): credentials captured as the user types
- Social engineering: IT help desk tricked into resetting passwords

**Consequence**: a perimeter firewall that permits authenticated sessions cannot distinguish between the legitimate user and an attacker using stolen credentials. Once inside, the attacker has the same access as the legitimate user — including to sensitive internal servers that the firewall would never expose directly to the internet.

**What a firewall cannot do**: a packet filter (ch3.7 p.51) inspects IP addresses, ports, and protocols. It does not inspect application-layer authentication tokens, session validity, or whether the behaviour matches the user's normal pattern. An authenticated but stolen session bypasses the firewall legitimately.

---

### Limitation 2 — Malware (ch3.7 p.38–43)

**The problem**: the perimeter model assumes that threats come from outside. Malware can be introduced into the interior network through paths that the firewall legitimately allows:
- Email with malicious attachments (delivered through SMTP/HTTPS — allowed inbound channels)
- USB drives brought inside by employees
- Software vulnerabilities in internally-facing services
- Malicious code in supply chain software updates (legitimate software channels)

Once malware establishes a presence inside the perimeter, the firewall provides no protection against its lateral movement to other internal systems. The malware can:
- Connect to internal file shares, databases, and servers (all inbound-blocked from the internet but accessible internally)
- Exfiltrate data via outbound HTTPS connections (firewalls typically permit outbound HTTPS — it would block all web browsing otherwise)
- Spread to other internal hosts via network shares, vulnerabilities, or lateral movement techniques
- Download additional malicious payloads from external command-and-control servers (via allowed outbound HTTP/HTTPS)

**Command-and-control evasion**: modern malware initiates outbound connections (not inbound) to attacker-controlled servers. Firewalls configured to "allow outbound, block inbound" pass these connections freely. The firewall rule designed to block external attack attempts is entirely irrelevant when the attacker's foothold is already inside.

**Ransomware** (ch3.7 p.38): a specific case — ransomware deployed inside the perimeter encrypts all accessible file systems, including network shares. The firewall never sees this attack; it is entirely internal.

---

### Limitation 3 — Mobile and Cloud-Based Users (ch3.7 p.46)

**The problem**: modern work does not happen inside a physical building on a corporate network. Employees work from:
- Home networks (not controlled by the organisation)
- Coffee shops and airports (public, hostile networks)
- Cloud platforms (AWS, Azure, GCP — not "inside" any traditional perimeter)
- Personal mobile devices (BYOD — not managed by the organisation)

**Breakdown of the inside/outside distinction**:

- **Cloud services**: when an organisation's data sits in AWS S3 or Azure Blob Storage, it is physically located in a data centre outside any corporate firewall. A perimeter that protects the corporate building provides zero protection to cloud-hosted resources. The cloud provider's network is not "inside" the perimeter.

- **Remote workers**: an employee working from home is outside the perimeter. If they VPN in, traffic flows through the perimeter firewall. But the employee's home device (potentially compromised, unpatched, shared with family members) is not subject to corporate security policy. Malware on the home device can traverse the VPN tunnel.

- **Mobile devices**: smartphones and tablets used for email, calendar, and document access operate entirely outside any corporate perimeter. A firewall protecting the corporate network provides no protection for data on a stolen or compromised phone.

- **SaaS applications**: when users access Office 365, Salesforce, or other SaaS platforms, traffic goes directly from the user's device to the SaaS provider — bypassing any corporate perimeter entirely. The firewall never sees this traffic.

---

### How These Limitations Influence Modern Security Design (ch3.7 p.46–47)

**1. Defence in depth** (ch3.7 p.46): since the perimeter cannot be relied upon as the last line of defence, multiple independent layers must be deployed at each level:
- Endpoint Protection Platforms (EPP/EDR) on every device (ch3.7 p.38–43) — protection moves to the endpoint, inside the perimeter
- Network segmentation: internal networks divided into zones; traffic between zones filtered by additional internal firewalls. A compromised host in one zone cannot freely access other zones.
- Application-level gateways (ch3.7 p.62–65) in front of internal services, applying protocol-aware filtering even for internal traffic

**2. Zero Trust Architecture** (ch3.7 p.46 — minimise attack surface): the zero-trust model assumes no user, device, or network segment is inherently trusted. Every access request is authenticated and authorised, regardless of whether it originates inside or outside the physical perimeter. Concretely:
- Every service requires authentication (not just "are you on the internal network?")
- Per-session, per-action authorisation checks
- Least privilege: users and services have access only to what they need for the current task
- Continuous monitoring: behaviour analytics detect anomalies even for authenticated sessions

**3. Credential protection** (ch3.7 p.20): multi-factor authentication is required for all access — stolen passwords alone are insufficient. Session token management (short lifetimes, device binding) limits the damage from credential theft.

**4. Endpoint security as primary defence**: since the perimeter cannot stop malware delivered through legitimate channels (email, USB), the endpoint must detect and contain threats. EDR (Endpoint Detection and Response, ch3.7 p.42) monitors process behaviour, file system changes, and network connections in real time, quarantining suspicious activity before it spreads.

**5. IDS/IPS on internal traffic** (ch3.7 p.77, p.85): intrusion detection systems monitor internal network traffic — not just the perimeter — to detect lateral movement, unusual data access patterns, and ransomware-like behaviour. An IDS that only monitors perimeter traffic is blind to 70% of modern attack activity.

**6. Cloud-native security controls**: security policies must follow data to the cloud through cloud-provider access controls (IAM policies, security groups, encryption of data at rest and in transit), not rely on a corporate firewall that the traffic never passes through.

### Sources

- IS_UG_3_7_Appl_System (p.38–43: EPP and EDR; p.46–47: minimise attack surface, defence in depth, zero-trust principles; p.51: packet filter firewalls and their limitations; p.62–65: application-level gateways; p.77: IDS; p.85: threshold detection; p.20: multi-factor authentication)
- IS_UG_3_2_Appl_AuthMeth (p.11: credential storage; p.20: TOTP for MFA)

_Status: Complete_  
_Done by: William_
