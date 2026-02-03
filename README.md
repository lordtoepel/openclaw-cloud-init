# OpenClaw Cloud Infrastructure

> **New here?** Start with [HATCHING-GUIDE.md](./HATCHING-GUIDE.md) for a complete walkthrough.

## Quick Deploy

### 1. Create Server
Use any cloud provider (Hetzner, DigitalOcean, Vultr, etc.):
- **OS:** Ubuntu 24.04 LTS
- **Size:** 2GB RAM minimum (4GB recommended)
- **User data:** Paste contents of `cloud-init-openclaw.yaml`

### 2. Replace SSH Key
In the cloud-init file, replace `YOUR_SSH_PUBLIC_KEY_HERE` with your actual SSH public key.

### 3. Wait for Setup (~3-5 min)
Cloud-init installs Node.js, OpenClaw, and configures the system.

### 4. SSH In and Configure
```bash
ssh clawdbot@YOUR_SERVER_IP

# Add your Anthropic API key
echo 'ANTHROPIC_API_KEY=sk-ant-...' >> ~/.env

# Run OpenClaw setup wizard
openclaw setup

# Start the gateway
sudo systemctl start openclaw

# Check logs
journalctl -u openclaw -f
```

## What's Installed

| Component | Location |
|-----------|----------|
| OpenClaw | `~/.npm-global/bin/openclaw` |
| Workspace | `~/clawd/` |
| Config | `~/.openclaw/openclaw.json` |
| Env vars | `~/.env` |
| Service | `/etc/systemd/system/openclaw.service` |

## Firewall Rules

- SSH (22): Open
- OpenClaw Gateway (18789): Open (for Tailscale/webhooks)

## Commands

```bash
# Start/stop/restart
sudo systemctl start openclaw
sudo systemctl stop openclaw
sudo systemctl restart openclaw

# View logs
journalctl -u openclaw -f

# Check status
openclaw status
```

## Clone an Existing Workspace

If you want to clone an existing agent's workspace:

```bash
# Clone your workspace repo
git clone https://github.com/YOUR_USER/clawd.git ~/clawd

# Copy environment variables (secure transfer)
scp user@old-server:~/.env ~/.env

# Start
sudo systemctl start openclaw
```
