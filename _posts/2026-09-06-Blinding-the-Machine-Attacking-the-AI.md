---
layout: post
title: "Blinding the Machine: Attacking the AI Perception Stack in Autonomous Robots"
date: 2026-09-06
---

# Blinding the Machine: Attacking the AI Perception Stack in Autonomous Robots

*ROS2Reaper series — Phase: Edge AI & Perception*
*by @gh057x*

Every phase of ROS2Reaper up to this point has been about getting a foothold in the robot's nervous system: sitting on the DDS graph, subverting security plugins, injecting into control topics, standing up covert C2 over the same middleware the robot trusts. Useful, but ultimately those are means to an end. The end — the thing that actually decides whether a robot drives into a wall, hands a payload to the wrong person, or reports "all clear" while it plows through a restricted zone — lives higher up the stack, in the perception and decision layer.

That's the AI section. And it's where the most interesting, most durable, and least understood attacks live.

This post lays out how I think about the AI/perception attack surface on modern autonomous mobile robots and legged platforms, the classes of technique ROS2Reaper exercises there, and how they map back to MITRE ATT&CK for ICS. It's deliberately vendor-neutral — the point is the pattern, not any one platform. If you're building or defending these systems, the patterns are what you need anyway.

## Why perception is the crown jewel

A robot's autonomy stack is a pipeline: sensors produce raw observations, perception models turn those into a semantic understanding of the world (obstacles, free space, objects, poses), a localization and mapping layer places the robot in that world, and a planning/control layer turns all of it into motion. Compromise any earlier stage and every downstream stage inherits your lie — but does so *legitimately*, executing its normal logic on corrupted inputs.

That's the whole game. If I inject a control command directly, I'm fighting the robot's safety logic, its watchdogs, and anyone watching the control topics. If I corrupt what the robot *believes about the world*, the robot's own safety logic becomes my enforcement mechanism. It will confidently, correctly, and observably do the wrong thing, because from its point of view it's doing exactly the right thing given what it "sees." There's no exception thrown, no fault raised, no obvious tamper signature. The system is behaving as designed on inputs that are a fabrication.

Perception attacks are also durable. A patched authentication bug is gone. A model that misclassifies a crafted pattern, a pipeline that trusts any well-formed message on a sensor topic, a fusion stage with no cross-sensor sanity check — those are architectural properties. They don't get closed by a point fix; they get closed by redesign, which is slow and rare.

## Threat model

For this phase I assume the attacker has already achieved what the earlier ROS2Reaper phases establish: a position on the robot's internal graph, or on the segment between an MCU-class sensor coprocessor and the main compute node, or physical proximity to the sensors themselves. In practice, open-by-default middleware configurations make the first of these far easier than it should be — a discovery-enabled data bus with no meaningful authentication is a gift.

From that position, the AI section asks a different question than the rest of the framework: not "can I run a command," but "can I control what the robot believes is true?" Everything below is a way to answer yes.

## The attack classes

### 1. Sensor-level spoofing and injection

The earliest point in the pipeline is the raw observation. There are two flavors:

- **Physical / environmental spoofing** — manipulating the real-world stimulus a sensor receives: crafted patterns in the visual field, retroreflective or absorptive material that distorts depth and ranging returns, structured light or acoustic interference against active sensors. No access to the robot required, which makes it the hardest to attribute and, often, the hardest to detect.
- **On-graph observation injection** — from a foothold on the data bus, publishing or overwriting sensor messages directly. If a perception node subscribes to a sensor topic and there's no origin authentication or integrity check on that topic (the common case on permissive middleware setups), a well-formed fake observation is indistinguishable from a real one. Camera frame takeover and range-data forgery both live here.

The combination is what matters: physical spoofing to shape the *content* of the lie, graph injection to guarantee its *delivery*.

### 2. Adversarial perturbation of perception models

Once observations reach the models, the models themselves are attackable. Learned perception — object detection, segmentation, classification, depth estimation — carries the well-documented fragility of neural networks: small, structured perturbations that a human wouldn't notice can flip a model's output. In a robotics context this is not an academic curiosity. Making an obstacle read as free space, a person read as background, or a "stop" cue read as "proceed" has immediate kinetic consequences.

What's specific to the robot setting is that perturbations have to survive the real pipeline — sensor noise, varying distance and angle, the fusion stage, temporal smoothing. Techniques that account for that (transformation-robust, physically-realizable perturbations) are the ones ROS2Reaper cares about, because a lab-only adversarial example that dies in the field isn't a TTP, it's a demo.

### 3. Perception-pipeline and message manipulation

Between the model outputs and the planner sits a lot of glue: intermediate topics carrying detections, costmaps, point clouds, transforms, and pose estimates. Each of those is another injection or tampering point. You don't always need to fool a model if you can simply rewrite its output before the planner reads it — publish a fabricated obstacle map, drop detections, or shift the transform tree so the robot's sense of *where it is* silently drifts. Localization and mapping are especially high-value here: corrupt the pose or the map and every subsequent decision is wrong in a coordinated, coherent way that looks entirely normal.

### 4. Model and data supply chain

The model didn't appear from nowhere, and neither did its training data. That opens two slower but heavier attacks:

- **Data / model poisoning** — influencing the model during training or fine-tuning so it carries a latent behavior: a trigger pattern that reliably causes a chosen misclassification, dormant until the attacker presents it in the field. This is a backdoor with a physical activation key.
- **Model extraction and analysis** — pulling the deployed model off the edge device (or reconstructing its behavior through queries) to study it offline, craft reliable adversarial inputs against the exact weights in use, and identify blind spots. Edge inference means the model is *on the robot* — recoverable by anyone who reaches the filesystem or the accelerator, which the earlier phases increasingly do.

### 5. Downstream decision effects

None of the above is the goal in itself; the goal is what the planning and control layer does with the corrupted belief. A poisoned costmap becomes a path through a forbidden zone. A dropped detection becomes a collision. A drifted pose becomes a delivery to the wrong location that the robot logs as successful. This is where a perception lie converts into physical impact — and, importantly, into *false telemetry*, because the robot reports the world as it (mis)perceived it.

## Mapping to ATT&CK for ICS

ROS2Reaper frames all of this against MITRE ATT&CK for ICS rather than chasing CVEs, because these are architectural weaknesses, not discrete software defects — they don't fit the CVE model, and TTP framing is far more useful to both attackers reasoning about kill chains and defenders reasoning about coverage. The AI section lands most naturally on:

- **Spoof Reporting Message** — the robot's own telemetry reflects the fabricated world state; the operator's picture is wrong by design.
- **Manipulation of View** — corrupting what the system (and by extension the operator) perceives about the process, here quite literally the robot's perception of its environment.
- **Manipulation of Control** — using corrupted perception to steer physical behavior without ever touching a control command directly.
- **Impair Process Control / Loss of Safety** — safety logic operating faithfully on false inputs, producing unsafe motion while raising no fault.

The through-line: these techniques achieve control-layer impact *from the perception layer*, which is exactly why they slip past defenses that watch commands and controllers rather than beliefs.

## Notes for defenders

Because the AI section targets architecture, the defenses are architectural too, and worth stating plainly for the folks who have to hold these systems:

- **Authenticate the data bus.** Origin authentication and integrity on sensor and perception topics collapses the on-graph injection surface. Permissive, discovery-open middleware is the single biggest force multiplier for everything above.
- **Cross-check sensors.** Fusion stages that sanity-check modalities against each other (does the depth return agree with the image, does the new pose agree with odometry) break single-sensor lies.
- **Bound belief, not just commands.** Watchdogs that validate *perceptual plausibility* — sudden map changes, physically impossible pose jumps, detections that wink in and out — catch attacks that command-level monitoring never sees.
- **Protect the model at rest.** Treat the on-device model and its weights as sensitive material; extraction is the precursor to reliable adversarial input.
- **Assume the training pipeline is in scope.** Provenance and validation of data and model artifacts is the only thing standing between you and a backdoor with a physical trigger.

## Where this goes next

The AI section is the part of ROS2Reaper I'm still actively building out, and it's the centerpiece of where the research is heading: validating these techniques in simulation against a full autonomy stack, then confirming they survive the jump to physical testbeds. Perception is where robotics security stops being about protocols and starts being about *trust in a machine's model of reality* — and that's a much harder problem to patch than a bad auth check.

If you're working the same corner of the field — offensive or defensive — I'd like to compare notes. This is a surface that rewards a community, and it's badly under-explored relative to how much of the world is about to be driven by these systems.

*— @gh057x*
