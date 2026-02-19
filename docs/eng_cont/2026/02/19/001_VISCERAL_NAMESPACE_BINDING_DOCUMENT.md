
# VISCERAL Namespace Binding Document
**Date:** 2026‑02‑18  
**Purpose:** Explicit referent binding across all namespaces to eliminate distributed topology ambiguity and epistemic control failure.

---

# 1. Referential Binding Protocol

Per Architectural Note: *Epistemic Control Failure Under Distributed Topology Ambiguity*, every symbolic referent must bind to:

(namespace, machine, process, address, port, protocol)

This document establishes those bindings explicitly for the VISCERAL / OpenClaw deployment.

---

# 2. Namespace Definitions and Bindings

## Namespace: CLIENT_LOCAL_MAC

Machine: Foster‑MacBook  
Role: Human interaction environment  

Processes currently bound:

- Process: Code Helper  
  PID: 58444  
  Address: 127.0.0.1  
  Port: 18789  
  Protocol: TCP  

- Process: ssh  
  PID: 58520  
  Address: ::1  
  Port: 18789  
  Protocol: TCP  

Interpretation:

This namespace contains a listener occupying the canonical ControlUI port locally.

This means:

localhost:18789 (CLIENT_LOCAL_MAC) refers to a macOS process, NOT necessarily the OpenClaw gateway.

---

## Namespace: SERVER_GCP_VM_HOST

Machine: ros2‑ubuntu  
Role: Authoritative OpenClaw gateway runtime host  

Expected binding:

(openclaw‑gateway, ros2‑ubuntu, PID ?, 127.0.0.1, 18789, ws/http)

This is the intended authoritative gateway referent.

Current binding must always be verified with:

lsof or ss inside SERVER_GCP_VM_HOST namespace.

---

## Namespace: SERVER_DOCKER_CONTAINER

Machine: ros2‑ubuntu  
Execution context: Docker container namespace  

Role: Optional OpenClaw runtime execution environment  

Important invariant:

container localhost ≠ host localhost

Only identical if network_mode: host is active.

---

## Namespace: TUNNEL_SSH_FORWARD

Machine: Foster‑MacBook  
Process: ssh (PID 58520)

Function:

Maps:

CLIENT_LOCAL_MAC:18789 → SERVER_GCP_VM_HOST:18789

This creates referent aliasing.

CLIENT_LOCAL_MAC localhost becomes proxy to remote gateway.

---

## Namespace: OPENCLAW_GATEWAY_PROCESS

Authoritative runtime referent.

Correct binding must resolve to exactly one:

(namespace, machine, PID, addr, port)

Violation creates referential split‑brain.

---

# 3. Architectural Consistency Analysis

The existing architectural documentation is fundamentally correct.

However, it omits explicit namespace binding tables.

Required architectural correction:

Add namespace binding registry as first‑class architectural artifact.

Architecture must explicitly name:

CLIENT_LOCAL_MAC  
SERVER_GCP_VM_HOST  
SERVER_DOCKER_CONTAINER  
TUNNEL_SSH_FORWARD  
OPENCLAW_GATEWAY_PROCESS  

---

# 4. Referential Invariant

The OpenClaw gateway must exist in exactly one authoritative namespace binding.

All client connections must resolve to that referent via explicit translation layers.

---

# 5. Failure Mode Prevented

Without this document, localhost references remain ambiguous.

With this document, every localhost reference becomes namespace‑qualified.

Example:

Incorrect:

localhost:18789

Correct:

CLIENT_LOCAL_MAC localhost:18789  
SERVER_GCP_VM_HOST localhost:18789  

---

# 6. Architectural Amendment Requirement

The VISCERAL architecture must adopt:

Namespace Binding Registry

as mandatory component of operational documentation.

This eliminates epistemic control failure.

---

# End of document
