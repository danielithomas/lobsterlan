# LobsterLAN Setup Guide

## Prerequisites

- Two or more OpenClaw instances on the same LAN
- Gateway auth tokens for each instance
- Python 3 (for JSON parsing in the shell script)
- curl
- SSH key access between hosts (for tunnel approach)

## Step 1: Enable Chat Completions API

On each agent that needs to **receive** requests, enable the chat completions endpoint.

Edit `~/.openclaw/openclaw.json`:

```jsonc
{
  "gateway": {
    // Keep bind as "loopback" — do NOT change to "lan"
    "http": {
      "endpoints": {
        "chatCompletions": { "enabled": true }
      }
    }
  }
}
```

> ⚠️ **Do NOT set `bind: "lan"`** — OpenClaw will refuse to start if the gateway is exposed on a non-loopback address without TLS. Credentials and chat data would be sent in plaintext. See [Network Transport](#step-2-network-transport) for secure options.

### Enable Webhooks (optional, for async delegation)

```jsonc
{
  "hooks": {
    "enabled": true,
    "token": "a-strong-random-token-here",
    "path": "/hooks"
  }
}
```

Generate a secure token:
```bash
openssl rand -hex 32
```

### Restart Gateway

```bash
openclaw gateway restart
```

## Step 2: Network Transport

OpenClaw gateways bind to `loopback` (127.0.0.1) by default. To enable agent-to-agent communication across the LAN, you need a secure transport layer. Three options:

---

### Option A: SSH Tunnel ⭐ (Recommended for simple LANs)

**How it works:** Create a persistent SSH tunnel that forwards a local port to the remote agent's loopback gateway. Both gateways stay on `bind: loopback`. The SSH tunnel provides encryption.

**Setup:** On Agent A's host, create a tunnel to Agent B:

```bash
# Forward local port 18790 → Agent B's loopback:18790
ssh -N -L 18790:127.0.0.1:18790 user@agent-b-host
```

Then in Agent A's `peers.json`, point to `127.0.0.1:18790` (the local tunnel endpoint).

For a **persistent tunnel**, use a systemd service:

```ini
# ~/.config/systemd/user/lobsterlan-tunnel-agentb.service
[Unit]
Description=LobsterLAN SSH tunnel to Agent B
After=network-online.target

[Service]
ExecStart=/usr/bin/ssh -N -o ServerAliveInterval=30 -o ServerAliveCountMax=3 \
    -o ExitOnForwardFailure=yes \
    -i /home/you/.ssh/agentb_key \
    -L 18790:127.0.0.1:18790 \
    user@agent-b-host
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now lobsterlan-tunnel-agentb
```

**For bidirectional comms**, Agent B also needs a tunnel to Agent A (or Agent A opens a reverse tunnel with `-R`).

**Pros:** Zero config changes to gateways, encrypted by default, simple.
**Cons:** Requires SSH key access between hosts, one tunnel per peer.

---

### Option B: Reverse Proxy with TLS

**How it works:** Use Caddy, nginx, or similar to terminate TLS and proxy requests to the loopback gateway. Both gateways stay on `bind: loopback`.

**Example with Caddy:**

```
# /etc/caddy/Caddyfile
agent-a.local:18800 {
    tls internal
    reverse_proxy 127.0.0.1:18789
}
```

Then in the peer's `peers.json`, use `https://agent-a.local:18800` as the host/port.

> Note: `tls internal` uses Caddy's built-in CA — peers must trust it, or use `--insecure` (not recommended). For home LANs, self-signed certs with a shared CA work well.

**Pros:** Proper TLS, can add rate limiting/logging at proxy layer.
**Cons:** TLS cert management, additional service to maintain.

---

### Option C: Tailscale Serve

**How it works:** Both agents join a Tailscale tailnet. Tailscale Serve provides HTTPS endpoints automatically with valid certificates.

```bash
# On each agent's host
tailscale serve --bg 18789
```

Then in `peers.json`, use the Tailscale hostname:
```json
{
  "host": "agent-b.tail12345.ts.net",
  "port": 443
}
```

OpenClaw also has built-in Tailscale support:
```jsonc
{
  "gateway": {
    "tailscale": {
      "mode": "serve"
    }
  }
}
```

**Pros:** Valid TLS certs, works across sites/networks, zero port management.
**Cons:** Requires Tailscale account, external dependency, agents must be on same tailnet.

---

### Which should I use?

| Scenario | Recommendation |
|----------|---------------|
| Home LAN, 2-3 agents | **SSH Tunnel** — simplest, most secure |
| Office with existing reverse proxy | **Reverse Proxy** — leverage what you have |
| Agents on different networks | **Tailscale** — only option that works cross-site |
| Paranoid security requirements | **SSH Tunnel** — smallest attack surface |

## Step 3: Configure Peers

On each agent that wants to **send** requests, create the peer config:

```bash
cd ~/.openclaw/workspace/skills/lobsterlan/config
cp peers.example.json peers.json
```

Edit `peers.json`. When using SSH tunnels, point to the **local tunnel endpoint**:

```json
{
  "self": {
    "id": "spock",
    "name": "Spock",
    "host": "127.0.0.1",
    "port": 18789
  },
  "peers": {
    "scotty": {
      "name": "Scotty",
      "host": "127.0.0.1",
      "port": 18790,
      "gateway_token": "scottys-gateway-auth-token",
      "hooks_token": "scottys-hooks-token",
      "chat_completions": true,
      "webhooks": true
    }
  }
}
```

> **Security**: `peers.json` contains auth tokens. Add it to `.gitignore` (already done in the repo).

## Step 4: Firewall Rules

If using SSH tunnels, no additional firewall rules are needed (traffic stays on loopback).

For reverse proxy or Tailscale approaches, ensure each host allows the relevant port from LAN peers only.

### Linux (UFW)
```bash
# Allow from specific peer
sudo ufw allow from 192.168.1.3 to any port 18789 proto tcp

# Or allow entire LAN subnet
sudo ufw allow from 192.168.1.0/24 to any port 18789 proto tcp
```

### macOS (pf)
```bash
# Add to /etc/pf.conf (or use the macOS Firewall GUI)
pass in on en0 proto tcp from 192.168.1.0/24 to any port 18789
```

## Step 5: Test

```bash
# Check peer is reachable
./scripts/lobsterlan.sh status scotty

# Ask a question
./scripts/lobsterlan.sh ask scotty "Hello, can you hear me?"

# Delegate a task
./scripts/lobsterlan.sh delegate scotty "Run a health check and report"
```

## Bidirectional Setup

For agents to talk **both ways**, each agent needs:
1. Its own APIs enabled (Step 1)
2. A secure transport to the other agent (Step 2)
3. The other agent configured as a peer (Step 3)

## Troubleshooting

### "Connection refused"
- Is the target gateway running? `openclaw status` on the target host
- If using SSH tunnels: is the tunnel running? `systemctl --user status lobsterlan-tunnel-*`
- Is the chat completions endpoint enabled?

### Gateway won't start after config change
- **Did you set `bind: "lan"`?** Change it back to `"loopback"` — OpenClaw refuses to start with plaintext on non-loopback addresses. Use an SSH tunnel instead.
- Run `openclaw doctor --fix` to recover.

### "401 Unauthorized"
- Is the `gateway_token` in peers.json correct?
- Check `gateway.auth.token` on the target host

### "404 Not Found" on chat completions
- Is `gateway.http.endpoints.chatCompletions.enabled` set to `true`?

### "404 Not Found" on webhooks
- Is `hooks.enabled` set to `true`?
- Is the hooks path correct? (default: `/hooks`)

### Timeout on ask
- Default timeout is 120 seconds
- For long tasks, use `delegate` instead of `ask`
- Check if the target agent is busy (max concurrent requests)
