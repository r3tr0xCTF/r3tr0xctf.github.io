---
layout: post
title: "Hacking AVGs and AMRs"
date: 2026-02-10
---

Ghost in the Machine: Hijacking Industrial Robots with ROS 1

By: Gh057x  |  Cacklacky | Hacking Con(s) Community Research Series

Walk into any modern warehouse or "smart" factory, and you’ll see them: AMRs (Autonomous Mobile Robots). They’re heavy, they’re fast, and they’re carrying the literal weight of global logistics. But here’s the scary part—underneath that high-tech industrial shell, many of these bots are running a middleware called ROS 1 (Robot Operating System).

ROS 1 is over a decade old, and it was built for labs, not for the wild west of modern networks. In this post, we’re diving into how a complete lack of authentication lets us turn a 500kg robot into a physical projectile.

The Vulnerability: "Trust Me, Bro"

ROS 1 uses a "Master" node architecture. Think of the Master as a DNS server for the robot's internal organs. It lets different nodes (sensors, motors, logic) find each other and talk via "topics."

The catch? ROS 1 has zero native authentication. If you can hit the ROS Master (usually on port 11311), you are the robot. You can eavesdrop on its cameras, fake its LIDAR readings, and—worst of all—hijack the /cmd_vel topic. That’s the "steering wheel" stream.

The Attack: Taking the Wheel

We recently demoed a "Remote Kinetic Hijack" that’s surprisingly simple. Once you’re on the factory’s internal VLAN, you just point your own machine at the robot’s Master:

[export ROS_MASTER_URI=http://<ROBOT_IP>:11311]


You don't need a zero-day. You don't need to be a memory corruption wizard. You just run a Python script that floods malicious velocity commands at 100Hz. Since the robot’s actual controller is likely only talking at 10Hz, your attack stream wins the race every single time.

In our Gazebo + TurtleBot3 lab, the results were brutal. The bot ignores its sensors, spins out of control, and slams into the nearest wall. To make it even worse, we can flood the /scan topic with "all clear" data, effectively blinding the robot’s LIDAR so it thinks it’s in an open field while it’s actually about to wreck a forklift.

Detection: Hunting the Ghost

If you’re on the Purple Team side, you can catch this happening in real-time. If a bot starts acting possessed, use the robot's own toolkit to find the intruder:

Spot the Stranger: rosnode list shows all active nodes. Look for anything suspicious or nodes with random strings in their names.

Trace the Source: rostopic info /cmd_vel is your best friend. It lists the IP address of every machine sending movement commands. If you see an IP that isn't supposed to be there, you've found your ghost.

Check the Pulse: rostopic hz /cmd_vel lets you see how fast commands are coming in. Anything over 100Hz is a massive red flag for a flood attack.

How We Actually Fix This

We can't just stop using robots, so we have to stop trusting them blindly. Here’s the game plan:

Move to ROS 2: SROS2 is the answer. It uses DDS-Security to give us the encryption and authentication we should have had from day one.

Segment Everything: Your robots should live in their own "cells" behind a strict firewall. There is zero reason for a bot to be reachable from the main corporate Wi-Fi.

Hardware Safety: Never trust software with a human life. E-Stops and safety-rated LIDAR slowdowns should be hardwired directly to the motor controllers, bypassing the OS entirely.

Final Thought

The "Ghost in the Machine" isn't a sci-fi trope—it's what happens when we connect legacy protocols to the internet. As we bridge the gap between bits and atoms, security can't be an "extra" feature. It’s a safety requirement.

Ready to try it yourself? Check out my Demo Setup Guide to fire up the Gazebo lab, just ping me for the info :)
