
# OpenClaw Control UI Identity Reconciliation Gameplan
**Date:** 2026‑02‑19  
**Namespace Scope:** CLIENT_LOCAL_MAC, SERVER_GCP_VM_HOST, TUNNEL_SSH_FORWARD, OPENCLAW_GATEWAY_PROCESS  
**Purpose:** Establish a deterministic engineering plan to reconcile Control UI device identity with the authoritative gateway identity universe.

---

# Executive Summary

The OpenClaw gateway is operational, transport‑correct, and identity‑anchored correctly on SERVER_GCP_VM_HOST.

The remaining failure is exclusively due to a stale browser identity in CLIENT_LOCAL_MAC.

The Control UI is attempting to authenticate using a device identity that does not exist in the authoritative gateway device registry:

Authoritative registry:
```
SERVER_GCP_VM_HOST:/home/foster/.openclaw/devices/paired.json
```

Browser identity authority:
```
CLIENT_LOCAL_MAC: Chrome Local Storage LevelDB
~/Library/Application Support/Google/Chrome/Default/Local Storage/leveldb/
```

These identity authorities currently disagree.

The reconciliation objective is to cause the Control UI to register itself as a new device and approve that device in the gateway authority domain.

---

# Authority Model

## Authoritative Identity Registry

Namespace: SERVER_GCP_VM_HOST

Path:
```
/home/foster/.openclaw/devices/paired.json
```

This is the authoritative identity registry used by OPENCLAW_GATEWAY_PROCESS.

---

## Browser Identity Cache

Namespace: CLIENT_LOCAL_MAC

Path:
```
~/Library/Application Support/Google/Chrome/Default/Local Storage/leveldb/
```

This is the client‑side identity authority.

This identity is currently stale and rejected by the gateway.

---

# Transport Path (Verified)

CLIENT_LOCAL_MAC Browser  
→ TUNNEL_SSH_FORWARD  
→ SERVER_GCP_VM_HOST Gateway  
→ OPENCLAW_GATEWAY_PROCESS  

Transport authority is verified correct.

---

# Required Reconciliation Operation

The Control UI must register itself as a new device identity.

This occurs automatically when the browser connects without a valid cached identity.

---

# Execution Plan

## Phase 1 — Remove stale browser identity

Namespace: CLIENT_LOCAL_MAC

Operation:

```
rm -rf "$HOME/Library/Application Support/Google/Chrome/Default/Local Storage/leveldb"
rm -rf "$HOME/Library/Application Support/Google/Chrome/Default/Session Storage"
```

Then fully restart Chrome.

---

## Phase 2 — Trigger new device registration

Namespace: CLIENT_LOCAL_MAC

Open:

```
http://127.0.0.1:18790/__openclaw__/canvas/
```

Expected result:

Gateway receives pairing request.

---

## Phase 3 — Approve new device

Namespace: SERVER_GCP_VM_HOST

Command:

```
OPENCLAW_STATE_DIR=/home/foster/.openclaw openclaw devices list --json
```

Find new deviceId.

Approve:

```
OPENCLAW_STATE_DIR=/home/foster/.openclaw openclaw devices approve --device <deviceId>
```

---

# Expected Gateway Log Sequence

SERVER_GCP_VM_HOST journal:

```
pairing request received device=<deviceId>
device approved device=<deviceId>
client authenticated device=<deviceId>
```

---

# Success Criteria

Control UI connects without:

```
device_token_mismatch
```

Control UI transitions to:

```
connected
```

---

# Failure Modes

Possible causes of failure:

• Browser identity not fully cleared  
• Incorrect Chrome profile  
• Incorrect tunnel endpoint used  
• Gateway state directory mismatch  

---

# Conclusion

This is a deterministic identity reconciliation process.

Transport authority is correct.

Gateway authority is correct.

Client identity authority must be regenerated and approved.

This operation restores global identity consistency across namespaces.

---

# End of Gameplan
