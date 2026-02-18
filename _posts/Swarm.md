---
layout: post
title: "Swarm: Weaponizing ROS 1's Trust Model"
date: 2026-02-18
---

# Ghost in the Swarm: Weaponizing ROS 1's Trust Model to Hijack Robot Fleets

**Purple Team // Robotics Security Research // Gh057x**

---

## TL;DR

ROS 1 (Robot Operating System) was designed for research — not production, not adversarial environments, and definitely not the real world it now operates in. Its publish-subscribe architecture has **zero authentication, zero encryption, and zero access control** by default. We demonstrate how an attacker on the same network segment can discover, enumerate, and simultaneously hijack an entire fleet of autonomous robots, forcing them to converge on a single physical coordinate — a technique we call **Swarm Convergence Attack (SCA)**.

The result? Robots collide. Hardware breaks. Operations halt. And nobody had to exploit a single CVE to do it — because the architecture never assumed an adversary existed.

---

## 0x00 — Background: Why ROS 1 Is Everywhere (and Why That's a Problem)

ROS 1 was born at Willow Garage in 2007. It was a research framework — a way for academics to stop reinventing the wheel (literally) and share reusable robotics code. It succeeded wildly. Today, ROS powers:

- Warehouse logistics fleets (think automated fulfillment centers)
- Agricultural robots
- Hospital delivery bots
- Industrial inspection drones
- Research platforms at every major university

The problem? **ROS 1 was never designed with security in mind.** Quoting the official ROS wiki:

> *"ROS is not a secure communication framework. It was designed for collaborative research environments."*

Yet here we are — ROS 1 nodes running in production, on networks that are "air-gapped" (they aren't), behind firewalls that "block everything" (they don't), managed by teams that assume physical isolation equals security (it doesn't).

### The Architecture in 30 Seconds

```
┌──────────────┐       ┌─────────────┐       ┌──────────────┐
│   Robot A    │       │  ROS Master │       │   Robot B    │
│  (Node)      │◄─────►│  (Registry) │◄─────►│  (Node)      │
│              │       │  Port 11311 │       │              │
└──────┬───────┘       └─────────────┘       └──────┬───────┘
       │                                            │
       │         Direct TCP (TCPROS)                │
       └────────────────────────────────────────────┘
                  No auth. No encryption.
                  Just vibes.
```

- **ROS Master**: XML-RPC registry on port 11311. Any node can register. Any node can query.
- **Topics**: Named pub-sub channels. `/cmd_vel` for velocity. `/move_base/goal` for navigation targets. Anyone can publish. Anyone can subscribe.
- **No identity model**: There is no concept of "who" a node is. If you can reach the Master, you're trusted.

---

## 0x01 — Threat Model: The Swarm Convergence Attack

### Attack Premise

An attacker with network access to a ROS 1 fleet can:

1. **Discover** all ROS Master instances on the network
2. **Enumerate** every robot's topics, services, and parameters
3. **Inject** malicious navigation goals to every robot simultaneously
4. **Force convergence** — all robots navigate to a single coordinate
5. **Result**: Physical collisions, hardware damage, operational denial-of-service

### Why Convergence?

Traditional DoS kills a service. Swarm Convergence is a **kinetic** denial-of-service — the robots become the weapon against themselves. It's:

- **Harder to detect**: Each robot receives what appears to be a valid `move_base` goal
- **Self-amplifying**: More robots = more damage at the convergence point
- **Physically destructive**: Unlike a network DoS, you can't just restart the service

### Attack Surface Diagram

```
  ATTACKER (same network segment)
      │
      ├── 1. RECON ──────────► nmap scan :11311 (ROS Master discovery)
      │
      ├── 2. ENUMERATE ──────► XML-RPC queries to each Master
      │                        - Published topics
      │                        - Subscribed topics
      │                        - Active nodes
      │                        - Parameter server dump
      │
      ├── 3. HIJACK ─────────► Publish to /move_base/goal on each robot
      │                        - Inject MoveBaseActionGoal messages
      │                        - Override legitimate navigation
      │                        - All goals point to single coordinate
      │
      └── 4. SUSTAIN ────────► Continuous goal injection at 10Hz
                               - Re-inject if robot's planner reroutes
                               - Cancel legitimate goals via /move_base/cancel
```

---

## 0x02 — Phase 1: Reconnaissance

### Finding the Robots

ROS Master runs an XML-RPC server on port **11311** by default. It's not optional — it's how the entire framework discovers nodes and topics. Scanning for it is trivial:

```bash
nmap -p 11311 --open -T4 192.168.1.0/24
```

Every host with 11311 open is running a ROS Master. In a fleet deployment, you'll typically find:

- One Master per robot (common in multi-robot setups)
- Or one central Master with all robots as nodes (centralized architecture — even easier to attack)

### What the Master Will Tell You (For Free)

Once you find a Master, you don't need credentials. The XML-RPC API is fully open:

```python
import xmlrpc.client

master = xmlrpc.client.ServerProxy('http://192.168.1.42:11311')

# Get every published topic
code, msg, topics = master.getPublishedTopics('/attacker', '')
# Returns: [['/cmd_vel', 'geometry_msgs/Twist'], ['/odom', 'nav_msgs/Odometry'], ...]

# Get every registered node
code, msg, state = master.getSystemState('/attacker')
# Returns: publishers, subscribers, services — the complete topology
```

No tokens. No keys. No questions asked.

---

## 0x03 — Phase 2: The Attack

### Target: `/move_base/goal`

The ROS navigation stack (`move_base`) is the standard way robots navigate. It accepts `MoveBaseActionGoal` messages on the `/move_base/goal` topic. The message contains:

- A target pose (x, y, z coordinates + orientation)
- A frame reference (usually `"map"`)
- A goal ID (arbitrary string)

**There is no validation of who publishes this message.** If you can reach the topic, the robot will navigate to your coordinates.

### The Injection

```python
goal = MoveBaseActionGoal()
goal.goal.target_pose.header.frame_id = "map"
goal.goal.target_pose.pose.position.x = 10.0  # convergence X
goal.goal.target_pose.pose.position.y = 5.0   # convergence Y
goal.goal.target_pose.pose.orientation.w = 1.0

publisher.publish(goal)  # Robot now drives to (10, 5)
```

Multiply this across every robot on the network. Inject at 10Hz to override any legitimate replanning. Every robot in the fleet now has a single destination.

### Alternative: Chaos Mode (`/cmd_vel` Flooding)

For a less surgical, more chaotic effect, flood the `/cmd_vel` (velocity command) topic:

```python
twist = Twist()
twist.linear.x = 0.5   # forward
twist.angular.z = 2.0   # hard spin

# Publish at 100Hz — overwhelms legitimate velocity commands
```

This causes robots to spin erratically. Less targeted than convergence, but useful for demonstrating the total lack of input validation.

---

## 0x04 — Phase 3: Impact Analysis

### What Happens at the Convergence Point

In our simulation environment (Gazebo, closed-loop, 12 TurtleBot3 instances):

| Metric | Result |
|--------|--------|
| Time to full convergence | ~45 seconds (depending on fleet spread) |
| Collision events | 37 (in a single 2-minute run) |
| Robots immobilized post-collision | 9 of 12 |
| Legitimate nav goals processed | 0 (completely overridden) |
| Alerts generated by default ROS tools | 0 |

**Zero alerts.** The robots faithfully executed every injected goal. `move_base` has no concept of "this goal came from an unauthorized source." `rostopic echo` would show the malicious goals if someone was watching — but there's no built-in anomaly detection, no audit log, and no authentication layer to flag the intrusion.

### The Real-World Implications

In production fleet deployments, this attack could cause:

- **Warehouse disruption**: Robots collide, block aisles, damage inventory
- **Safety hazards**: Humans working alongside robots are at risk
- **Hardware destruction**: Repeated collisions damage sensors, actuators, chassis
- **Operational downtime**: Fleet recovery requires manual intervention per robot

---

## 0x05 — Why This Works (Root Causes)

| Root Cause | Description |
|------------|-------------|
| **No authentication**  | Any network participant can register as a node |
| **No message signing** | Published messages have no origin verification |
| **No encryption**      | All traffic is plaintext TCPROS — trivially sniffable and injectable |
| **No access control**  | Any node can publish to any topic |
| **No rate limiting**   | A single attacker can flood at arbitrary rates |
| **Implicit trust model** | ROS 1 assumes all participants are cooperative |

This isn't a bug. It's the architecture. ROS 1 was designed this way intentionally — for a use case (academic research) where these properties were acceptable. The failure is deploying it in environments where they aren't.

---

## 0x06 — Mitigations & Defenses

### Short-Term (ROS 1 Hardening)

| Mitigation | Effectiveness | Difficulty |
|------------|--------------|------------|
| **Network segmentation** | High | Medium — isolate ROS traffic on a dedicated VLAN |
| **SROS (Secure ROS)** | Medium | High — adds TLS + x.509 auth, but limited adoption |
| **Firewall rules** | Medium | Low — restrict port 11311 access to known IPs |
| **Topic whitelisting** | Medium | Medium — custom node to reject unauthorized publishers |
| **VPN tunneling** | Medium | Medium — encrypt ROS traffic, limit network access |
| **Anomaly detection** | Low-Medium | High — monitor for unusual publishing patterns |

### Long-Term (Migrate to ROS 2)

ROS 2 addresses many of these issues through its DDS (Data Distribution Service) transport layer:

- **SROS2**: Built-in security using DDS Security specification
- **Authentication**: X.509 certificate-based node identity
- **Access control**: Per-topic publish/subscribe permissions
- **Encryption**: TLS for all inter-node communication
- **Governance files**: Declarative security policies

**Migration is the real fix.** ROS 1 Noetic reaches end-of-life in May 2025. If you're still running ROS 1 in production — this talk is your wake-up call.

### Detection Signatures

For blue teams monitoring ROS 1 networks:

```
# Suspicious indicators:
- New node registrations from unknown IPs
- High-frequency publishing to /cmd_vel or /move_base/goal
- Multiple /move_base/cancel messages (goal preemption)
- Simultaneous goal injection across multiple robots
- XML-RPC enumeration requests from non-robot IPs
```

---

## 0x07 — Responsible Disclosure & Ethics

This research was conducted entirely in **closed-loop simulation environments** (Gazebo + ROS Noetic on isolated networks). No production systems, public networks, or physical robots were targeted.

The vulnerabilities described are **known and documented** by the ROS maintainers. ROS 1's lack of security is not a secret — it's a design decision with well-understood tradeoffs. The purpose of this talk is to:

1. **Raise awareness** among teams deploying ROS 1 in production
2. **Demonstrate concrete impact** beyond theoretical risk
3. **Accelerate migration** to ROS 2 with SROS2 security enabled
4. **Provide actionable mitigations** for teams that can't migrate immediately

If you are running ROS 1 in a production environment: **assume your network is hostile, because it is.**

---

## 0x08 — Tools & References

### Tools Used
- **Gazebo**: Physics simulation for multi-robot testing
- **ROS Noetic**: Target framework
- **nmap / python-nmap**: Network reconnaissance
- **Custom PoC tooling**: Swarm Convergence Attack framework (not publicly released)

### References

- ROS Wiki — Security: `wiki.ros.org/Security`
- SROS2 Documentation: `docs.ros.org/en/rolling/Concepts/About-Security.html`
- Dieber, B. et al. — *"Penetration Testing ROS"* (2020)
- McClean, J. et al. — *"A Preliminary Cyber-Physical Security Assessment of the Robot Operating System (ROS)"* (2013)
- Rivera, S. et al. — *"RosPenTo: ROS Penetration Testing Tool"*
- White, R. et al. — *"SROS: Securing ROS over the wire, in the graph, and through the kernel"* (2016)

---

## About the Author

**Gh057x** — Security researcher focused on the intersection of robotics, embedded systems, and adversarial machine learning. Believes that if your robot trusts everyone on the network, it trusts no one that matters.

---

*"The robots aren't rising up. They're being told to. And nobody's checking who's doing the telling."*

---

**License**: This writeup is provided for educational and defensive security purposes only. The techniques described should only be used in authorized testing environments. Unauthorized access to computer systems and robotic platforms is illegal under the CFAA and equivalent international laws.
