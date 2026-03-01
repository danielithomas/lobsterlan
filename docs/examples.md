# LobsterLAN Examples

## From the Command Line

### Quick status check
```bash
./scripts/lobsterlan.sh status scotty
# ✅ scotty is reachable at 192.168.1.3:18789
```

### Ask a question (sync)
```bash
./scripts/lobsterlan.sh ask scotty "What Docker containers are running?"
# Currently running: ollama, image-gen, chatterbox, dockge, uptime-kuma...
```

### Delegate a task (async)
```bash
./scripts/lobsterlan.sh delegate scotty "Generate a photorealistic wallpaper of a mountain lake at sunset, 3840x2160, push to the file share when done"
# Task delegated to scotty (async)
```

### List all peers
```bash
./scripts/lobsterlan.sh peers
# Self: spock (Spock) @ 192.168.1.2:18789
#
# scotty: Scotty @ 192.168.1.3:18789 [completions:✅ hooks:✅]
```

## From Within an OpenClaw Agent Session

### Agent asks another agent via exec
```bash
# In Spock's session — ask Scotty for system health
cd ~/.openclaw/workspace/skills/lobsterlan && scripts/lobsterlan.sh ask scotty "Run 'docker ps' and 'df -h /data' and summarise the health"
```

### Agent delegates a long-running task
```bash
# Spock delegates wallpaper generation to Scotty
cd ~/.openclaw/workspace/skills/lobsterlan && scripts/lobsterlan.sh delegate scotty "Generate 5 zen-themed wallpapers using dreamshaper model at 512x512, then upscale each to 4K. Save to /data/wallpapers/ and push to the Samba share."
```

## Practical Patterns

### Pattern 1: Spock as Chief of Staff, Scotty as Ops

```
User → Spock: "Generate some new wallpapers for the house"
Spock → Scotty (delegate): "Generate 5 cozy winter wallpapers, 4K..."
Scotty: [processes independently, takes 30 minutes]
Scotty → Spock (webhook callback): "Done. 5 wallpapers at /data/wallpapers/"
Spock → the user: "Scotty's finished — 5 new wallpapers on the file share 🎨"
```

### Pattern 2: Health Monitoring

```
Spock (heartbeat) → Scotty (ask): "Quick health check — CPU, RAM, disk, Docker"
Scotty → Spock: "CPU 42°C, RAM 38/61GB, Disk /data 34% used, all 6 containers healthy"
Spock: [logs to memory, alerts the user only if something is wrong]
```

### Pattern 3: Cross-Agent Research

```
Spock → Scotty (ask): "What Ollama models are currently loaded and what's their VRAM usage?"
Scotty → Spock: "qwen3:30b loaded (18.6GB), idle. No other models active."
Spock: [uses this info to decide whether to request a compute-heavy task]
```

## With Callback (Bidirectional Webhooks)

For Scotty to report results back to Spock after an async task, Scotty needs Spock's webhook endpoint configured:

```bash
# Scotty's task completion notifies Spock
curl -X POST http://192.168.1.2:18789/hooks/agent \
  -H 'Authorization: Bearer PEER_HOOKS_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "message": "Task complete: 5 wallpapers generated and pushed to file share. Files: zen-1.png through zen-5.png",
    "name": "scotty",
    "wakeMode": "now"
  }'
```

This triggers a system event in Spock's main session, which Spock can then relay to the user.
