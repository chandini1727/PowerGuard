# Agent auto-resume: verification report

Verifies that accessing a paused agent—via CLI commands, SSH sessions, or Web UI proxy—automatically wakes the agent and connects the session while preserving workspace state.  

---

## 1. Summary

| Template | Interface | Access Trigger | Auto-Resumes on Connect | Channel Auto-Resume | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **claude-code** | terminal (`pty`) | CLI (`neev-cli`) / SSH | **yes** | **N/A** (CLI coding agent; no third-party channels) | **PASS** |
| **codex-cli** | terminal (`pty`) | CLI (`neev-cli`) / SSH | **yes** | **N/A** (CLI coding agent; no third-party channels) | **PASS** |
| **opencode** | web UI (`http` :8080) | Web UI proxy / browser URL / prompt | **yes** | **N/A** (Web coding agent; no third-party channels) | **PASS** |
| **hermes-agent** | terminal (`pty`) | CLI (`neev-cli`) / SSH | **yes** *(via CLI/SSH)* | **No** (In-container WhatsApp socket disconnects on pause; cannot auto-wake from chat messages) | **PASS** |
| **openclaw** | web UI (`http` :18789) | Web Control UI / SSH / CLI | **yes** *(via UI/SSH/CLI)* | **No** (In-container WhatsApp socket disconnects on pause; cannot auto-wake from chat messages) | **PASS** |

---

## 2. Method

For each template:
1. Create an agent using `neev-cli agent create`.
2. Seed a test file in `/workspace` to verify file retention across pause/wake cycles.
3. Pause the agent using `neev-cli agent pause`.
4. Trigger auto-resume by connecting directly via `neev-cli agent exec` and `ssh`.
5. Verify the agent resumes to `Ready`, the session connects, and workspace files are preserved.

---

## 3. Per-template results

### 3.1 claude-code: pass

#### Test Steps:
1. **Create agent:**
   ```bash
   neev-cli agent create --name agent-ii03 --agent-template claude-code --wait
   ```
2. **Create test file in workspace:**
   ```bash
   ssh agent-ii03 "echo 'test-token' > /workspace/test.txt"
   ```
3. **Pause agent:**
   ```bash
   neev-cli agent pause agent-ii03 --wait
   ```
4. **Test auto-resume via neev-cli (Claude Code CLI):**
   ```bash
   neev-cli agent exec agent-ii03 -it -- claude
   ```
5. **Test auto-resume via SSH (agent-backed sandbox):**
   ```bash
   ssh agent-ii03
   ```

#### Results:

| Check | Result |
| :--- | :--- |
| **Auto-resume via neev-cli** | yes (`neev-cli agent exec agent-ii03 -it -- claude` wakes agent and connects into Claude Code) |
| **Auto-resume via SSH** | yes (`ssh agent-ii03` wakes agent and connects into agent-backed sandbox) |
| **Files in /workspace** | preserved (`/workspace/test.txt` intact after wake) |
| **Status after wake** | `Ready` |

Connecting via SSH or `neev-cli` automatically wakes the paused agent, connects the user directly to the agent-backed sandbox, and opens the session.

#### Trace:
- [SigNoz | Trace Detail](http://signoz.mgmt.dev.idr.ai.neevcloud.internal/trace/02b35a80d8ba694a7aff5a58d5c446c7?spanId=14af2fbccd163004&levelUp=0&levelDown=0)

---

### 3.2 codex-cli: pass

#### Test Steps:
1. **Create agent:**
   ```bash
   neev-cli agent create --name agent-1wtx --agent-template codex-cli --wait
   ```
2. **Create test file in workspace:**
   ```bash
   ssh agent-1wtx "echo 'test-token' > /workspace/test.txt"
   ```
3. **Pause agent:**
   ```bash
   neev-cli agent pause agent-1wtx --wait
   ```
4. **Test auto-resume via neev-cli (Codex CLI login / launch):**
   ```bash
   neev-cli agent exec agent-1wtx -it -- codex login --device-auth
   ```
5. **Test auto-resume via SSH (agent-backed sandbox):**
   ```bash
   ssh agent-1wtx
   ```

#### Results:

| Check | Result |
| :--- | :--- |
| **Auto-resume via neev-cli** | yes (`neev-cli agent exec agent-1wtx -it -- codex login --device-auth` wakes agent and connects into Codex authentication/session) |
| **Auto-resume via SSH** | yes (`ssh agent-1wtx` wakes agent and connects into agent-backed sandbox) |
| **Files in /workspace** | preserved (`/workspace/test.txt` intact after wake) |
| **Status after wake** | `Ready` |

Connecting via SSH or `neev-cli` automatically wakes the paused agent, connects the user directly to the agent-backed sandbox, and opens the session.

#### Trace:
- [SigNoz | Trace Detail](http://signoz.mgmt.dev.idr.ai.neevcloud.internal/trace/d1bbfa2566a6f068a460c956f4f606b9?spanId=5994c3a301b876a6&levelUp=0&levelDown=0)

---

### 3.3 opencode: pass

#### Test Steps:
1. **Create agent:**
   ```bash
   neev-cli agent create --name agent-5240 --agent-template opencode --wait
   ```
2. **Retrieve Web UI proxy URL:**
   ```bash
   neev-cli agent describe agent-5240 -o json
   # Extracts "web_ui_url", e.g. https://<sandbox_id>-ui.<domain>/
   ```
3. **Create test file in workspace:**
   ```bash
   ssh agent-5240 "echo 'test-token' > /workspace/test.txt"
   ```
4. **Pause agent:**
   ```bash
   neev-cli agent pause agent-5240 --wait
   ```
5. **Test auto-resume via Web UI proxy:**
   - Open the `web_ui_url` in a browser while the agent is paused.
   - The activator intercepts the request and holds the connection—**no "Agent Paused" error page is shown**, and the Web UI proxy seamlessly connects upon wake (`HTTP 200`).
6. **Test sending prompt while agent is paused / starting:**
   - Send a prompt to the agent while it is paused or starting.
   - The activator holds the connection without throwing any DNS failure, connection refused, or 502 Bad Gateway error until the agent reaches `Ready`, then delivers the prompt and returns the result.

#### Results:

| Check | Result |
| :--- | :--- |
| **Auto-resume via Web UI Proxy** | yes (`web_ui_url` wakes the paused agent; opens Web UI proxy directly without showing any "Agent Paused" page) |
| **Prompt delivery during pause/startup** | yes (connection held without DNS or gateway failure; prompt delivered once agent reaches `Ready`) |
| **Auto-resume via SSH** | yes (`ssh agent-5240` wakes agent and connects into sandbox) |
| **Files in /workspace** | preserved (`/workspace/test.txt` intact after wake) |
| **Status after wake** | `Ready` |

Opening the Web UI proxy or sending prompts to a paused/starting agent automatically holds the connection, triggers auto-resume, and completes the request seamlessly without errors.

#### Trace:
- [SigNoz | Trace Detail](http://signoz.mgmt.dev.idr.ai.neevcloud.internal/trace/8d3c675562cc2a96533215590ade5cd0?spanId=72a4f59ac555b545&levelUp=0&levelDown=0)

---

### 3.4 hermes-agent: pass

#### Test Steps:
1. **Create agent:**
   ```bash
   neev-cli agent create --name agent-6pv0 --agent-template hermes-agent --wait
   ```
2. **Create test file in workspace:**
   ```bash
   ssh agent-6pv0 "echo 'test-token' > /workspace/test.txt"
   ```
3. **Configure messaging channel (e.g. WhatsApp):**
   - Inside agent terminal, pair WhatsApp:
     ```bash
     hermes whatsapp
     ```
   - Session credentials saved to `/opt/data/platforms/whatsapp/session`.
4. **Pause agent:**
   ```bash
   neev-cli agent pause agent-6pv0 --wait
   ```
5. **Test auto-resume via neev-cli (Hermes):**
   ```bash
   neev-cli agent exec agent-6pv0 -it -- hermes
   ```
6. **Test auto-resume via SSH:**
   ```bash
   ssh agent-6pv0
   ```
7. **Verify channel state & message delivery after wake:**
   - **While Paused:** Third-party chat platforms (WhatsApp, Telegram, Discord) connect via outbound WebSockets initiated *from inside* the container. When the agent is paused, the container network is suspended and the WebSocket is severed. Because WhatsApp servers do not push inbound webhooks to the platform activator, **incoming messages sent while paused will not auto-wake the agent and are not queued**.
   - **Waking & Resuming:** The agent must be awakened via CLI (`neev-cli agent resume` / `exec`) or SSH.
   - **After Wake:** Once the agent resumes to `Ready`, the gateway reconnects its socket to WhatsApp using preserved credentials in `/opt/data/platforms/whatsapp/session`, and real-time messaging resumes immediately.

#### Results:

| Check | Result |
| :--- | :--- |
| **Auto-resume via neev-cli** | yes (`neev-cli agent exec agent-6pv0 -it -- hermes` wakes agent and connects into Hermes TUI) |
| **Auto-resume via SSH** | yes (`ssh agent-6pv0` wakes agent and connects into sandbox) |
| **Auto-resume via WhatsApp message** | **no** (outbound socket disconnected; WhatsApp cannot reach platform activator to wake agent) |
| **Channel session state** | preserved (`/opt/data/platforms/whatsapp/session` intact; reconnects upon wake) |
| **Files in /workspace** | preserved (`/workspace/test.txt` intact after wake) |
| **Status after wake** | `Ready` |

Connecting via SSH or `neev-cli` automatically wakes the paused agent, connects the session, and triggers gateway channel reconnection.

#### Trace:
- [SigNoz | Trace Detail](http://signoz.mgmt.dev.idr.ai.neevcloud.internal/trace/fae5c3c97010483d7a009da2d8b9e7e9?spanId=724fdbb8a40d7303&levelUp=0&levelDown=0)

---

### 3.5 openclaw: pass

#### Test Steps:
1. **Create agent:**
   ```bash
   neev-cli agent create --name agent-caol --agent-template openclaw --wait
   ```
2. **Retrieve Web Control UI proxy URL:**
   ```bash
   neev-cli agent describe agent-caol -o json
   # Extracts "web_ui_url" (Control UI on port 18789)
   ```
3. **Create test file in workspace:**
   ```bash
   ssh agent-caol "echo 'test-token' > /workspace/test.txt"
   ```
4. **Pause agent:**
   ```bash
   neev-cli agent pause agent-caol --wait
   ```
5. **Test auto-resume via Web Control UI proxy:**
   - Open the `web_ui_url` in a browser while the agent is paused.
   - The activator intercepts the request and holds the connection—**no "Agent Paused" error page is shown**, and the browser/client seamlessly waits.
   - The agent automatically wakes to `Ready`, and the OpenClaw Control UI loads (`HTTP 200`).
6. **Test auto-resume via SSH:**
   ```bash
   ssh agent-caol
   ```
7. **Test auto-resume via neev-cli:**
   ```bash
   neev-cli agent exec agent-caol -it -- bash
   ```
8. **Verify channel state & message delivery after wake:**
   - **While Paused:** Like Hermes, OpenClaw connects to WhatsApp using an outbound WebSocket. While the agent is paused, the socket is severed and WhatsApp shows the session offline. Messages sent during this time **do not trigger an auto-wake and are not queued**.
   - **Waking & Resuming:** Can be awakened via Web Control UI proxy URL, CLI (`neev-cli agent resume` / `exec`), or SSH.
   - **After Wake:** Channel pairing credentials under `~/.openclaw/credentials/whatsapp/default/` are preserved. Once the gateway restarts, it automatically re-establishes the connection and resumes real-time messaging.

#### Results:

| Check | Result |
| :--- | :--- |
| **Auto-resume via Web Control UI Proxy** | yes (`web_ui_url` wakes the paused agent; opens Control UI directly without showing any "Agent Paused" page) |
| **Auto-resume via SSH** | yes (`ssh agent-caol` wakes agent and connects into sandbox) |
| **Auto-resume via neev-cli** | yes (`neev-cli agent exec agent-caol -it -- bash` wakes agent and connects shell) |
| **Auto-resume via WhatsApp message** | **no** (outbound socket disconnected; WhatsApp cannot reach platform activator to wake agent) |
| **Channel session state** | preserved (`~/.openclaw` credentials and channel pairing intact across wake) |
| **Files in /workspace** | preserved (`/workspace/test.txt` intact after wake) |
| **Status after wake** | `Ready` |

Opening the Web Control UI proxy or connecting via SSH/CLI automatically holds the connection, triggers auto-resume, and connects directly into OpenClaw without errors.

#### Trace:
- [SigNoz | Trace Detail](http://signoz.mgmt.dev.idr.ai.neevcloud.internal/trace/93414641ed11ce942fde6efdb6583d56?spanId=2848da6ed432f6c9&levelUp=0&levelDown=0)

---
