# OpenClaw Gateway and ControlUI Continuity Report
**Project:** VISCERAL OpenClaw Deployment  
**Author:** ChatGPT (Engineering Continuity Documentation)  
**Date:** 2026-02-18 14:18:33 UTC  
**Gateway Host:** ros2-ubuntu (GCP instance, external IP 34.24.11.22)  
**Client Host:** Foster-MacBook  

---

# PURPOSE

This document provides a complete, precise, and exhaustive engineering continuity record of all actions performed to deploy, debug, and stabilize the OpenClaw gateway and ControlUI system. It is written so that a new engineer can assume control immediately with full situational awareness and zero ambiguity.

---

# SYSTEM ARCHITECTURE

## Physical machines

### Gateway Host (GCP VM)
Hostname: ros2-ubuntu  
External IP: 34.24.11.22  
Internal IP: 10.142.0.3  
OS: Ubuntu 22.04  
Role:
- Runs openclaw-gateway service
- Hosts ControlUI
- Hosts model runtime
- Hosts browser automation runtime

### Client Machine
Hostname: Foster-MacBook  
Role:
- Runs browser ControlUI
- Runs openclaw CLI
- SSH tunnel origin

---

# INITIAL FAILURE CONDITION

Primary symptom:

unauthorized: device token mismatch

Secondary symptoms:

- Multiple gateway instances running
- Gateway auto‑respawning
- Docker container conflicting with systemd service
- CLI unable to authenticate
- ControlUI unreachable remotely

---

# DISCOVERY PROCESS

## Port inspection

Command:

sudo lsof -nP -iTCP:18789 -sTCP:LISTEN

Result:

openclaw-gateway running under ubuntu user

This revealed rogue instance.

---

# PROCESS TREE ANALYSIS

pstree output:

systemd
  └ containerd-shim
      └ openclaw
          └ openclaw-gateway

Conclusion:

Gateway was being launched inside Docker.

---

# DOCKER CONTAINER DISCOVERY

Command:

sudo docker ps

Result:

ghcr.io/openclaw/openclaw:latest

Container name: openclaw

Restart policy:

unless-stopped

This caused persistent gateway conflict.

---

# DOCKER REMOVAL

Commands:

sudo docker stop openclaw  
sudo docker rm openclaw  

Verification:

PORT FREE

This permanently removed rogue gateway.

---

# SYSTEMD GATEWAY INITIALIZATION

Command:

systemctl --user enable --now openclaw-gateway.service

Verification:

sudo lsof -nP -iTCP:18789 -sTCP:LISTEN

Result:

openclaw-gateway running under foster user

Correct configuration achieved.

---

# DEVICE TOKEN RESET

Removed stale identity:

rm -rf ~/.openclaw/devices  
rm ~/.openclaw/identity/device.json  
rm ~/.openclaw/identity/device-auth.json  
rm -rf ~/.cache/openclaw  

This cleared invalid device state.

---

# CURRENT SYSTEM STATE

Gateway status:

Running
Systemd-managed
Port: 18789
User: foster
Docker conflict removed

Gateway endpoint:

ws://127.0.0.1:18789

ControlUI endpoint:

http://127.0.0.1:18789/

---

# REMAINING REQUIRED STEPS

## Step 1: Restore SSH tunnel

Run on local machine:

ssh -i ~/.ssh/google_compute_engine -N -L 18789:127.0.0.1:18789 ubuntu@34.24.11.22

---

## Step 2: Open ControlUI

Open browser:

http://localhost:18789/

---

## Step 3: Approve device

Run:

openclaw devices list

Approve request:

openclaw devices approve <requestId>

---

## Step 4: Verify gateway

Run:

openclaw gateway status

Expected:

Authorized
Running
Connected

---

# ROOT CAUSE SUMMARY

Root causes:

Docker gateway conflict  
Device identity mismatch  
SSH authentication failure  

---

# FINAL CORRECT CONFIGURATION

ONLY gateway running:

systemctl --user openclaw-gateway.service

NO Docker gateway present.

---

# SYSTEM STATUS

Gateway state: Stable  
Gateway ownership: Correct  
Remaining tasks: SSH tunnel and device approval  

---

# END OF REPORT
