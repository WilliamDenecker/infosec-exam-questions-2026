# Theme 5 — Fail-Safe and Availability: Local Function Must Survive Network Failure

**Appears in**: Cases 26, 27, 34, 37

---

## The Core Principle

Whenever a physical device has both a **local physical function** and a **remote network-controlled function**, the local physical function must operate regardless of network availability. The network layer is a convenience or enhancement layer — not a dependency for basic safety operation.

This principle appears in every case that involves a physical device connected to a cloud or remote control system. It is a security property (availability — ch1 p.42) and a safety requirement simultaneously.

**Stated precisely**: if the network, cloud server, or remote operator becomes unavailable, the device must:
1. Continue performing its primary safety/physical function
2. Not enter an unsafe or indeterminate state
3. Revert to its last-known safe local configuration

---

## Part 1 — Availability as a Security Service (ch1 p.42)

Availability is listed alongside confidentiality, integrity, and authentication as a core security service (ch1 p.10). A system that is encrypted and authenticated but unavailable when needed has failed. In physical systems:

- A smoke detector that does not sound an alarm because the cloud is down has failed its safety purpose
- An aircraft that cannot be controlled because the satellite link is broken is in immediate danger
- A car that cannot be locked because the cloud service is offline has failed its primary function

**Why availability is a security concern**: a Denial of Service (DoS) attack that takes down the cloud server does not need to break any cryptography to cause harm. If the device fails safe (continues operating locally), the DoS attack is neutralised. If the device fails unsafe (stops operating when cloud is unreachable), the DoS attack is effective — the attacker does not need to decrypt anything; they just need to make the server unavailable.

---

## Part 2 — Case 34: Smoke Detector

### Physical Function (Primary)
The smoke detector sensors CO and particulate matter. When the threshold is exceeded, the local buzzer sounds. This happens in hardware/firmware, entirely locally.

**Fail-safe requirement**: the alarm must sound even if:
- WiFi is not connected
- The home router is offline
- The cloud server is down
- The phone is out of battery

The alarm sounds in hardware the moment the sensor triggers. No cloud acknowledgment is required, no app interaction is needed, no network packet is sent or received before the buzzer activates.

### Remote Function (Enhancement Layer)
When the alarm sounds, a notification is sent over WiFi → cloud → push notification to the user's phone. This is a convenience layer: the user is notified when away from home.

**Fail-safe**: if the cloud is unreachable, the alarm still sounds locally. The user misses the push notification but the alarm is loud enough to wake anyone in the house. The remote notification's failure does not compromise the primary safety function.

### Silence Command
The user can silence the alarm via the app (send authenticated command to cloud → relay to detector → stop buzzer). But the detector also has a physical button that silences the alarm locally, requiring no network path.

**Why the physical button matters**: if the cloud is down, the user would otherwise be unable to silence an alarm in a legitimate false-alarm situation (burnt toast). The physical button is the fail-safe.

### Alarm Flooding Attack Mitigation
An attacker who can silence alarms remotely (by compromising the cloud account) could send repeated silence commands during a real fire. Mitigations:
- TOTP required for each silence command (attacker needs the second factor)
- IDS on server: repeated silence commands trigger alert (ch3.7 p.77)
- Physical button: local override always available to the resident

---

## Part 3 — Case 37: Aircraft

### Physical Function (Primary)
The onboard pilot physically operates the aircraft via the flight controls. This is always available — physical controls connected directly to flight surfaces, hydraulics, and engines. No network dependency.

### Remote Function (Enhancement Layer — Advisory Mode)
The ground co-pilot monitors sensor data (altitude, heading, speed, engine telemetry) transmitted from the aircraft. In advisory mode, the ground co-pilot can only observe; the onboard pilot retains full control. No command is sent; no network dependency for flight operation.

### Override Mode: The Exception That Confirms the Rule
In emergency override mode, the ground co-pilot can issue flight commands. This is the exception — and it is explicitly bounded:

```
Override mode requirements:
  - Onboard pilot activates or does not contest transfer
  - Ground co-pilot authenticates with fresh TOTP + signed request
  - Aircraft displays visible alert in cockpit ("GROUND OVERRIDE ACTIVE")
  - Onboard pilot can send signed REJECT OVERRIDE at any time
```

### Fail-Safe: Link Loss During Override

**This is the most critical fail-safe in case 37**: if the satellite datalink fails while override mode is active:

```
Link loss detected → immediate revert to local pilot control
Aircraft displays: "LINK LOST — LOCAL CONTROL RESTORED"
Ground co-pilot status: advisory only (no commands accepted)
```

**Why revert to local and not to "wait for link"?** An aircraft in flight cannot wait. If the link is lost:
- Waiting for reconnection means no one is flying the aircraft during the wait
- Attempting to maintain override with a broken link means the ground co-pilot sends commands that never arrive
- The only safe state is: the person physically present in the aircraft controls it

**Why not leave override active and use the backup VHF channel?** The VHF backup (case 37) is line-of-sight only. Over oceans (where commercial flight occurs), VHF may not reach a ground station. Link loss means truly isolated — revert to local is the only safe option.

**Dual redundant communication channels**:
- Primary: satellite datalink (~600ms latency, global coverage)
- Backup: VHF radio (line-of-sight, low latency, fallback for near-ground phases)
- VHF is activated automatically when satellite link quality degrades

---

## Part 4 — Case 26: Car Key Fob

### Physical Function (Primary)
The car fob uses a local radio signal directly to the car's receiver. No internet, no cloud, no server. The entire communication path is: fob radio transmitter → car's local RF receiver.

**Fail-safe by design**: the car fob is inherently fail-safe with respect to network failures because there is no network involved in the primary function. The car locks and unlocks regardless of mobile internet coverage, regardless of whether the manufacturer's servers are online.

**Why this matters**: if Deutsche Post's servers go offline, electronic stamps stop working (the barcode verification relies on the central used-stamp database). If the car manufacturer's servers go offline, the physical key fob still works. The fob mechanism's independence from any network is a core reliability property.

---

## Part 5 — Case 27: Smart Plug

### Physical Function
The plug passes or interrupts mains power to the connected device. The relay state (on/off) persists in firmware.

**Fail-safe**: if the cloud is unavailable:
- The plug maintains its last relay state — it does not default to off (which could disable critical equipment) or on (which could leave a heater running indefinitely)
- The plug cannot be remotely controlled while cloud is unavailable (acknowledged trade-off)
- Local function continues: the plug does what it was last told to do

**This is a deliberate trade-off**: cloud-mediated architecture means remote control depends on cloud availability. This is accepted because the plug's safety mode (maintaining last state) is non-dangerous for typical loads (lamps, coffee makers). For life-critical loads, a cloud-mediated smart plug is not appropriate — a direct-control system would be needed.

**Why not a local web server on the plug?** Security: the plug on the home network directly accessible as a web server creates an attack surface. A local server on the plug could be reached by any compromised device on the same network. Cloud-mediated architecture means the plug initiates outbound only; it is never directly reachable from the internet or from other local devices. Security is prioritised over availability of local control.

---

## Part 6 — General Design Rules for Fail-Safe Systems

### Rule 1: Identify the Safety-Critical Local Function First

Before designing any remote control capability, specify what the device does without any network:
- Smoke detector: sounds an alarm when sensor threshold exceeded
- Car: unlocks when correct rolling code received on local RF
- Aircraft: follows physical control inputs from the onboard pilot
- Smart plug: maintains current relay state

### Rule 2: Make the Local Function Independent of All Network Paths

No network call, no cloud acknowledgment, no server authentication may be required for the local function. Test: unplug the router. Does the alarm still sound? Does the fob still work? If yes — correct. If no — the design has a network dependency that must be eliminated.

### Rule 3: Remote Access is a Layer On Top

Design the remote access capability as an addition to the already-working local system. The device is complete and functional without remote access. Remote access adds capability; its absence must not subtract basic function.

### Rule 4: Define the Fail-Safe State Explicitly

What is the correct state when the network fails while remote operation is in progress?
- Aircraft: immediate revert to local pilot control
- Smart plug: maintain current relay state (not on, not off — last state)
- Smoke detector: alarm continues sounding; push notification not delivered
- Car fob: already independent; no state change needed

The fail-safe state should be the **safe** state, not the **last commanded** state:
- Aircraft: the onboard pilot is the safe controller; override revert is safe
- Smart plug: last relay state is safe for typical loads

### Rule 5: The DoS Attack Test

An attacker can always attempt to make the cloud service unavailable. Ask: "If the server is taken down completely, what happens to the physical device?" If the answer is "it fails to perform its primary safety function," the design fails availability.

---

## Part 7 — Availability vs Confidentiality Trade-Off

In some cases, increasing availability conflicts with security:

**Case 34 (smoke detector)**: the detector could connect to multiple cloud services in parallel (higher availability). But connecting to more services increases attack surface — more services that could be compromised to gain access to the silence command. Single trusted cloud service with high availability (ch1 p.42) is chosen over multi-cloud.

**Case 37 (aircraft)**: redundant communication channels (satellite + VHF) increase availability. They do not compromise confidentiality — both channels use TLS 1.3 encryption and mutual authentication. Redundancy for availability is compatible with strong security.

**Case 36 (electronic stamp)**: the central used-stamp database (for serial number uniqueness checking) is a single point of availability failure. If the database is down, offline scanners cannot perform serial number checks — they still accept stamps (cryptographically valid) but cannot detect copies. High-availability database architecture (replication, failover) is required. This is a fundamental tension: availability of the anti-copy mechanism conflicts with the distributed nature of postal scanning.

---

## Slide References

- IS_UG_1_Introduction (p.42: availability as a security service)
- IS_UG_3_7_Appl_System (p.46: minimise attack surface; graceful degradation)
- IS_UG_3_6_Appl_TLS (p.37: link-level availability via redundant channels)
