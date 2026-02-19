# VISCERAL Runtime Authority Graph — Most‑True Topology Artifact (as of 2026‑02‑19)

**Purpose:**  
This document captures the authoritative runtime topology of the OpenClaw gateway deployment and the client‑side SSH forwarding environment.  
It explicitly identifies authority bindings, ambiguous authorities, and verified client‑side identity storage locations.

**Key update vs prior versions:**  
- **Gateway identity universe is proven** by the running gateway process environment: `HOME=/home/foster` ⇒ **identity root = `/home/foster/.openclaw`**.  
- **Client transport has shifted:** `127.0.0.1:18790` now returns `HTTP/1.1 200 OK` from the gateway canvas endpoint.  
- **IPv6 local listener on `::1:18789` is not currently reachable** (`Connection refused`), and `lsof` now shows **only Code Helper listening on IPv4 `127.0.0.1:18789`**.  
- **UI reports:** `disconnected (1008): unauthorized: device token mismatch (rotate/reissue device token)` — pointing to a **client identity mismatch** rather than transport failure.

---

## Verified Evidence Summary (high signal)

### SERVER_VM (ros2-ubuntu)
- Gateway process **PID 83685** listens on **loopback 18789** (v4 + v6) (previous evidence via `ss -lntp`).
- Gateway process environment (authoritative):
  - `HOME=/home/foster`
  - `OPENCLAW_GATEWAY_PORT=18789`
  - `OPENCLAW_GATEWAY_TOKEN=5a625f60...88fc`
  - `OPENCLAW_SERVICE_KIND=gateway`
  - `OPENCLAW_SERVICE_VERSION=2026.2.15`

**Consequence (authority binding):**  
`HOME=/home/foster` anchors OpenClaw’s runtime identity/config universe to:
- **IDENTITY_ROOT = `/home/foster/.openclaw`**  
and not any legacy directory.

### CLIENT_MAC (Foster-MacBook)
- `lsof -nP -iTCP:18789 -sTCP:LISTEN` now shows:
  - **Code Helper** listening on **IPv4 `127.0.0.1:18789`**
  - **no ssh listener** on that port at this moment
- `curl "http://[::1]:18789/__openclaw__/canvas/"` ⇒ **connection refused** (no IPv6 local listener)
- `curl http://127.0.0.1:18790/__openclaw__/canvas/` ⇒ **HTTP 200 OK** and returns Canvas HTML (5731 bytes)

### Client identity storage location (Chrome)
- Chrome Local Storage LevelDB directory exists at:
  - `~/Library/Application Support/Google/Chrome/Default/Local Storage/leveldb/`
  - contains many `*.ldb` files (evidence: directory listing)

**Consequence:**  
The browser’s device token / identity is *plausibly stored* in this Local Storage DB (exact key/value not yet extracted in this artifact).

---

# Diagram A — Detailed Runtime Reality (Full Fidelity)

```mermaid
flowchart LR
  %% =========================
  %% Diagram A: Detailed Runtime Reality (MOST TRUE as of 2026-02-19)
  %% =========================

  subgraph CLIENT_MAC["CLIENT_MAC  Foster-MacBook"]
    BROWSER["BROWSER  OpenClaw Dashboard / Control UI"]
    UIERR["UI_STATUS  disconnected (1008)\nunauthorized: device token mismatch\n(rotate/reissue device token)"]

    CODEHELP["CODEHELP  Code Helper Chromium\nLISTEN 127.0.0.1:18789 (IPv4)"]

    %% Local forwarding processes (state may change; only 18790 proven working now)
    SSH_A["SSH_A  (historical) -L 18789:127.0.0.1:18789 -> VM\nNOTE: NOT currently observed listening on ::1:18789\n(curl to ::1:18789 refused)"]
    SSH_B["SSH_B  -L 18790:127.0.0.1:18789 -> VM\nSTATUS: WORKING\nEVIDENCE: curl gets HTTP 200 + Canvas HTML"]
    SSH_C["SSH_C  (historical) -L 18791:127.0.0.1:18789 -> VM\nSTATUS: unknown now (previously stalled)"]

    LOCAL_IPV4_18789["LOCAL_ENDPOINT 127.0.0.1:18789 (IPv4)\nAUTHORITY: Code Helper\nNOTE: not OpenClaw tunnel"]
    LOCAL_IPV6_18789["LOCAL_ENDPOINT [::1]:18789 (IPv6)\nSTATUS: no listener reachable\nEVIDENCE: curl connection refused"]
    LOCAL_18790["LOCAL_ENDPOINT 127.0.0.1:18790\nAUTHORITY: ssh_B\nSTATUS: returns OpenClaw Canvas HTML"]
    LOCAL_18791["LOCAL_ENDPOINT 127.0.0.1:18791\nSTATUS: not re-verified in this snapshot"]

    CHROME_LS["CHROME_LOCAL_STORAGE (LevelDB)\n~/Library/.../Local Storage/leveldb/\nPOTENTIAL device token cache"]
  end

  subgraph SERVER_VM["SERVER_VM  ros2-ubuntu  GCP"]
    SYSD["SYSTEMD_USER_UNIT openclaw-gateway.service\nv2026.2.15\nExecStart: node ... openclaw gateway --port 18789"]
    GW["GATEWAY_PROCESS openclaw-gateway\nPID 83685\nLISTEN 127.0.0.1:18789 + [::1]:18789\nENV: HOME=/home/foster\nENV: OPENCLAW_GATEWAY_TOKEN=5a625f60...88fc"]
    TOK["TOKEN_AUTHORITY\nOPENCLAW_GATEWAY_TOKEN\n(injected via systemd -> process env)"]
    HOMEANCHOR["IDENTITY_UNIVERSE_ANCHOR\nHOME=/home/foster"]
    IDROOT["IDENTITY_ROOT (AUTHORITATIVE)\n/home/foster/.openclaw"]
    CFG["CONFIG_AUTHORITY\n/home/foster/.openclaw/openclaw.json"]
    DEVJSON["DEVICE_IDENTITY\n/home/foster/.openclaw/identity/device.json"]
    DEVAUTH["DEVICE_AUTH\n/home/foster/.openclaw/identity/device-auth.json"]
    IDLEG["LEGACY_IDENTITY_ROOT\n/home/foster/visceral/data/openclaw_home\nSTATUS: exists on disk\nNOT authoritative for current GW (HOME anchor excludes it)"]
  end

  %% --------- Client routing / authority ----------
  BROWSER -->|"tries localhost:18789 (IPv4)"| LOCAL_IPV4_18789
  BROWSER -->|"tries localhost:18789 (IPv6)"| LOCAL_IPV6_18789
  BROWSER -->|"uses known-good access"| LOCAL_18790
  BROWSER -->|"optional"| LOCAL_18791

  LOCAL_IPV4_18789 -->|"HTTP handled by Code Helper"| CODEHELP
  LOCAL_IPV6_18789 -->|"no listener reachable"| SSH_A
  LOCAL_18790 -->|"HTTP handled by ssh_B"| SSH_B

  %% Identity cache relationship
  BROWSER -. "stores device token / identity" .- CHROME_LS
  CHROME_LS -. "presents identity to gateway (WS auth)" .- UIERR

  %% --------- Tunnel to server ----------
  SSH_B -->|"forwards to SERVER_VM 127.0.0.1:18789"| GW
  SSH_C -. "not verified in this snapshot" .- GW

  %% --------- Server authority bindings ----------
  SYSD -->|"launches"| GW
  SYSD -->|"injects token env"| TOK
  SYSD -->|"sets HOME=/home/foster"| HOMEANCHOR

  HOMEANCHOR -->|"selects identity universe"| IDROOT
  TOK -->|"required for WS auth"| GW

  GW -->|"loads"| CFG
  GW -->|"uses identity root"| IDROOT
  IDROOT -->|"contains"| DEVJSON
  IDROOT -->|"contains"| DEVAUTH

  IDLEG -. "legacy universe exists but not authoritative" .- GW
```

---

# Diagram A (Condensed Transport + Auth View)

```mermaid
flowchart LR
  subgraph CLIENT_MAC
    BROWSER["Browser UI"]
    P18789["127.0.0.1:18789 (IPv4)"]
    P18790["127.0.0.1:18790 (forward)"]
    CODEHELP["Code Helper owns 18789"]
    SSH_B["ssh forward owns 18790"]
  end

  subgraph SERVER_VM
    GW["Gateway PID 83685\nloopback:18789"]
    AUTH["Auth layer\n(token + device identity)"]
  end

  BROWSER --> P18789 --> CODEHELP
  BROWSER --> P18790 --> SSH_B --> GW --> AUTH
```

---

# Diagram B — Minimal Authority Split Mechanism (Why “split reality” persists)

```mermaid
flowchart TB
  BROWSER["BROWSER chooses a local endpoint"]
  IPV4AUTH["IPv4 authority: 127.0.0.1:18789\n(Code Helper)"]
  FWDGOOD["Known-good forward: 127.0.0.1:18790\n(ssh -> VM gateway)"]
  AUTHFAIL["Auth reject: device token mismatch\n(browser identity != gateway identity universe)"]

  BROWSER --> IPV4AUTH
  BROWSER --> FWDGOOD --> AUTHFAIL
```

---

## Authority Conclusions (Current Snapshot)

### Server (canonical)
- **Gateway authority:** `openclaw-gateway` process **PID 83685**, systemd‑launched, loopback‑bound on `:18789`.
- **Token authority:** `OPENCLAW_GATEWAY_TOKEN=5a625f60...88fc` injected into the gateway process environment.
- **Identity authority:** **`/home/foster/.openclaw`**, proven by gateway process env `HOME=/home/foster`.

### Client (canonical access path)
- **Do not use** `127.0.0.1:18789` (IPv4) — currently owned by Code Helper.
- **Use** `127.0.0.1:18790` — proven to return gateway Canvas HTML over the SSH forward.

### Current failure class
- Transport is now good (at least via `18790`).
- Failure is now **authorization/identity mismatch**:
  - `unauthorized: device token mismatch (rotate/reissue device token)`
  - Likely involves stale browser identity cached in Chrome Local Storage vs gateway identity store under `/home/foster/.openclaw`.

---

## What remains to complete the graph for a handoff engineer
(These are *diagnosis targets*, not actions in this artifact.)

1) **Client identity binding**
- Identify the exact key(s) in Chrome Local Storage that store the OpenClaw device token / client id.
- Confirm what value is presented during WS auth.

2) **Server expected device roster**
- Enumerate the gateway’s paired/expected devices under `/home/foster/.openclaw` (exact file(s) / schema).
- Confirm mismatch is between browser token and server expected token.

3) **Reconciliation mechanism**
- Determine the canonical, supported way to rotate/reissue device token (UI or CLI), and whether it is server-side, client-side, or both.

---

# End of Document
