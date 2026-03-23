---
layout: post
title: "Building an Offensive Security Toolkit for ROS 2 / DDS Environments"
date: 2026-03-23
---

# ROS2Reaper: Building an Offensive Security Toolkit for ROS 2 / DDS Environments

**Purple Team // Robotics Security Research // ROS 2 · DDS · ICS/OT Gh057x**

> **⚠️ Responsible Disclosure:** This research and the tools described are intended for authorized security assessments only. All testing was conducted in isolated lab environments. Any vulnerabilities discovered during this research were reported through proper disclosure channels.
 
---
 
## The Gap in Robotics Security Tooling
 
ROS 2 is no longer just for academic robotics. It now powers autonomous mobile robots (AMRs) in warehouses, surgical assistance systems, autonomous vehicles, and industrial automation infrastructure. The attack surface is real — and largely unexplored from an offensive security perspective.
 
The old guard of ROS security tooling — namely **ROSPenTo** — was built for ROS 1. It has no equivalent for ROS 2's completely redesigned communication layer. When I started looking at ROS 2 environments during engagements, the gap was immediately obvious: there's no pentest framework that operates at the RTPS wire protocol layer against modern DDS middleware.
 
That's what I set out to fix with **ROS2Reaper**.
 
---
 
## What Makes ROS 2 Interesting (and Vulnerable)
 
ROS 2 replaced the original ROS master/slave architecture with a publish-subscribe middleware called **DDS (Data Distribution Service)**. All communication happens over **RTPS (Real-Time Publish-Subscribe Protocol)** — an OASIS standard implemented by vendors like eProsima (Fast DDS), Eclipse (Cyclone DDS), and RTI (Connext DDS).
 
The default security posture of a stock ROS 2 deployment is effectively zero:
 
- **Unauthenticated SPDP discovery** — any node on the subnet can discover all participants via multicast
- **No publisher authorization** — any participant can publish to any topic
- **Flat DDS domain** — the entire DDS domain is a broadcast domain; there's no per-topic ACL
- **RTPS reachable over UDP** — accessible from anywhere on the local network
 
```bash
# On an unprotected ROS 2 deployment, this is all you need:
$ ros2 topic list
/robot1/cmd_vel          # velocity control — open to anyone
/robot1/scan             # LIDAR sensor feed — open to anyone
/robot2/navigate_to_pose # navigation goals — open to anyone
```
 
This means any participant on the subnet can silently read all sensor data, inject velocity commands, spoof navigation goals, or impersonate legitimate nodes — all without authentication.
 
---
 
## ROS2Reaper: Framework Overview
 
ROS2Reaper is a modular Python offensive security toolkit that operates against ROS 2 environments at two distinct layers:
 
**Layer 1 — DDS/RTPS Protocol Layer**
 
Works at the wire level against the underlying DDS middleware. This layer is DDS-vendor agnostic and works regardless of whether the target is running ROS 2 or just bare DDS.
 
**Layer 2 — ROS 2 Application Layer**
 
Works at the application level: enumerating ROS 2 topics, services, and actions; auditing SROS2 security configurations; and executing robotics-specific attack primitives like cmd_vel injection and node impersonation.
 
### Project Structure
 
```
ros2reaper/
├── ros2reaper.py              # Unified CLI entry point
├── core/
│   ├── rtps_scanner.py        # RTPS/DDS participant discovery engine
│   ├── rtps_parser.py         # Protocol dissector (Scapy-based)
│   └── config.py
├── modules/
│   ├── discovery.py           # DDS participant & endpoint enumeration
│   ├── fingerprint.py         # 7-layer DDS vendor fingerprinting
│   ├── topic_enum.py          # ROS 2 topic/service/action enum via SEDP
│   ├── sros2_audit.py         # SROS2 security configuration auditor
│   ├── topic_injection.py     # ★ Phase 2 — topic injection attacks
│   ├── node_impersonation.py  # ★ Phase 2 — rogue node creation
│   └── amplification.py       # ★ Phase 2 — RTPS amplification PoC
└── lab/
    ├── docker-compose.yml
    └── scripts/lab_walkthrough.sh
```
 
---
 
## Phase 1: Reconnaissance & Enumeration
 
### RTPS Scanner
 
The core recon engine implements three scan modes:
 
- **Active** — Sends SPDP multicast probes to the standard DDS discovery addresses and listens for participant announcements
- **Passive/Stealth** — Pure traffic capture; no probes sent; useful for avoiding detection in sensitive environments
- **Port scan** — Uses DDS's deterministic port formula (`7400 + 250 * domainId`) to fingerprint participants across domain IDs
 
```python
# RTPS participant discovery — active probe
$ ros2reaper.py --mode discover --interface eth0 --domain 0
 
[*] Sending SPDP multicast probe to 239.255.0.1:7400
[+] Participant discovered: 172.20.0.10:7410
    GUID prefix: 01.0f.be.ef.00.00.00.01.00.00.00.00
    Vendor: eProsima Fast DDS v2.12.1
[+] Participant discovered: 172.20.0.11:7410
[+] Participant discovered: 172.20.0.12:7660  (domain 5 — isolated)
```
 
### 7-Layer DDS Fingerprinting
 
The `fingerprint.py` module runs a 7-layer analysis pipeline against discovered participants:
 
1. **Vendor ID resolution** — maps GUID vendor bytes to known DDS implementations
2. **Property analysis** — extracts participant properties from PID_PROPERTY_LIST
3. **Participant name patterns** — identifies ROS 2 naming conventions vs. raw DDS
4. **Security posture** — checks for DDS-Security builtin plugins (`AUTH_PKI_DH`, `CRYPTO_AES_GCM_GMAC`)
5. **Builtin endpoint flags** — determines subscriber/publisher capability sets
6. **Locator exposure** — flags internal IP address leakage in unicast locators
7. **GUID pattern analysis** — identifies participant creation patterns
 
```bash
$ ros2reaper.py --mode fingerprint --target 172.20.0.10
 
[+] Target: 172.20.0.10
    DDS Vendor:       eProsima Fast DDS v2.12.1
    RMW Layer:        rmw_fastrtps_cpp
    ROS 2 Detected:   YES (Humble)
    Security Plugins: NONE
    Risk Level:       HIGH
    CVE Exposure:     CVE-2023-XXXX (Fast DDS PDP buffer overflow pattern detected)
    Internal IPs:     172.20.0.10:7410 leaked in locator list
```
 
### Topic Enumeration via SEDP
 
Rather than querying ROS 2's built-in `ros2 topic list` (which requires being a ROS 2 participant yourself), `topic_enum.py` passively sniffs SEDP (Simple Endpoint Discovery Protocol) traffic to reconstruct the full topic graph — including security-critical topics flagged automatically:
 
```
Discovered topics:
  /robot1/cmd_vel          [geometry_msgs/Twist]   ★ CRITICAL — mobility control
  /robot1/scan             [sensor_msgs/LaserScan] ★ HIGH — sensor integrity
  /robot1/odom             [nav_msgs/Odometry]
  /robot1/battery_state    [sensor_msgs/BatteryState]
  /robot2/navigate_to_pose [nav_msgs/NavigateToPose] ★ CRITICAL — navigation
  /robot2/joint_states     [sensor_msgs/JointState] ★ HIGH — arm control
```
 
### SROS2 Auditor
 
The `sros2_audit.py` module runs 8 checks against the target's security configuration:
 
| Check | Description |
|---|---|
| Security plugin enabled? | Is `DDS:Auth:PKI-DH` loaded at all? |
| Encrypt vs sign-only | Sign-only is insufficient — data still readable |
| Metadata exposure | Participant properties leaking sensitive info |
| Internal IP leakage | Locator lists advertising internal addresses |
| Known CVE patterns | Fast DDS and Cyclone DDS known-bad config patterns |
| Default config detection | Unmodified security XML templates in use |
| Domain isolation | Cross-domain topic leakage via non-isolated bridges |
| Security token completeness | Missing or malformed identity/permission tokens |
 
---
 
## Phase 2: Exploitation Modules
 
### Topic Injection
 
`topic_injection.py` publishes arbitrary messages to any ROS 2 topic. Pre-built payloads cover the most impactful attack primitives:
 
```bash
# Emergency stop — zero velocity to all joints
$ python3 modules/topic_injection.py \
    --topic /robot1/cmd_vel \
    --payload emergency_stop \
    --domain 0
 
# Velocity override — full speed in a direction
$ python3 modules/topic_injection.py \
    --topic /robot1/cmd_vel \
    --payload "linear.x=2.0,angular.z=0.0" \
    --rate 10
 
# LIDAR blinding — inject fake obstacle at 180 degrees
$ python3 modules/topic_injection.py \
    --topic /robot1/scan \
    --payload "fake_wall:180deg" \
    --rate 10
```
 
In a live demo against an unprotected Nav2 stack, the fake LaserScan injection reliably causes the navigation planner to recompute around a non-existent obstacle, effectively blocking robot movement or forcing it into a reroute that an attacker can guide.
 
### Node Impersonation
 
`node_impersonation.py` spawns a rogue DDS participant that masquerades as a legitimate ROS 2 node, enabling man-in-the-middle attacks on the publisher/subscriber chain:
 
```bash
# Become /navigation_stack and intercept nav goals
$ python3 modules/node_impersonation.py \
    --spoof /navigation_stack \
    --intercept \
    --forward-modified
```
 
Once impersonating a node, the attacker can selectively drop, replay, or modify messages in transit — for example, intercepting `navigate_to_pose` goal messages and silently substituting a different destination.
 
### RTPS Amplification PoC
 
`amplification.py` tests DDS deployments for RTPS reflection and amplification characteristics. Unprotected DDS participants will respond to forged SPDP probes with participant data responses — useful for measuring amplification ratios and demonstrating bandwidth exhaustion potential.
 
---
 
## The Docker Lab Environment
 
All Phase 1 and Phase 2 development and testing was done against a local Docker lab with macvlan networking on Proxmox, simulating a real LAN environment (no Docker bridge NAT — actual L2 adjacency).
 
### Lab Architecture
 
| Container | IP | Role | DDS Domain | Security |
|---|---|---|---|---|
| `robot1` | 172.20.0.10 | Victim AMR | 0 | None |
| `robot2` | 172.20.0.11 | Victim AMR | 0 | None |
| `robot3_secured` | 172.20.0.12 | SROS2 hardened | 5 | SROS2 Enforce |
| `attacker` | 172.20.0.100 | ROS2Reaper | 0 | N/A |
 
`robot3_secured` runs in DDS domain 5 with SROS2 in Enforce mode and auto-generated keystores, providing a comparison target to validate that the Phase 1 recon modules correctly identify and respect security boundaries.
 
The `fake_robot.py` node publishes realistic AGV-style topics — `cmd_vel`, `LaserScan`, `Odometry`, `BatteryState`, `JointState`, and TF transforms — including a canary that logs when suspicious velocity commands are received.
 
```bash
# Spin up the full lab
$ cd ros2reaper/lab && docker compose up -d
 
# Shell into the attacker container
$ docker compose exec attacker bash
 
# Run the full Phase 1 → Phase 2 walkthrough
$ bash /lab/scripts/lab_walkthrough.sh
```
 
---
 
## Full Attack Chain
 
A complete end-to-end attack against an unprotected ROS 2 deployment flows like this:
 
```
[1] RTPS multicast probe
        ↓
[2] Enumerate DDS participants + GUID extraction
        ↓
[3] SEDP passive listen → full topic/service/action map
        ↓
[4] 7-layer fingerprint → vendor, RMW, SROS2 posture, CVE exposure
        ↓
[5] Identify critical topics (cmd_vel, nav goals, sensor feeds)
        ↓
[6] Topic injection / LIDAR blinding / emergency stop
        ↓
[7] Node impersonation → persistent MitM on nav goal channel
```
 
The entire recon phase (steps 1–4) can be run in stealth/passive mode with zero packets sent from the attacker — pure traffic observation.
 
---
 
## Defensive Takeaways
 
ROS2Reaper was built to expose these weaknesses, not just exploit them. Here's what the tool reveals about your defensive posture:
 
**Enable SROS2 — in Enforce mode, not permissive.** Permissive mode logs violations but doesn't block them. Enforce mode actually rejects unauthenticated participants.
 
**Sign AND encrypt.** Sign-only SROS2 configurations authenticate publishers but leave message content readable. Use `AES-GCM-GMAC` encryption for topics carrying sensitive data.
 
**Segment DDS domains by trust zone.** Don't run safety-critical control topics on the same DDS domain as sensor feeds and diagnostics. Use domain-bridging only where strictly necessary, with proper ACLs.
 
**Restrict multicast to the robot VLAN.** SPDP multicast should never cross network boundaries. VLAN segmentation is your first line of defense even before SROS2.
 
**Monitor for anomalous DDS participants.** A participant with an unknown GUID prefix appearing on the domain is a red flag. ROS IDS with ML-based anomaly detection (Isolation Forest on topic timing/rate/payload distributions) can surface this.
 
---
 
## Roadmap
 
**Phase 1 — Complete**
- RTPS parser + scanner (active, passive, stealth modes)
- 7-layer DDS fingerprinting
- Topic/service/action enumeration via SEDP
- SROS2 auditor (8 checks)
- Docker lab environment
 
**Phase 2 — Complete**
- Topic injection attacks (cmd_vel, LaserScan, nav goals)
- Node impersonation + MitM
- RTPS amplification PoC
- Multi-robot lab with SROS2 comparison target
 
**Phase 3 — In Progress**
- Custom DDS fuzzing harness (building on DDSFuzz patterns) for 0-day research in Fast DDS and Cyclone DDS
- ICS/OT DDS bridge attack research — targeting DDS deployments in non-robotics industrial control environments
- ROS IDS evasion techniques
- Second conference track: OT/ICS focus
 
---
 
## Conclusion
 
The ROS 2 attack surface is real, under-researched, and increasingly deployed in safety-critical environments. The tooling gap that ROSPenTo's obsolescence left hasn't been filled — ROS2Reaper is an attempt to fix that.
 
Phase 1 and Phase 2 are complete. The framework is modular, extensible, and designed to grow into Phase 3's 0-day research and ICS/OT extension. Conference submissions for DEF CON ICS Village and Black Hat Arsenal are in progress.
 
If you're doing robotics security work, drop me a line.
 
---
 
*This research will be presented at different CON(s) ICS Villages and submitted to Black Hat Arsenal (Hopefully :)). All Phase 2 PoC modules are designed for authorized use only — see the repository's legal notice for terms.*
 
*— Gh057x*
