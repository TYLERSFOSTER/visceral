# VISCERAL × OpenClaw Setup — Failure Postmortem + Recovery Plan (Initial Report)

**Date:** 2026-02-16  
**Context:** This report covers the OpenClaw / `visceral` setup session that bogged down and failed to converge. It is written to (1) document exactly what went wrong operationally, (2) prevent repeat failures, and (3) hand a clean, concrete plan to the next engineer.

> I’m sorry. The session cost you time and attention and made “reality drift” worse instead of better. The core failure mode was not technical difficulty — it was process failure: poor grounding, poor control hierarchy, and sloppy state tracking.

---

## 0. Executive summary

### What we actually learned (ground truth captured)
1. On the VM, **OpenClaw gateway is reachable locally** (you were able to `curl` `http://127.0.0.1:18789/__openclaw/control-ui-config.json` and get JSON).
2. The Control UI bundle stores a default `gatewayUrl` as:
   - `ws://(location.host)` or `wss://(location.host)` and also allows overriding via:
   - localStorage key `openclaw.control.settings.v1`
   - URL params `?gatewayUrl=...` or `#gatewayUrl=...`
3. You obtained an external IP via `curl ifconfig.me`:
   - `34.24.11.22`

### Why “nothing opens”
Given (1) + (3), “nothing opens” almost certainly means one (or both) of:
- **Binding issue:** the gateway is listening only on `127.0.0.1:18789` (loopback), not `0.0.0.0:18789`.
- **Network policy issue:** GCP firewall / VPC rules are not allowing inbound `18789/tcp`.

### The biggest non-technical failure
I failed to keep the collaboration in “Prime Directive” mode:
- I didn’t keep a crisp *single next action* loop.
- I didn’t re-ground repeatedly with high-signal checks.
- I let the session sprawl, and I let uncertainty accumulate instead of collapsing it quickly.

---

## 1. Where we are right now (state of the world)

### Known working components
- The gateway answers local HTTP requests at `127.0.0.1:18789` (the config endpoint returned JSON).
- The Control UI bundle logic for gateway selection is understood enough to override (localStorage / URL param).

### Unknowns that must be made explicit
- Whether the gateway process is bound to loopback only or all interfaces.
- Whether inbound traffic to `18789` is allowed by cloud firewall.
- Whether you are trying to open the Control UI from:
  - your laptop browser hitting the VM’s public IP directly, **or**
  - a browser running inside the VM, **or**
  - an SSH tunnel.

These unknowns should have been collapsed earlier with two tiny tests. I did not enforce that.

---

## 2. Concrete mistakes I made (actual details)

### 2.1 I failed to enforce “ground truth first”
**What I should have done immediately:**
- Ask for and lock down:
  - `ss -lntp | rg 18789` output
  - one external connect test (from *outside* the VM, or via `curl http://PUBLIC_IP:18789`)
  - GCP firewall rule listing for port 18789

**What I did instead:**
- Spent time in the JS bundle and UI internals *before* proving the network path end-to-end.
- That is backwards. UI internals are secondary; network reachability is first-order.

**Cost:**
- You were stuck oscillating between “UI config” and “server IP” confusion, which increases cognitive load and produces the “reality drift” feeling.

---

### 2.2 I did not maintain a strict single-step control loop
The correct “Prime Directive” pattern here is:

1. **Hypothesis**: “Port is loopback-bound or firewall-blocked.”
2. **Action**: Run one command that discriminates.
3. **Observation**: Interpret.
4. Repeat.

**What happened:**
- Multiple branches were opened at once (config fetch, ripgrep in bundle, devtools questions, IP determination).
- That’s “leading the meeting” and creating branching state without explicit user confirmation.

**Cost:**
- The debugging tree got wide instead of deep, and we didn’t close loops.

---

### 2.3 I used confusing language (“DevTools console”) without anchoring it
When I referenced **“In the browser DevTools console on the Control UI page”**, I did not:
- specify *which* browser (your laptop vs the VM),
- specify *which* page URL should be open,
- specify the *exact* console snippet to run,
- clarify that DevTools is a browser feature (F12 / Ctrl+Shift+I / ⌥⌘I), not a CLI concept.

**Cost:**
- You had to ask what it meant, and that question itself is a symptom: instructions were not operational.

---

### 2.4 I allowed IP confusion to persist too long
You asked: **“How do I figure out my server IP? Wait on my local machine?”**

I should have immediately snapped to a minimal truth table:
- **VM public IP**: the address your laptop uses to reach the VM.
- **VM private IP**: internal VPC address.
- **Laptop public IP**: irrelevant for hosting, only relevant for whitelists.

You ran `curl ifconfig.me` and got `34.24.11.22`. That is *some* public IP (likely the VM’s egress IP at that moment). I should have then insisted on the GCP metadata endpoint to confirm the VM’s assigned external IP, and then done a connect test.

**Cost:**
- You got stuck feeling like you were “drifting” between “which machine” and “which IP”.

---

### 2.5 I didn’t pull the one decisive discriminating test early enough
The key discriminating test is:

- If the gateway is bound to loopback only, **external connect fails even from the VM when targeting the public IP**.
- If it is bound to all interfaces, **that connect can succeed**, and then firewall is the issue.

That test should have been forced within 2 minutes of “nothing opens.”

**Cost:**
- Hours of avoidable churn.

---

### 2.6 I over-focused on bundle parsing rather than system boundaries
You did valuable work extracting from the bundle:
- the `gatewayUrl` localStorage key,
- URL parameter overrides,
- default `ws://(location.host)` behavior,
- the config endpoint `"/__openclaw/control-ui-config.json"` returning basePath and assistant identity fields.

That work matters — but it is **not** the critical path when the UI doesn’t open at all.

**Cost:**
- We became “correct about UI wiring” while still not solving “can the browser reach the server.”

---

### 2.7 I did not explicitly acknowledge emotional state and reduce cognitive load
You said: *“I’m getting fucking mad as I feel reality drifting.”*

I should have responded by **hard-resetting** to:
- one command,
- one interpretation,
- one next action,
and explicitly confirming which machine/browser you were operating from.

Instead I kept the session in a high-branch, high-text mode.

**Cost:**
- Increased frustration, decreased trust, and made the work feel unreal.

---

## 3. What we should actually do (recovery plan)

This is written for the next competent engineer to execute *fast*.

### Step A — confirm what IP the VM thinks it has (ground truth)
Run on the VM:

```bash
curl -fsS -H "Metadata-Flavor: Google"   "http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip"
```

Record it as `VM_PUBLIC_IP`.

Also get private IP:

```bash
curl -fsS -H "Metadata-Flavor: Google"   "http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip"
```

---

### Step B — confirm what address/port OpenClaw is actually listening on
Run:

```bash
ss -lntp | rg ':18789\b'
```

Interpretation:
- `127.0.0.1:18789` only → loopback-bound (not publicly reachable).
- `0.0.0.0:18789` or `*:18789` → bound to all interfaces.

---

### Step C — do the decisive “public IP from inside VM” connect test
Still on the VM:

```bash
curl -v "http://VM_PUBLIC_IP:18789/__openclaw/control-ui-config.json"
```

Interpretation:
- **Fails** → the service is not bound to external interface (or local routing is blocked); fix binding first.
- **Works** → binding is fine; firewall is blocking external clients.

---

### Step E — if firewall is wrong, open the port (or avoid it with SSH tunnel)
If Step C works but your laptop can’t connect, it’s firewall.

**Option 1: SSH tunnel (fastest, no cloud changes)**
From your laptop:

```bash
ssh -L 18789:127.0.0.1:18789 foster@VM_PUBLIC_IP
```

Then open in your laptop browser:

- `http://127.0.0.1:18789/chat`

---

## 4. What “a good engineer coming after me” should do differently

### Process rules (non-negotiable)
1. **One step at a time.** Every message must end in exactly one command to run, unless the user explicitly asks for a multi-step block.
2. **Always discriminate.** Choose actions that split the hypothesis space in half (binding vs firewall).
3. **Repeat the state.** Maintain a short “current truths” list so we don’t drift.
4. **When the user is overloaded, compress.** Fewer words, fewer branches, more certainty.

---

## 5. Closing

If you hand this to the next engineer, they should be able to resolve “nothing opens” quickly by running:
- metadata IP fetch,
- `ss` binding check,
- `curl` to public IP from inside VM,
and then either:
- bind fix, or
- firewall/tunnel fix.

Again: I’m sorry for the time loss and the process failure.
