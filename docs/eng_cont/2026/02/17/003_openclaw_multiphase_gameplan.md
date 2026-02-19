# OpenClaw Deployment -- Multi‑Phase Engineering Game Plan

**Project Context:** VISCERAL Infrastructure Setup\
**Author:** Engineering Continuity System\
**Date:** 2026-02-18 14:25 UTC

------------------------------------------------------------------------

# Executive Summary

This document defines the complete multi-phase recovery and
stabilization plan for the OpenClaw Gateway deployment on the
`ros2-ubuntu` GCP instance, including:

-   Root-cause analysis of the Docker vs systemd conflict
-   Device token mismatch resolution
-   SSH tunneling restoration
-   Control UI stabilization
-   Long-term hardened deployment plan
-   Production-grade VISCERAL alignment plan

It also clearly states **where we are right now** and what must happen
next.

------------------------------------------------------------------------

# Phase 0 -- Initial State (What Happened)

## 0.1 Installer Execution

We ran:

``` bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

This installed:

-   Node v22 (NodeSource)
-   npm user-local global install
-   OpenClaw v2026.2.15
-   systemd user service:
    \~/.config/systemd/user/openclaw-gateway.service

## 0.2 Unexpected Runtime Behavior

We discovered:

-   Port `18789` repeatedly occupied

-   Gateway failing to start with:

        unauthorized: device token mismatch

## 0.3 Root Cause

There were **two independent gateway instances running**:

1.  Docker container:

        ghcr.io/openclaw/openclaw:latest
        restart policy: unless-stopped

2.  systemd user service (foster)

The Docker container was auto-respawning and binding the same port.

This created: - Port conflicts - Identity/token conflicts - CLI device
authorization failures

------------------------------------------------------------------------

# Phase 1 -- Container Forensics and Removal

## 1.1 Container Discovery

We executed:

``` bash
sudo docker ps
```

Found container:

    ghcr.io/openclaw/openclaw:latest   openclaw

## 1.2 Restart Policy Confirmed

``` bash
sudo docker inspect -f '{.HostConfig.RestartPolicy.Name}' openclaw
```

Result:

    unless-stopped

## 1.3 Remediation

We executed:

``` bash
sudo docker stop openclaw
sudo docker rm openclaw
```

Port confirmed free afterward.

------------------------------------------------------------------------

# Phase 2 -- Systemd Gateway Stabilization

## 2.1 Enabled systemd Service

``` bash
systemctl --user enable --now openclaw-gateway.service
```

## 2.2 Confirmed Binding

``` bash
sudo lsof -nP -iTCP:18789 -sTCP:LISTEN
```

Confirmed:

    openclaw-gateway foster 127.0.0.1:18789

Gateway is now correctly running under:

-   user: foster
-   systemd user service
-   loopback bind
-   port: 18789

------------------------------------------------------------------------

# Phase 3 -- Device Token Mismatch Analysis

## 3.1 Observed Error

CLI attempts produce:

    gateway closed (1008): unauthorized: device token mismatch

## 3.2 Why This Happens

OpenClaw uses:

-   Gateway token (auth)
-   Device token (per-client trust identity)

The CLI has cached a device token that no longer matches the gateway's
identity store.

This can happen when:

-   Docker instance previously generated a device identity
-   systemd instance generated a new identity
-   CLI cache still holds old device credentials

------------------------------------------------------------------------

# Phase 4 -- Identity Reset Strategy

We removed:

    ~/.openclaw/devices
    ~/.openclaw/identity/device.json
    ~/.openclaw/identity/device-auth.json
    ~/.cache/openclaw

Goal: force clean device re-registration.

Current status: Gateway runs cleanly but CLI handshake still pending.

------------------------------------------------------------------------

# Phase 5 -- Control UI Access Strategy

Gateway binds to:

    127.0.0.1:18789

Since this is a GCP VM, access requires SSH tunneling.

Correct tunnel command (from Mac):

``` bash
ssh -i ~/.ssh/google_compute_engine     -N -L 18789:127.0.0.1:18789     foster@34.24.11.22
```

Then open locally:

    http://localhost:18789/

------------------------------------------------------------------------

# Phase 6 -- SSH Key Resolution

We observed:

    Permission denied (publickey)

This indicates:

-   Mac SSH key not authorized on GCP instance
-   Or wrong identity file selected

Resolution path:

1.  Use GCP console to inject key
2.  Or use `gcloud compute ssh`
3.  Confirm correct key in `~/.ssh/config`

------------------------------------------------------------------------

# Phase 7 -- Where We Are Now

Current state:

-   Docker gateway removed
-   systemd gateway running cleanly
-   Port 18789 stable
-   No competing processes
-   Gateway responding
-   CLI rejected due to device token mismatch
-   SSH tunnel not yet re-established

Infrastructure is now clean.

Identity layer needs re-pairing.

------------------------------------------------------------------------

# Phase 8 -- Immediate Next Steps

## Step 1 -- Recreate Device Identity

Stop gateway:

``` bash
systemctl --user stop openclaw-gateway.service
```

Remove identity folder completely:

``` bash
rm -rf ~/.openclaw/identity
```

Start gateway fresh:

``` bash
systemctl --user start openclaw-gateway.service
```

Then run:

``` bash
openclaw devices list
```

Approve new device.

------------------------------------------------------------------------

## Step 2 -- Restore SSH Tunnel

Verify key works:

``` bash
ssh -i ~/.ssh/google_compute_engine foster@34.24.11.22
```

Then create tunnel.

------------------------------------------------------------------------

## Step 3 -- Validate Control UI

Open:

    http://localhost:18789/#token=<gateway_token>

Confirm dashboard loads and agents visible.

------------------------------------------------------------------------

# Phase 9 -- Production Hardening Plan (VISCERAL Alignment)

## 9.1 Lock Deployment Model

Choose ONE deployment mode:

-   systemd user service (recommended)
-   OR containerized deployment
-   NEVER both

## 9.2 Pin Version

Avoid `latest` Docker tags. Pin version to prevent identity drift.

## 9.3 Backup Identity + Config

Backup:

    ~/.openclaw/openclaw.json
    ~/.openclaw/identity/

## 9.4 Establish Formal Startup Checklist

-   Confirm no container running
-   Confirm port free
-   Confirm service enabled
-   Confirm tunnel active
-   Confirm device authorized

------------------------------------------------------------------------

# Phase 10 -- Long-Term Architecture Strategy

For VISCERAL project integration:

-   Use dedicated GCP VM
-   Dedicated service account
-   Restrict firewall
-   Add monitoring
-   Log rotation for /tmp/openclaw
-   Optional: migrate to LAN bind + firewall rules

------------------------------------------------------------------------

# Strategic Position Assessment

We are no longer in a broken state.

We are in a controlled, clean infrastructure state with:

-   Known runtime
-   Known identity layer
-   Known gateway
-   Known failure mechanism

Remaining work is authentication reconciliation + tunnel restoration.

------------------------------------------------------------------------

# Final Assessment

Infrastructure Layer: Stabilized\
Port Conflict: Resolved\
Container Interference: Eliminated\
Gateway Service: Healthy\
Authentication: Pending device re-pair\
Tunnel: Needs restoration\
Production Hardening: Not yet applied

------------------------------------------------------------------------

**This document serves as the authoritative forward plan.**
