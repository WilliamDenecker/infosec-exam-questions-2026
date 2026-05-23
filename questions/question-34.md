# Question 34

Intrusion detection systems are commonly based on either statistical methods or rule-based approaches.

**Compare statistical detection of anomalous behaviour with rule-based detection using expert-defined rules.**

**For each approach, discuss:**

- **strengths and weaknesses,**
- **types of attackers it is better suited to detect,**
- **dependence on prior knowledge of vulnerabilities.**

**Explain why hybrid IDS solutions are commonly used.**

## Answer

### Rule-Based (Signature-Based) Detection (ch3.7 p.77–85)

**How it works** (ch3.7 p.79): the IDS maintains a database of **signatures** — patterns describing known attacks. Signatures are typically derived from actual attack traffic: specific byte sequences in packets, specific system call sequences, specific network protocol anomalies. For each monitored event (packet, log entry, system call), the IDS checks whether it matches any signature in the database.

**Strengths**:
- **Low false positive rate for known attacks**: a signature precisely describes a known attack. If the signature matches, there is high confidence that the attack is occurring. Security teams can tune signatures to be specific enough that false positives are rare for known threats.
- **Immediate actionability**: an alert from a signature tells the analyst exactly which attack is occurring (e.g., "Heartbleed exploit attempt on port 443"). No further investigation needed to understand the nature of the event.
- **Fast detection**: matching against signatures is computationally efficient — even large signature databases can be searched in microseconds using string matching algorithms (Aho-Corasick, Boyer-Moore).
- **Deterministic**: the same traffic always produces the same outcome. Reproducible, testable, auditable.

**Weaknesses**:
- **Cannot detect zero-day attacks**: if no signature exists for a new attack, it will not be detected. An attacker using a newly discovered vulnerability (zero-day) bypasses the IDS completely. The IDS is only as current as its signature database.
- **Evasion via obfuscation**: sophisticated attackers can modify the form of an attack (fragment packets, encode payloads, vary byte patterns) to avoid matching existing signatures while still executing the same attack. The underlying exploit is unchanged but the signature no longer matches.
- **Signature database maintenance**: signatures require constant updating as new attacks are discovered. A database that is days or weeks out of date leaves the system blind to recent attacks.
- **False negatives from signature gaps**: any attack variant not covered by existing signatures passes undetected.

**Best suited to detect**:
- **Script kiddies** and automated attack tools (Metasploit, off-the-shelf exploits): these use known, fixed attack patterns that match published signatures
- **Mass-exploitation campaigns**: automated attacks using the same payload against millions of targets
- **Malware with known signatures**: known malware families have characteristic network behaviours

**Dependence on prior knowledge**:
- **High**: a rule-based IDS is completely dependent on prior knowledge of vulnerabilities and their exploitation patterns. Unknown attacks are invisible.

---

### Statistical (Anomaly-Based) Detection (ch3.7 p.79–85)

**How it works** (ch3.7 p.80): the IDS first establishes a **baseline** of normal behaviour — typical network traffic volumes, normal system call patterns, expected user login times, usual data transfer rates, etc. During operation, the IDS measures current behaviour and computes statistical deviations from the baseline. Events that deviate significantly (above a configurable threshold, ch3.7 p.85) trigger an alert.

**Strengths**:
- **Can detect novel/zero-day attacks**: anomaly detection does not require prior knowledge of specific attacks. Any behaviour that deviates significantly from the baseline — including completely new attack types — triggers an alert. This is the primary advantage over rule-based detection.
- **No signature database required**: the IDS learns normal behaviour and flags deviations; it does not need attack signatures to be updated.
- **Detects slow/subtle attacks**: if an attacker gradually escalates privileges or exfiltrates data slowly over time, statistical methods accumulate deviations and eventually trigger an alert — even if no single event matches any known signature.
- **Detects insider threats**: a legitimate user who begins behaving abnormally (accessing unusual resources, exfiltrating data outside business hours) deviates from their personal baseline. Rule-based detection, which looks for external attack patterns, misses this.

**Weaknesses**:
- **High false positive rate** (ch3.7 p.85): legitimate changes in behaviour (seasonal traffic peaks, new software deployments, staff working unusual hours) trigger false positives. The baseline may not capture all legitimate behaviour patterns, leading to chronic alert noise (see Question 2 on alert fatigue).
- **Baseline pollution**: if an attacker establishes a presence slowly and consistently over time, their malicious activity can become "normalised" into the baseline. The IDS eventually treats the attack as normal behaviour.
- **Requires training period**: establishing a meaningful baseline requires weeks of normal operation data. A newly deployed IDS has poor anomaly detection until it has learned the environment.
- **Difficult to tune**: the threshold between "anomalous" and "normal" is not obvious. Set too low → excessive false positives; set too high → attacks are missed. Finding the right threshold requires ongoing tuning.
- **Low specificity in alerts**: an anomaly alert says "something unusual happened" but not "this specific attack occurred." Analysts must investigate further to determine if the deviation is malicious.
- **Evasion via "normalisation"**: sophisticated attackers who understand the baseline can craft attacks that fall within expected statistical bounds (e.g., data exfiltration at normal throughput rates over an extended period).

**Best suited to detect**:
- **Advanced Persistent Threats (APTs)**: sophisticated, patient attackers who use novel techniques not yet covered by signatures, or who behave subtly enough to avoid signature detection
- **Zero-day exploits**: new attacks without known signatures but with abnormal system behaviour
- **Insider threats**: authorised users who abuse their access (statistical deviation from their normal pattern)
- **Unknown malware**: malware with novel network behaviour patterns

**Dependence on prior knowledge**:
- **Low**: anomaly detection requires knowledge of **normal behaviour** (not of attacks). An unknown attack is detectable if it deviates from normal. However, defining "normal" accurately is itself challenging and environment-specific.

---

### Why Hybrid IDS Solutions Are Commonly Used (ch3.7 p.77–85)

A hybrid IDS combines both approaches, compensating for each other's weaknesses:

| Coverage | Rule-based alone | Anomaly alone | Hybrid |
|---|---|---|---|
| Known attacks | **Yes** (signature match) | Inconsistent | Yes |
| Zero-day attacks | **No** | **Yes** (if deviation large) | Yes |
| Insider threats | **No** | **Yes** | Yes |
| False positive rate | Low (for known attacks) | **High** | Moderate (each approach catches what the other misses cleanly) |
| Alert specificity | **High** (identifies attack) | Low | High for known; low for anomalies |

**Practical complementarity**:

1. The rule-based component handles the high-volume, well-defined alerts from known attacks quickly and with precision. Security teams process these efficiently because the alert type identifies the response action immediately.

2. The anomaly-based component catches novel attacks and insider threats that rule-based detection misses. Its alerts are lower-specificity and require investigation, but they fill the critical gap of zero-day coverage.

3. The two components can cross-validate: an event that triggers both a signature alert AND an anomaly alert is more likely to be a genuine attack. Events that trigger only the anomaly detector require investigation; events that trigger only the signature detector can be handled by playbook.

4. False positive reduction: anomaly alerts that co-occur with signature alerts can be deprioritised (the signature provides the specific identification); signature alerts for traffic that the anomaly detector considers normal can be treated as higher-confidence.

**Example integration**: a SIEM (Security Information and Event Management) system collects logs from both detection approaches. Rule-based alerts are automatically correlated with asset inventories and threat intelligence feeds. Anomaly alerts from user behaviour analytics are escalated only when combined with other indicators (unusual access time + high data volume + new destination).

The hybrid approach reflects a core principle from the course: **defence in depth** (ch3.7 p.46) — multiple independent security mechanisms, each catching what the other misses.

### Sources

- IS_UG_3_7_Appl_System (p.77: IDS overview; p.79–81: anomaly-based vs. signature-based detection; p.83: alert management and correlation; p.85: threshold detection and tuning; p.46: defence in depth)

_Status: Complete_  
_Done by: William_
