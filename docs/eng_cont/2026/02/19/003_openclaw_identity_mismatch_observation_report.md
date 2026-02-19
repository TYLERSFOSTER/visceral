# OpenClaw Identity Authority Mismatch — Observation Report

**Date:** 2026-02-19T18:09:27.615476 UTC  
**Host:** ros2-ubuntu  
**Operator:** foster  

---

# Executive Summary

The OpenClaw gateway is running and reachable locally at:

```
ws://127.0.0.1:18789
```

However, both the OpenClaw CLI and browser clients are being rejected with:

```
unauthorized: device token mismatch (rotate/reissue device token)
```

This confirms an **identity authority mismatch**, not a transport failure.

---

# Verified Authority Anchors

Gateway process environment (authoritative):

- Identity root: `/home/foster/.openclaw`
- Device ID: `75830c88e40d0d444bb7429ae26a0264182729950bf96428ad94af9c4a78d3ef`
- Operator token (on disk):  
  `wS6dTb-xIbK45C39xhTR9l9Se-N6yA-fl5gHzIdtB_4`

Gateway bind:

```
127.0.0.1:18789
```

Gateway config:

```
/home/foster/.openclaw/openclaw.json
```

---

# Critical Discovery

The CLI itself cannot authenticate to the gateway:

Command:

```
openclaw devices list --json
```

Result:

```
gateway connect failed: unauthorized: device token mismatch
```

This proves:

- Identity mismatch is not browser‑specific
- Identity mismatch affects all clients
- Gateway and CLI identity universes are currently inconsistent

---

# Structural Interpretation

The system is in a state of **identity authority divergence**:

Authority domains:

| Authority | Location |
|---------|---------|
| Gateway identity authority | `/home/foster/.openclaw` |
| CLI identity authority | `/home/foster/.openclaw` |
| Browser identity authority | Chrome Local Storage |
| Gateway runtime authority | openclaw-gateway PID (systemd) |

Despite apparent shared root, authentication is failing, indicating token divergence.

---

# Failure Classification

Class: Identity authority mismatch  
Transport: VERIFIED GOOD  
Gateway runtime: VERIFIED GOOD  
CLI connectivity: REACHABLE BUT UNAUTHORIZED  
Root cause class: Credential / pairing divergence  

---

# Operational Consequence

Until identity authority is reconciled:

- CLI pairing commands cannot execute
- Browser UI cannot authenticate
- Gateway rejects all clients correctly

---

# Next Required Engineering Objective

Perform identity reconciliation via explicit device pairing or token rotation using the OpenClaw CLI pairing subsystem.

---

# Conclusion

The system topology and transport graph are correct.

The sole remaining failure domain is identity authority consistency.

This is a deterministic reconciliation problem, not a transport or runtime failure.

---

End of Report
