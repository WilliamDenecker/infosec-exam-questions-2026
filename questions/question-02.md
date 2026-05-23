# Question 2

Intrusion detection systems must balance detection accuracy with operational usability.

**Explain the concepts of false positives and false negatives in intrusion detection.**

**Why are false positives often the more severe operational problem in practice?**

**Discuss how alert fatigue arises and why it undermines security monitoring.**

## Answer

### False Positives and False Negatives in IDS (ch3.7 p.77)

An **intrusion detection system (IDS)** (ch3.7 p.77) monitors network traffic or host activity and raises alerts when it detects patterns that match known attacks or anomalous behaviour.

**False positive** (ch3.7 p.77): the IDS raises an alarm when no actual attack is occurring. Legitimate traffic or normal system behaviour matches a detection rule, causing the IDS to incorrectly classify it as an intrusion. Example: a port scan used by a network administrator during maintenance triggers the IDS's scan-detection rule.

**False negative** (ch3.7 p.77): the IDS fails to raise an alarm when a real attack is occurring. The attack traffic does not match any detection rule, or falls below the detection threshold, and passes through undetected. Example: a sophisticated attacker fragments packets to avoid signature matching, and the IDS misses the intrusion entirely.

The two errors represent opposite trade-offs in the IDS sensitivity configuration:
- Increasing sensitivity (lower threshold) → detects more real attacks (fewer false negatives) but also raises more spurious alerts (more false positives)
- Decreasing sensitivity (higher threshold) → fewer spurious alerts but more attacks go undetected

---

### Why False Positives Are the More Severe Operational Problem

**False negatives are dangerous** — a missed attack can result in a successful breach. However, a single missed attack is a bounded event that may or may not succeed depending on subsequent defences (packet filters, EPP, application-level controls).

**False positives are operationally destructive at scale** for several interconnected reasons:

**1. Volume multiplication**: In a production network handling thousands or millions of connections per hour, even a 0.001% false positive rate generates dozens of spurious alerts per hour. A 0.1% rate generates thousands. Each alert nominally requires human investigation. No security team has the capacity to investigate thousands of alarms per day — the workload is simply infeasible.

**2. Opportunity cost**: Every hour spent investigating a false positive is an hour not spent on real threats, patch management, threat hunting, or improving defences. False positives do not just waste time — they actively consume resources that would otherwise contribute to security.

**3. Credibility erosion**: When analysts repeatedly investigate alerts and find nothing, they begin to form a mental model that the IDS is unreliable. This is a rational response to repeated negative outcomes. The IDS loses credibility as a detection tool even when it generates legitimate alarms.

**4. Alert fatigue** (ch3.7 p.85): the cumulative effect of sustained high false-positive volumes leads to a systematic degradation of analyst response quality. This is the most dangerous consequence and is discussed in detail below.

---

### Alert Fatigue and Why It Undermines Security Monitoring (ch3.7 p.85)

**Alert fatigue** is the condition in which security analysts, overwhelmed by a sustained volume of alerts, develop behavioural patterns that reduce their effectiveness at detecting and responding to real intrusions (ch3.7 p.85).

**Mechanism of alert fatigue**:

**Phase 1 — Overload**: the IDS generates more alerts than analysts can investigate. With a threshold-based IDS (ch3.7 p.85), common network events (failed logins, port probes, abnormal traffic volumes) generate continuous low-priority alerts. The analyst queue grows faster than it can be processed.

**Phase 2 — Prioritisation shortcuts**: analysts cannot investigate every alert. They develop heuristics: "this alert type is almost always false" or "alerts from this source are never real." These heuristics are rational efficiency responses but introduce systematic blind spots. An attacker who understands the IDS ruleset can craft attacks that match the profiles analysts have learned to dismiss.

**Phase 3 — Delayed response**: even when a real attack generates an alert, it enters a queue behind dozens of lower-priority (but still unprocessed) alerts. By the time it reaches an analyst, the attacker has had hours of uncontested access. Real-time detection (ch3.7 p.83) becomes impossible when the alert queue is backlogged.

**Phase 4 — Systematic dismissal**: at peak fatigue, analysts may disable or tune out entire alert categories to reduce workload. This is the ultimate failure mode — the IDS is effectively decommissioned in the categories where it has generated the most false positives. If an attacker has already identified these categories, they can operate in them with near-zero detection risk.

**Why this fundamentally undermines IDS value**:

The purpose of an IDS is to detect intrusions in time to respond (ch3.7 p.77). Alert fatigue breaks this in two ways:

1. **Detection failure**: real attack alerts are missed, dismissed, or delayed because the analyst's attention and cognitive capacity have been depleted by false positive processing.

2. **Response failure**: even when a real alert is noticed, the analyst may under-prioritise it ("this is probably another false positive") and not escalate promptly. The dwell time — the period between intrusion and response — increases.

The consequence is that a high-false-positive IDS can be worse than no IDS: it consumes significant analyst resources while delivering poor detection, and creates an illusion of security monitoring that masks the actual absence of effective detection.

**Mitigations** (ch3.7 p.85):
- Careful tuning of detection thresholds to the specific network baseline
- Tiered alerting: high-confidence alerts escalate immediately; low-confidence alerts are batched
- Automated correlation: group related low-confidence events into higher-confidence compound alerts
- Regular review of false-positive rates by alert category and tuning or retiring ineffective rules
- Supplementing signature-based detection with anomaly-based detection (ch3.7 p.79–81) which can reduce false positives on known-good traffic profiles

### Sources

- IS_UG_3_7_Appl_System (p.77: IDS definition and error types; p.83: real-time alerting; p.85: threshold detection and alert fatigue)

_Status: Complete_  
_Done by: William_
