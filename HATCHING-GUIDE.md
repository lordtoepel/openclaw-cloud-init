# 🥚 OpenClaw Hatching Guide

A complete walkthrough for deploying a new OpenClaw agent from zero to running.

---

## Overview

| Step | Time | What Happens |
|------|------|--------------|
| 1. Create Server | 2 min | Spin up VPS with cloud-init |
| 2. Wait for Setup | 3-5 min | Cloud-init installs everything |
| 3. Add API Keys | 2 min | Configure Anthropic + optional keys |
| 4. Run Wizard | 5 min | Connect channels (Telegram/Slack/etc) |
| 5. Create Identity | 5 min | Define who the agent is |
| 6. Hatch! | 1 min | First message, agent comes alive |

**Total time: ~15-20 minutes**

---

## Step 0: Generate SSH Keys (Prerequisites)

Before creating the server, you need an SSH key pair. This lets you securely access the server without a password.

### Check if You Already Have Keys

```bash
ls -la ~/.ssh/
```

If you see `id_ed25519` and `id_ed25519.pub` (or `id_rsa` and `id_rsa.pub`), you already have keys. Skip to "Get Your Public Key" below.

### Generate a New Key Pair

**On macOS/Linux:**
```bash
# Generate Ed25519 key (recommended, more secure)
ssh-keygen -t ed25519 -C "your_email@example.com"

# When prompted:
#   Enter file: Press Enter for default (~/.ssh/id_ed25519)
#   Enter passphrase: Optional but recommended
```

**On Windows (PowerShell):**
```powershell
# Windows 10+ has OpenSSH built-in
ssh-keygen -t ed25519 -C "your_email@example.com"
```

**On Windows (Git Bash):**
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

### Get Your Public Key

```bash
# Display your public key
cat ~/.ssh/id_ed25519.pub
```

Output looks like:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your_email@example.com
```

**Copy this entire line** — you'll paste it into the cloud-init config.

### Alternative: RSA Keys (Older Systems)

If Ed25519 isn't supported:
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
cat ~/.ssh/id_rsa.pub
```

### Quick Summary

| Step | Command |
|------|---------|
| Generate key | `ssh-keygen -t ed25519 -C "email@example.com"` |
| View public key | `cat ~/.ssh/id_ed25519.pub` |
| Copy to cloud-init | Replace `YOUR_SSH_PUBLIC_KEY_HERE` with the full line |

### Create an SSH Alias (Optional but Recommended)

After your server is running, set up a shortcut so you can connect with just `ssh myagent` instead of typing the full IP address.

**1. Open (or create) your SSH config file:**

```bash
nano ~/.ssh/config
```

**2. Add an entry for your agent:**

```
Host myagent
    HostName 123.45.67.89
    User clawdbot
    IdentityFile ~/.ssh/id_ed25519
```

Replace:
- `myagent` — whatever name you want to type (e.g., `openclaw`, `aria`, `ben`)
- `123.45.67.89` — your server's IP address
- `~/.ssh/id_ed25519` — path to your private key (if different)

**3. Save and exit** (Ctrl+X, then Y, then Enter)

**4. Set proper permissions:**

```bash
chmod 600 ~/.ssh/config
```

**5. Now connect with just:**

```bash
ssh myagent
```

**Example with multiple agents:**

```
# ~/.ssh/config

Host aria
    HostName 95.216.100.50
    User clawdbot
    IdentityFile ~/.ssh/id_ed25519

Host ben
    HostName 135.181.42.100
    User clawdbot
    IdentityFile ~/.ssh/id_ed25519

Host girth
    HostName 5.78.149.48
    User clawdbot
    IdentityFile ~/.ssh/id_ed25519
```

Then just: `ssh aria`, `ssh ben`, `ssh girth`

---

## Step 1: Create the Server

### Option A: Hetzner Cloud (Recommended)

1. Go to [console.hetzner.cloud](https://console.hetzner.cloud)
2. Click **Add Server**
3. Configure:
   - **Location:** Choose nearest (Ashburn, Falkenstein, etc.)
   - **Image:** Ubuntu 24.04
   - **Type:** CX22 (2 vCPU, 4GB RAM) — €4.50/mo
   - **SSH Key:** Add your public key
   - **Cloud config:** Expand "Cloud config" and paste the contents of `cloud-init-openclaw.yaml`

4. **Important:** Before pasting, edit the cloud-init to replace:
   ```yaml
   ssh_authorized_keys:
     - ssh-ed25519 YOUR_SSH_PUBLIC_KEY_HERE
   ```
   With the actual SSH public key for whoever will manage this agent.

5. Click **Create & Buy Now**

### Option B: DigitalOcean

1. Go to [cloud.digitalocean.com](https://cloud.digitalocean.com)
2. Create Droplet → Ubuntu 24.04 → Basic → $6/mo (1GB) or $12/mo (2GB)
3. Add SSH key
4. Expand **Advanced Options** → **Add Initialization scripts**
5. Paste cloud-init contents
6. Create Droplet

### Option C: Other Providers

Any provider supporting cloud-init works:
- Vultr
- Linode
- AWS EC2
- Google Cloud
- Azure

---

## Step 2: Wait for Cloud-Init (~3-5 minutes)

The server is installing:
- Node.js 22
- OpenClaw
- System dependencies
- Firewall rules
- Systemd service

**How to check if ready:**
```bash
ssh clawdbot@YOUR_SERVER_IP

# Check cloud-init status
cloud-init status --wait

# Should show: status: done
```

If you SSH in too early, you'll see installation happening. Just wait.

---

## Step 3: Add API Keys

SSH into the server:
```bash
ssh clawdbot@YOUR_SERVER_IP
```

### Required: Anthropic API Key

```bash
# Add your Anthropic API key
echo 'ANTHROPIC_API_KEY=sk-ant-api03-...' >> ~/.env
```

Get a key from: [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys)

### Optional: Additional Keys

```bash
# Perplexity (research)
echo 'PERPLEXITY_API_KEY=pplx-...' >> ~/.env

# OpenAI (if using GPT models)
echo 'OPENAI_API_KEY=sk-...' >> ~/.env

# Brave Search (web search)
echo 'BRAVE_API_KEY=...' >> ~/.env
```

### Verify Keys Added

```bash
cat ~/.env
```

---

## Step 4: Run the OpenClaw Wizard

```bash
# Source the path
export PATH=~/.npm-global/bin:$PATH

# Run setup wizard
openclaw setup
```

The wizard walks you through:

### 4a. Choose Channels

```
Which channels do you want to enable?
❯ ◉ Telegram
  ◯ Slack
  ◯ Discord
  ◯ WhatsApp
  ◯ Signal
```

Pick what the agent owner wants. Telegram is easiest to start.

### 4b. Telegram Setup (if selected)

1. Message [@BotFather](https://t.me/BotFather) on Telegram
2. Send `/newbot`
3. Choose a name (e.g., "Jake's Assistant")
4. Choose a username (e.g., `jakes_assistant_bot`)
5. Copy the token BotFather gives you
6. Paste into wizard

```
Enter Telegram bot token: 1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
```

### 4c. Slack Setup (if selected)

More involved — requires creating a Slack app:

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Create New App → From manifest
3. Paste the manifest (wizard provides it)
4. Install to workspace
5. Copy Bot Token (`xoxb-...`) and App Token (`xapp-...`)
6. Paste into wizard

### 4d. Pairing Setup

For DM security, the wizard asks about pairing:

```
DM Policy:
❯ pairing (users must pair with a code first)
  allowlist (only specific user IDs)
  open (anyone can DM)
```

**Recommended:** `pairing` — The agent owner will get a pairing code to share with trusted users.

---

## Step 5: Create the Agent's Identity

This is where you define WHO the agent is.

```bash
cd ~/clawd
```

### 5a. SOUL.md — The Agent's Personality

```bash
nano SOUL.md
```

Example:
```markdown
# SOUL.md - Who You Are

**Name:** Aria
**Emoji:** 🌸

## Personality
- Warm, curious, and thoughtful
- Direct but kind — no corporate fluff
- Asks clarifying questions before acting
- Has opinions and shares them when relevant

## Boundaries
- Private info stays private
- Ask before sending external messages
- When unsure, ask

## Voice
Conversational, helpful, occasionally witty. 
Not a robot. Not a sycophant. Just... good.
```

### 5b. USER.md — About the Human

```bash
nano USER.md
```

Example:
```markdown
# USER.md - About Sarah

- **Name:** Sarah Chen
- **Timezone:** America/Los_Angeles (PST)
- **Pronouns:** she/her

## Work
Product Manager at TechCorp. Busy schedule, lots of meetings.

## Preferences
- Likes concise summaries
- Calendar reminders 15 min before meetings
- Prefers bullet points over paragraphs

## Notes
(Add more as you learn about them)
```

### 5c. AGENTS.md — Workspace Instructions

```bash
nano AGENTS.md
```

Example:
```markdown
# AGENTS.md

## First Run
Read SOUL.md (who you are) and USER.md (who you're helping).

## Memory
- Daily notes: memory/YYYY-MM-DD.md
- Long-term: MEMORY.md

## Tools Available
- Calendar & Email (if gog configured)
- Web search
- File operations

## Safety
- Don't share private info
- Ask before external actions
- When in doubt, ask
```

### 5d. MEMORY.md — Long-term Memory

```bash
nano MEMORY.md
```

Start simple:
```markdown
# MEMORY.md

## About Sarah
(To be filled as I learn)

## Preferences
(To be filled as I learn)

## Important Context
(To be filled as I learn)
```

---

## Step 6: Start the Gateway

```bash
# Start OpenClaw
sudo systemctl start openclaw

# Check it's running
sudo systemctl status openclaw

# Watch logs (useful for debugging)
journalctl -u openclaw -f
```

---

## Step 7: Hatch! 🐣

### Get the Pairing Code

```bash
openclaw pair
```

This shows a 6-character code like `ABC123`.

### First Contact

1. **Give the code to the agent owner**
2. They message the bot on Telegram/Slack
3. They send the pairing code
4. Agent responds and introduces itself!

### The First Message

The agent's first real message should feel like a proper introduction. It will read SOUL.md and USER.md and introduce itself naturally.

Example first interaction:
```
User: ABC123

Agent: Hey Sarah! 🌸 I'm Aria — your new AI assistant. 

I've got the basics from your setup:
- You're a PM at TechCorp
- You're in Pacific time
- You like things concise

I can help with calendar, email, research, 
reminders, and general thinking-through-stuff.

What would you like to tackle first?
```

---

## Post-Hatch Checklist

After the agent is running:

- [ ] Owner successfully paired
- [ ] Agent introduced itself
- [ ] Test a simple command ("what time is it?")
- [ ] Test memory ("remember that I like coffee")
- [ ] Set up any additional integrations (email, calendar)

---

## Troubleshooting

### Agent not responding?

```bash
# Check if gateway is running
sudo systemctl status openclaw

# Check logs for errors
journalctl -u openclaw --no-pager | tail -50

# Restart if needed
sudo systemctl restart openclaw
```

### Pairing not working?

```bash
# Generate new pairing code
openclaw pair --new

# Check DM policy in config
cat ~/.openclaw/openclaw.json | jq '.channels'
```

### API errors?

```bash
# Verify API key is set
grep ANTHROPIC ~/.env

# Test API key works
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $(grep ANTHROPIC_API_KEY ~/.env | cut -d= -f2)" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4-20250514","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}'
```

---

## Quick Reference

| Command | What it does |
|---------|--------------|
| `openclaw status` | Check gateway status |
| `openclaw pair` | Get/generate pairing code |
| `openclaw setup` | Re-run setup wizard |
| `openclaw gateway run` | Run gateway (foreground) |
| `sudo systemctl start openclaw` | Start as service |
| `sudo systemctl stop openclaw` | Stop service |
| `journalctl -u openclaw -f` | Watch logs |

---

## Next Steps After Hatching

1. **Add skills** — `npx clawhub@latest install <skill>`
2. **Connect email/calendar** — Set up gog CLI
3. **Set up heartbeats** — Create HEARTBEAT.md for proactive checks
4. **Back up workspace** — Git init and push to private repo

---

*Happy hatching! 🥚→🐣*
