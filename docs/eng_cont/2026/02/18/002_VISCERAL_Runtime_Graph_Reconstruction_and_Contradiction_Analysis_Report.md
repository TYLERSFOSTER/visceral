# VISCERAL Runtime Graph Reconstruction & Contradiction Analysis Report

**Author:** ChatGPT (Consultant Mode -- Topology Stabilization Phase)\
**Date:** 2026‑02‑18\
**Scope:** Explicit system graph reconstruction, invariant declaration,
and contradiction analysis for OpenClaw deployment under the VISCERAL
architecture.

------------------------------------------------------------------------

# Executive Summary

This document establishes the authoritative runtime graph of the
OpenClaw deployment, explicitly binding processes, identity roots,
namespaces, and transport relationships.

This is not a debugging report.

This is a **referential stabilization artifact**, intended to eliminate
ambiguity and provide a complete system topology model from which
deterministic resolution can proceed.

No procedural actions are defined in this document.

------------------------------------------------------------------------

# Phase 1 --- System Graph Reconstruction

This section defines all runtime nodes and edges.

------------------------------------------------------------------------

## Node N1 --- OPENCLAW_GATEWAY_PROCESS

namespace: SERVER_GCP_VM_HOST\
machine: ros2‑ubuntu\
process: openclaw‑gateway\
PID: 76509\
parent: systemd --user (PID 52226)

Bind addresses:

-   IPv4: 127.0.0.1:18789\
-   IPv6: \[::1\]:18789

Authority:

systemd user service

Identity root:

/home/foster/.openclaw

Token authority:

systemd Environment directive

Status:

Authoritative gateway runtime.

------------------------------------------------------------------------

## Node N2 --- SYSTEMD_GATEWAY_AUTHORITY

namespace: SERVER_GCP_VM_HOST\
machine: ros2‑ubuntu\
unit: openclaw‑gateway.service

Defines:

-   OPENCLAW_GATEWAY_TOKEN
-   HOME=/home/foster

Identity authority:

/home/foster/.openclaw

This node defines the identity universe used by the gateway.

------------------------------------------------------------------------

## Node N3 --- DOCKER_IDENTITY_ROOT (Legacy Identity Universe)

namespace: SERVER_GCP_VM_HOST filesystem\
path: \~/visceral/data/openclaw_home

Ownership:

ubuntu:foster

Contains:

-   devices/paired.json
-   legacy identity state

Status:

NOT referenced by the current gateway authority.

Represents a separate identity universe.

------------------------------------------------------------------------

## Node N4 --- CLIENT_BROWSER_DEVICE_IDENTITY

namespace: CLIENT_LOCAL_MAC\
machine: Foster‑MacBook\
process: browser

Identity source:

browser local storage

clientId: openclaw-control-ui\
clientMode: webchat

Status:

Presenting identity rejected by gateway.

------------------------------------------------------------------------

## Node N5 --- SSH_TUNNEL_FORWARD

namespace: TUNNEL_SSH_FORWARD\
machine: Foster‑MacBook\
process: ssh

Local bind:

CLIENT_LOCAL_MAC:127.0.0.1:18790

Forward target:

SERVER_GCP_VM_HOST:127.0.0.1:18789

Function:

Transport alias between client and gateway.

Status:

Operational.

------------------------------------------------------------------------

## Node N6 --- CLIENT_LOCAL_PORT_ALIAS_CONFLICTOR

namespace: CLIENT_LOCAL_MAC\
process: Code Helper (Chromium subsystem)

Bind:

CLIENT_LOCAL_MAC:127.0.0.1:18789

Status:

Unrelated process occupying canonical port.

Does not affect tunnel binding on port 18790.

------------------------------------------------------------------------

# Phase 2 --- Edge Definition

Explicit runtime relationships:

Transport path:

CLIENT_BROWSER_DEVICE_IDENTITY → SSH_TUNNEL_FORWARD →
OPENCLAW_GATEWAY_PROCESS

Authority definition:

SYSTEMD_GATEWAY_AUTHORITY → OPENCLAW_GATEWAY_PROCESS

Legacy identity root exists but is not referenced:

DOCKER_IDENTITY_ROOT (disconnected identity universe)

------------------------------------------------------------------------

# Phase 3 --- Invariant Declaration

The following invariants define correct system operation.

------------------------------------------------------------------------

## Invariant I --- Single Gateway Authority

Satisfied.

Only one active gateway authority exists:

SYSTEMD_GATEWAY_AUTHORITY → OPENCLAW_GATEWAY_PROCESS

------------------------------------------------------------------------

## Invariant II --- Single Identity Universe

Violated.

Two identity roots exist:

Active identity root:

/home/foster/.openclaw

Legacy identity root:

\~/visceral/data/openclaw_home

These represent separate identity universes.

------------------------------------------------------------------------

## Invariant III --- Explicit Namespace‑Qualified Addressing

Satisfied.

Canonical access path:

CLIENT_LOCAL_MAC:18790 → SERVER_GCP_VM_HOST:18789 →
OPENCLAW_GATEWAY_PROCESS

------------------------------------------------------------------------

## Invariant IV --- Transport Reachability

Satisfied.

Gateway reachable via SSH tunnel.

Authentication failure occurs after transport stage.

------------------------------------------------------------------------

# Phase 4 --- Contradiction Table

------------------------------------------------------------------------

## Contradiction C1 --- Identity Split‑Brain

Evidence:

Two identity roots exist:

/home/foster/.openclaw\
\~/visceral/data/openclaw_home

Consequence:

Clients paired against legacy identity root rejected by current gateway.

Observed symptom:

device token mismatch

------------------------------------------------------------------------

## Contradiction C2 --- Client Identity Bound to Legacy Authority

Evidence:

paired.json under legacy identity root contains browser device
identities.

Gateway authority does not reference that identity root.

Consequence:

Client identity invalid relative to gateway authority.

------------------------------------------------------------------------

## Contradiction C3 --- Identity Authority Transition Without Identity Migration

Deployment transitioned from docker‑compose authority to systemd
authority.

Client identities were not migrated.

Identity graph discontinuity exists.

------------------------------------------------------------------------

# Phase 5 --- Strategic Interpretation

Failure classification:

Identity authority discontinuity.

Not transport failure.\
Not gateway runtime failure.\
Not token misconfiguration.

System behavior is correct under identity authority change.

------------------------------------------------------------------------

# Phase 6 --- Stabilized System State

Authoritative gateway:

systemd → openclaw‑gateway

Identity authority:

/home/foster/.openclaw

Transport:

Operational

Failure location:

Identity validation edge between client and gateway.

------------------------------------------------------------------------

# Final Status

System topology fully reconstructed.\
Identity authority contradiction explicitly identified.\
No procedural resolution executed in this report.

This document establishes the authoritative runtime graph.

------------------------------------------------------------------------

# End of Report
