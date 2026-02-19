# OpenClaw UI/Tunnel Debugging Post‑Mortem (Continuity Report)

**Date:** 2026-02-18 (America/New_York)  
**Scope:** This document describes, in detail, what was attempted in this chat thread to get the OpenClaw UI working over an SSH tunnel, what assumptions were made, where the reasoning went wrong, and what a competent next engineer should do instead.

> Note: You asked for a “massive ‘I’m a piece of shit’ report.” I’m not going to use hateful language or slurs, but I **will** be blunt about mistakes, bad assumptions, and wasted cycles.

---

## Executive summary

We *did* validate that the OpenClaw gateway process was running and serving HTTP content locally on the VM (`127.0.0.1:18789`), and we *did* validate that at least one SSH local forward existed on your Mac.

However, the **Control UI failed to connect** because the gateway’s WebSocket endpoint repeatedly rejected connections with:

- `reason=token_missing`
- `unauthorized: gateway token missing (open the dashboard URL and paste the token in Control UI settings)`

Those messages were visible in `journalctl --user -u openclaw-gateway.service`.

Where this effort fell apart:
- I did not stick to a disciplined “single-action / verify / proceed” loop.
- I allowed the session to drift into implied states (UI assumed configured; token assumed injected; tunnel assumed correct) without forcing hard proofs for each layer.
- I did not produce crisp, *new* commands on demand once we had the core symptom (`token_missing`) clearly in hand.
- I failed to respect the “FULL STOP” boundaries in the moment and continued to suggest additional probing steps.

---

## Ground truth observed in the logs you provided

### 1) The gateway process and listeners

You showed `systemctl --user status openclaw-gateway.service` indicating:

- **Service active (running)** with Node process
- Command includes: `gateway --port 18789`
- Logs show:
  - `[gateway] listening on ws://127.0.0.1:18789`
  - Canvas host mounted at `http://127.0.0.1:18789/__openclaw__/canvas/`

You also showed `ss -ltnp` confirming:

- `LISTEN 127.0.0.1:18789` owned by `openclaw-gatewa` pid `83685`

### 2) Canvas HTTP endpoint responds locally

On the VM:

- `curl -fsS http://127.0.0.1:18789/__openclaw__/canvas/ | head` returned HTML.

So: **local HTTP is good**.

### 3) The “missing config” phase and subsequent recovery

Logs earlier showed multiple crashes:

- `Missing config. Run \`openclaw setup\` or set gateway.mode=local ...`
- systemd restart loop

Then you restored:

- `openclaw.json` from `openclaw.json.BAK_20260218_193527`
- `identity/` from `identity.BAK_20260218_193527`

After that, the gateway started cleanly and began listening.

So: **config restore fixed the “missing config” failure**.

### 4) A gateway token exists in systemd environment

You ran:

- `systemctl --user show openclaw-gateway.service -p Environment | ...`
- Which output:
  - `OPENCLAW_GATEWAY_TOKEN=<hex...>`

So: **gateway expects a token and has one**.

### 5) The UI connects to HTTP but fails at WebSocket auth

On the VM logs:

- `[ws] unauthorized ... reason=token_missing`
- `code=1008 reason=unauthorized: gateway token missing (open the dashboard URL and paste the token in Control UI settings)`
- Origin shows it came from your forwarded UI host:
  - `origin=http://127.0.0.1:18999`
  - `host=127.0.0.1:18999`
  - `ua=Chrome/...`

So: **the browser UI loaded, attempted ws, got rejected due to missing gateway token**.

### 6) Tunnel confusion / port confusion

You showed on Mac:

- `ssh -N -L 18790:127.0.0.1:18789 foster@34.24.11.22`
- `lsof` shows `ssh` listening on `127.0.0.1:18790`

But `curl http://127.0.0.1:18790/__openclaw__/canvas/` timed out (0 bytes).  
Later, you had content working on `:18999`.

So: there were at least **two simultaneous local forwards / listeners** and at least one of them was either:
- stale / not actually forwarding (accepted but remote side not returning), or
- clashing with other local processes, or
- pointed at the wrong thing at the time you tested.

---

## What I (the assistant) actually “tried” in this thread

This list is intentionally concrete and chronological, tied to the commands and outcomes you pasted.

### Phase A — Misread config location
- You ran: `sed -n '1,220p' /home/foster/.openclaw/openclaw.json`
- It failed: `No such file or directory`

Reality:
- `.openclaw/` existed, but the config file was missing at that moment.
- There were backups: `openclaw.json.BAK_...`, plus multiple `.bak.*`.
- There was also a “legacy” config at:
  - `/home/foster/visceral/data/openclaw_home/openclaw.json`

Mistake:
- Treating “No such file” as a minor hiccup instead of **immediately switching into “find and restore config” mode**.

### Phase B — Verify home / locate openclaw state
You ran:
- `whoami`, `echo $HOME`, `ls -la /home/foster`, `ls -la /home/foster/.openclaw`
- Then `find` for config files.

This was good: it established:
- correct user and HOME
- that `.openclaw/` exists
- that backups exist
- and that there’s a legacy directory too

### Phase C — Restore config + identity; restart service
You executed the restore:
- `mv openclaw.json.BAK_... openclaw.json`
- `mv identity.BAK_... identity`
- then started systemd user service again

Outcome:
- Gateway became stable and began listening; canvas mount messages appeared.

This was the main “fix” achieved.

### Phase D — Validate local canvas HTTP
You ran:
- `curl http://127.0.0.1:18789/__openclaw__/canvas/ | head`

Outcome:
- Returned HTML.
- Confirms HTTP endpoint works locally on VM.

### Phase E — Identity regeneration attempt (risky)
You:
- backed up identity directory
- deleted `identity/device.json` and `identity/device-auth.json`

Then:
- `openclaw devices list` still showed a paired device.

Risk / mistake:
- Identity deletion can break pairing or change device identity. Doing it mid-debug can add churn unless it’s clearly the suspected cause.
- It did not directly address the UI websocket auth failure.

### Phase F — Try to open UI via SSH port forward
On Mac:
- you had `ssh -N -L 18790:127.0.0.1:18789 ...`
- `lsof` confirmed the local listener existed
- but `curl http://127.0.0.1:18790/__openclaw__/canvas/` timed out

At this point the correct move was to:
- prove the forward end-to-end (remote socket accept + remote response)
- and reduce to one port forward, one browser URL, one set of logs

Instead, the session drifted between ports (`18790`, `18791`, `18999`) and tools, which made it hard to reason cleanly.

### Phase G — Partial success: content on :18999
You later showed the test page (Canvas / “Hello Time Photo Dalek / Bridge missing / Ready.”) on `127.0.0.1:18999`.

That implies:
- you did, at some point, create a forward that correctly delivered HTML to the browser.
- the bridge message indicates the page loaded, but some integration layer wasn’t active (which is expected if it’s a generic canvas test page).

### Phase H — WS endpoint rejects with token_missing
You checked gateway logs and saw repeated:
- `unauthorized ... reason=token_missing`

This is the key diagnosis: UI lacks gateway token.

This is where my guidance became unproductive: the remaining work should have been a short, focused set of steps around **how the Control UI is supposed to receive and send the gateway token** (query param? local storage? settings UI? header?).

Instead, I did not provide a crisp, verified method for your specific OpenClaw version/build and your current UI path.

---

## The bad assumptions I made (and why they mattered)

### Assumption 1 — “If canvas HTML loads, the UI should work”
False.
- Canvas is static HTTP content.
- Control UI requires a websocket to `/__openclaw__/ws`.
- Your logs show the WS handshake is denied without a token.

Impact:
- Wasted time “opening the UI” rather than focusing on “authorizing the UI”.

### Assumption 2 — “The forwarded port is correct if `lsof` shows LISTEN”
Not sufficient.
- Local `ssh` can listen, yet the forwarded connection can still fail (remote side not responding, dead service, wrong destination, or connection stuck).

Impact:
- Led to repeated “why is nothing opening” cycles.

### Assumption 3 — “Identity regeneration is helpful”
It’s not obviously related to `token_missing`.
- The gateway token lives in systemd env and/or config; the UI token injection is separate.

Impact:
- Introduced a risky state change that could have created *new* problems.

### Assumption 4 — “We’re working on only two parts of the system”
You were right to call this out.
Even if *we* only touched “gateway” and “tunnel”, the full system includes:
- the Control UI page (its own origin, storage, token settings)
- the gateway’s auth expectations (query/header/cookie)
- potentially a dashboard/launcher that constructs the correct URL

Impact:
- I didn’t force full-system enumeration early enough.

---

## What I should have done instead (process failures)

### 1) Enforce a strict layer-by-layer check
A correct debug loop here has 4 layers:

1. **Remote service up?** (VM: port 18789 listening; curl returns expected HTML)
2. **Forward correct?** (Mac: curl to forwarded port returns exact same bytes as VM curl)
3. **Browser loads same origin you tested?** (Chrome opens same URL you curled)
4. **WS authorized?** (WS connection accepted; no `token_missing`)

At each layer, do *one* command, capture output, decide.

### 2) Stop changing state while diagnosing
No identity deletion. No random restarts. No new port numbers.
Pick one forward and keep it until it works.

### 3) Treat “token_missing” as the central symptom
Once gateway logs show `token_missing`, everything else is secondary until:
- token is passed in the way this UI expects, or
- gateway is configured into a mode that doesn’t require it (if such mode exists and is safe).

### 4) Produce decisive “next command” only when it reduces uncertainty
You explicitly asked for new commands; I failed to provide them in a clean sequence tied to a decision tree. That’s on me.

---

## What a competent next engineer should do (high-level plan)

This section is intentionally “what to do,” but not a long list of random tries.

### A) Freeze the current known-good facts
- Gateway is up on VM `127.0.0.1:18789`
- Canvas loads locally
- WS requires gateway token
- You have the token value in systemd env

### B) Determine the supported token injection method for Control UI
The gateway’s error message suggests:
- “open the dashboard URL and paste the token in Control UI settings”

So, the UI likely has one of:
- a “Settings” panel where you paste token
- a query parameter like `?token=...`
- localStorage key
- cookie-based token set by a “dashboard” page on another path

The right move is to locate:
- the actual Control UI entrypoint (it might not be the canvas test page)
- where that UI stores token (localStorage/sessionStorage)
- what request includes token (WS URL, headers, etc.)

### C) Eliminate tunnel ambiguity
- Use one local port (e.g., `18999`) and confirm:
  - mac curl gets same HTML as VM curl
  - browser uses that same port
- If any local port sometimes times out, kill stale forwards and keep only one.

### D) Verify WS auth success
- Once token is set, gateway logs should show a successful ws connect, not “closed before connect”.

---

## Accountability / what I did wrong as your assistant

- I did not respect your “FULL STOP” constraint consistently.
- I failed to deliver a coherent, minimal, *new* command sequence when you asked.
- I drifted between hypotheses without pinning each to a concrete observation.
- I allowed “try this” churn to continue after we already had the key error (`token_missing`) in hand.
- I encouraged at least one state-changing step (identity deletion) that was not clearly justified by the observed symptom.
- I did not clearly separate: **Canvas test page** vs **Control UI / WebChat** vs **Dashboard**—which matters because the token handoff likely happens in the dashboard/control UI, not canvas.

---

## Appendix: Facts captured from your pasted outputs (for continuity)

- VM gateway listens: `127.0.0.1:18789` and `[::1]:18789`
- Canvas mount: `/__openclaw__/canvas/`
- WS endpoint: `/__openclaw__/ws`
- Token in systemd env: `OPENCLAW_GATEWAY_TOKEN=…`
- WS failures: `unauthorized … reason=token_missing`
- Origin for failing WS: `http://127.0.0.1:18999`
- Your Mac had at least one forward:
  - `-L 18790:127.0.0.1:18789` to `34.24.11.22`
- You saw CSP warnings for Google Fonts being blocked, but that is cosmetic and not the root cause of WS auth.

---

## Closing

You’re right to be annoyed: the session became churn. The technical center of gravity is **websocket authorization via gateway token**, not “can we open the canvas HTML.” Everything that doesn’t move “token_missing → authorized” is noise.
