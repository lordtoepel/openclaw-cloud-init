# Fleet Update + Onboarding Runbook

Operational runbook for the Leanscale fleet of OpenClaw agents. Captures the procedures used on 2026-04-23 to bring the fleet to OpenClaw `2026.4.22` and model `claude-cli/claude-opus-4-7`, plus provisioning steps for new agents.

Fleet inventory: [`fleet.json`](./fleet.json). Keep it in sync — one entry per host.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Updating an existing bot (auth tour)](#updating-an-existing-bot-auth-tour)
3. [Fleet-wide OpenClaw update (bulk)](#fleet-wide-openclaw-update-bulk)
4. [Onboarding a new employee bot (end-to-end)](#onboarding-a-new-employee-bot-end-to-end)
5. [Adding a Slack user to allowlists](#adding-a-slack-user-to-allowlists)
6. [Gotchas & recovery](#gotchas--recovery)

---

## Prerequisites

- Access to Girth (`clawdbot@5.78.149.48`) with Jake's SSH key (you already have it)
- Hetzner API token stored at `~/.config/hcloud/cli.toml` (leanscale context) for provisioning + resizing
- Tailscale admin at <https://login.tailscale.com/admin/settings/keys> to mint auth keys
- Slack workspace admin to create new apps via manifest

---

## Updating an existing bot (auth tour)

Goal: move a bot from an older OpenClaw + older model to `2026.4.22` + `claude-cli/claude-opus-4-7` with OAuth against the owner's Claude Max account.

Per-bot steps:

```bash
# 1. SSH in (Tailscale IP or public IP from fleet.json)
ssh clawdbot@<TAILSCALE_IP>

# 2. Authenticate Claude CLI (interactive — browser opens)
~/.npm-global/bin/claude auth login --claudeai
# Click the URL, authenticate with the employee's Claude.ai Max account,
# paste the code back into the TTY

# 3. Run the finish-upgrade script (pre-deployed to each bot)
bash ~/finish-upgrade.sh
```

`~/finish-upgrade.sh` does:
1. Verifies `claude auth status` shows `loggedIn: true`
2. Backs up `~/.openclaw/openclaw.json`
3. Patches `agents.defaults.model.primary` → `claude-cli/claude-opus-4-7` + fallbacks
4. Adds auth profile `anthropic:claude-cli`
5. Installs `@anthropic-ai/claude-code` if missing
6. Restarts the gateway (systemd user service)
7. Verifies via `openclaw agents list`

Post-check:

```bash
~/.npm-global/bin/openclaw agents list         # Identity + Model line
~/.npm-global/bin/openclaw channels status --probe   # Slack/Telegram connectivity
```

### Alternate: bootstrap with Girth's creds (internal bots / before employee auth)

When you need a bot running immediately (e.g. Taskmaster, Admiral, or a new bot before the employee has their own Max account):

```bash
# From Girth:
scp ~/.claude/.credentials.json clawdbot@<TAILSCALE_IP>:~/.claude/.credentials.json
ssh clawdbot@<TAILSCALE_IP> "chmod 600 ~/.claude/.credentials.json && bash ~/finish-upgrade.sh"
```

Bot runs on Jake's Max account. Swap to employee's account later via the tour above.

---

## Fleet-wide OpenClaw update (bulk)

The `openclaw update` command is idempotent. For a sweep across the fleet, parallelize with a Python helper that SSHes and runs:

```bash
NODE_OPTIONS='--max-old-space-size=1536' \
  ~/.npm-global/bin/openclaw update --yes --json --timeout 800
```

The `NODE_OPTIONS` heap cap is important for **cpx11** (2GB) hosts — without it, V8 OOMs during the npm install phase.

See `/tmp/fleet-update-2.py` for the template used on 2026-04-23. Concurrency cap 4, per-host SSH timeout 800s, post-version verification 5s after completion.

### If update fails with OOM

```bash
# Option A: Retry with tighter heap cap
ssh clawdbot@<HOST> "NODE_OPTIONS='--max-old-space-size=1024' ~/.npm-global/bin/openclaw update --yes"

# Option B: Upgrade the Hetzner plan to cpx21 (4GB RAM, +€2/mo)
hcloud server poweroff <name>
hcloud server change-type <name> cpx21       # upgrades disk by default (one-way)
hcloud server poweron <name>
# Then retry the update
```

---

## Onboarding a new employee bot (end-to-end)

Reference example from 2026-04-23: Andrew / Flywheel Oneal.

### 1. Decide specs

| Setting | Default | Notes |
|---|---|---|
| Provider | Hetzner | consistent with fleet |
| Type | `cpx21` | 3 vCPU / 4GB / 80GB. **Avoid cpx11** for anything with regular updates. |
| Location | `hil` (Hillsboro) | same region as most of fleet |
| Image | `ubuntu-24.04` | |

### 2. Create the server

Cloud-init template lives at `infra/cloud-init-openclaw.yaml`. **Before using, edit it to use Node 22** (`setup_22.x`) — the checked-in template uses Node 20 which is too old for current OpenClaw.

```bash
hcloud server create \
  --name <employee>-openclaw \
  --type cpx21 \
  --image ubuntu-24.04 \
  --location hil \
  --user-data-from-file cloud-init.yaml \
  --label fleet=openclaw --label employee=<employee>
```

Captures the IPv4, instance ID. Wait for `~/.cloud-init-done` marker (the final runcmd in the template).

### 3. Verify / complete install

```bash
ssh clawdbot@<PUBLIC_IP> "
  node --version                      # want v22.x
  ~/.npm-global/bin/openclaw --version
  ~/.npm-global/bin/claude --version
"
```

If Node is v20 (cloud-init template outdated), upgrade:

```bash
ssh clawdbot@<IP> "
  curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
  sudo apt-get install -y nodejs
  # reinstall openclaw so it picks up the right node
  npm i -g openclaw@latest @anthropic-ai/claude-code
"
```

### 4. Join Tailscale

Mint an auth key at <https://login.tailscale.com/admin/settings/keys> (reusable: no, ephemeral: no, expires 1h).

```bash
ssh clawdbot@<IP> "
  curl -fsSL https://tailscale.com/install.sh | sudo sh
  sudo tailscale up --authkey=tskey-auth-XXXX --hostname=<employee>-openclaw
  tailscale ip -4
"
```

Record the Tailscale IP in `fleet.json`.

### 5. Create the Slack app

1. Go to <https://api.slack.com/apps> → **Create New App** → *From manifest*
2. Base manifest on `infra/slack-manifests/taskmaker.json`, swap `display_information.name` and `features.bot_user.display_name` to the new agent name
3. Enable **Socket Mode**
4. **Install to Workspace** → copy the **Bot Token** (`xoxb-...`)
5. **Basic Information** → *App-Level Tokens* → create token with `connections:write` scope → copy `xapp-...`

### 6. Write `~/.openclaw/openclaw.json`

Minimum viable config. Template:

```json
{
  "meta": {},
  "agents": {
    "defaults": {
      "model": {
        "primary": "claude-cli/claude-opus-4-7",
        "fallbacks": [
          "claude-cli/claude-opus-4-6",
          "claude-cli/claude-sonnet-4-6",
          "claude-cli/claude-opus-4-5",
          "claude-cli/claude-sonnet-4-5",
          "claude-cli/claude-haiku-4-5"
        ]
      },
      "models": {
        "claude-cli/claude-opus-4-7": {},
        "claude-cli/claude-opus-4-6": {},
        "claude-cli/claude-opus-4-5": { "alias": "opus" },
        "claude-cli/claude-sonnet-4-6": {},
        "claude-cli/claude-sonnet-4-5": {},
        "claude-cli/claude-haiku-4-5": {}
      },
      "heartbeat": { "every": "1h" }
    }
  },
  "auth": {
    "profiles": {
      "anthropic:claude-cli": { "provider": "claude-cli", "mode": "oauth" }
    }
  },
  "channels": {
    "slack": {
      "mode": "socket",
      "webhookPath": "/slack/events",
      "enabled": true,
      "botToken": "xoxb-...",
      "appToken": "xapp-...",
      "allowBots": true,
      "requireMention": true,
      "dmPolicy": "allowlist",
      "allowFrom": ["<EMPLOYEE_SLACK_USER_ID>"],
      "streaming": { "mode": "partial", "nativeTransport": true }
    }
  },
  "tools": {
    "profile": "full",
    "exec": { "security": "full" }
  }
}
```

Push to the new server:

```bash
scp openclaw.json clawdbot@<IP>:~/.openclaw/openclaw.json
ssh clawdbot@<IP> "chmod 600 ~/.openclaw/openclaw.json"
```

### 7. Onboard + install gateway

```bash
ssh clawdbot@<IP> "
  ~/.npm-global/bin/openclaw onboard --mode local --non-interactive --accept-risk --auth-choice skip
  ~/.npm-global/bin/openclaw gateway install
  ~/.npm-global/bin/openclaw gateway start
"
```

Without `openclaw onboard --mode local`, the gateway refuses to start (`gateway.mode` missing error).

### 8. Authenticate Claude CLI

Two options:

- **Interactive (correct long-term):** Employee SSHes in and runs `~/.npm-global/bin/claude auth login --claudeai`.
- **Bootstrap (for testing):** Copy Girth's creds with `scp ~/.claude/.credentials.json clawdbot@<IP>:~/.claude/.credentials.json`.

**⚠️ Important:** authenticate BEFORE sending the first Slack message to the bot. Otherwise the pre-auth failure gets cached as a "billing" error and `claude-cli` is locked out for ~5 hours. See [Gotchas](#gotchas--recovery) below if this happens.

### 9. Verify

```bash
ssh clawdbot@<IP> "
  ~/.npm-global/bin/openclaw agents list        # Model line
  ~/.npm-global/bin/openclaw channels status --probe  # Slack 'works'
"
```

### 10. Add to `fleet.json`

Append to the `fleet` array (for employee bots) or `servers` (for shared/experimental):

```json
{
  "name": "<employee>-openclaw",
  "agent": "<Agent Name>",
  "host": "<public_ip>",
  "tailscale": "<tailscale_ip>",
  "role": "...",
  "employee": "<Employee Name>",
  "provider": "hetzner",
  "type": "cpx21",
  "location": "hetzner-hil",
  "instanceId": "<hcloud_id>",
  "openclaw": "2026.4.22",
  "status": "active",
  "deployed": "YYYY-MM-DD",
  "notes": "SSH keys: Jake, Girth, Admiral."
}
```

---

## Adding a Slack user to allowlists

Target scope: **usually just that user's own bot**. Broader fan-out is generally not what's wanted.

### Single bot

```bash
ssh clawdbot@<TAILSCALE_IP> "python3 -c '
import json, pathlib
p = pathlib.Path.home() / \".openclaw/openclaw.json\"
d = json.loads(p.read_text())
af = d.setdefault(\"channels\", {}).setdefault(\"slack\", {}).setdefault(\"allowFrom\", [])
uid = \"U0XXXXXXX\"
if uid not in af:
    af.append(uid)
    p.write_text(json.dumps(d, indent=2) + \"\\n\")
    print(\"added\")
else:
    print(\"already-present\")
'"
# Restart if the allowlist isn't picked up live:
ssh clawdbot@<TAILSCALE_IP> "~/.npm-global/bin/openclaw gateway restart"
```

### Fleet-wide (rare)

Only do this if the user is a "power user" who should be able to DM every bot (e.g. an on-call engineer). Use a parallel Python driver like the patterns in this doc.

---

## Gotchas & recovery

### `claude-cli` provider stuck on "billing" error

**Symptom:** bot replies with *"API provider returned a billing error — your API key has run out of credits or has an insufficient balance."* Even after authenticating.

**Root cause:** Any pre-auth call failure on `claude-cli` gets persisted to `~/.openclaw/agents/main/agent/auth-state.json` with `disabledUntil` ~5 hours in the future. Subsequent restarts don't clear it.

**Fix:**

```bash
ssh clawdbot@<IP> "python3 -c '
import json, pathlib
p = pathlib.Path.home() / \".openclaw/agents/main/agent/auth-state.json\"
d = json.loads(p.read_text())
d[\"usageStats\"] = {}
p.write_text(json.dumps(d, indent=2) + \"\\n\")
' && ~/.npm-global/bin/openclaw gateway restart"
```

### `openclaw update` OOMs on cpx11 (2GB RAM)

**Symptom:** update exits mid-`npm install`, error mentions V8 `OOMDetails`.

**Fix:** either retry with heap cap (`NODE_OPTIONS='--max-old-space-size=1024'`) or resize host to cpx21.

### Remote `claude auth login` can't be automated

**Symptom:** tmux + `send-keys` / pexpect / bracketed-paste all fail to feed the code to the Ink TUI. It echoes the chars but never completes the OAuth exchange.

**Workaround:** auth must be done in a real interactive terminal (SSH in, run, paste). For bootstrap, copy Girth's `~/.claude/.credentials.json` via `scp` (works because credentials are a plain JSON file).

### My SSH key removed from a bot

**Symptom:** Girth can no longer SSH to a bot; Jake can. Happens when someone runs `openclaw doctor --fix` or similar which regenerates `authorized_keys`.

**Fix:** Jake SSHes in, appends Girth's pubkey to `~/.ssh/authorized_keys`:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILIws1KoJqNNczcRrZ/05+L5E43D1PfuKj8/nbsh36tS girth-brooks-automation
```

### Fleet.json version fields drift

`fleet.json` entries have an `openclaw` version string, but the daily auto-updater cron on each bot bumps the actual version independently. The field stays stale. Either drop it from the schema or add a periodic job that refreshes from `openclaw --version` probe.

### Hetzner `change-type --upgrade-disk`

The flag doesn't exist in hcloud 1.61+. Disk upgrade is the default behavior of `change-type`; use `--keep-disk` only to preserve current disk (e.g. for downgrades). Disk upgrade is one-way.

---

## Quick reference: fleet.json schema

Three arrays at top level:
- `fleet[]` — primary employee bots (one per human)
- `internalBots[]` — shared internal agents (Taskmaster, Sol, Diego, etc.)
- `servers[]` — catch-all for other Hetzner instances (experiments, finance-agent, etc.)

Each entry should have at minimum: `name`, `agent`, `host`, `tailscale`, `employee` (or `owner`), `type`, `location`, `status`. Top-level `updated: YYYY-MM-DD`.
