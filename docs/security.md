# LobsterLAN Security Model

## Three-Layer Defense

### Layer 1: Network Isolation

LobsterLAN is designed for **trusted local networks only**. Gateway ports should never be exposed to the internet.

**Enforcement:**
- UFW/pf firewall rules restricting gateway port to LAN IPs
- `gateway.bind: "lan"` binds to LAN interface only (not 0.0.0.0)
- No port forwarding, no public exposure

**Risk if breached:** Attacker on LAN can attempt to authenticate. Mitigated by Layer 2.

### Layer 2: Gateway Authentication (Primary)

Every OpenClaw gateway requires a bearer token for API access. This is the **real security boundary**.

**Enforcement:**
- `gateway.auth.mode: "token"` (default)
- Token sent as `Authorization: Bearer <token>` header
- Rate limiting on failed auth attempts (429 response)
- Separate tokens for gateway API vs webhook hooks

**Risk if breached:** Attacker can interact with the agent as if they were a trusted peer. Mitigated by Layer 3.

### Layer 3: Agent Identity (Optional, Defense-in-Depth)

LobsterLAN adds an `X-LobsterLAN-Agent` header containing the sender's self-declared agent ID.

**Current implementation:** Header is sent but not enforced server-side. It's an audit trail.

**Future enhancement:** The receiving agent could check this header against `security.allowed_agents` in peers.json and reject unknown senders.

**Why it's optional:**
- On a trusted LAN with token auth, the token IS the identity proof
- Adding header verification doesn't add meaningful security unless tokens are compromised
- It does add a useful audit trail for logging which agent sent a request

## Token Management

### Generating Tokens
```bash
# Generate a strong random token
openssl rand -hex 32
```

### Token Separation
Use **different tokens** for:
- Gateway API access (`gateway.auth.token`)
- Webhook hooks (`hooks.token`)

This limits blast radius: a leaked hooks token can only trigger agent runs, not access the full control plane.

### Token Storage
- Tokens in `peers.json` — add to `.gitignore`
- Never commit tokens to version control
- Consider environment variables for CI/CD or automated deployment

## Threat Model

| Threat | Mitigation |
|--------|------------|
| Attacker on LAN | Gateway auth token required (Layer 2) |
| Token leak | Separate gateway/hooks tokens; rotate if compromised |
| Agent impersonation | X-LobsterLAN-Agent header for audit; token rotation |
| Man-in-the-middle | LAN traffic only; TLS available if needed (see below) |
| Replay attack | Gateway nonce/session handling; stateless calls are idempotent |

## Network Transport Security

OpenClaw gateways **refuse to start** if `bind` is set to `lan` without TLS. This is intentional — gateway tokens and chat data in plaintext on the network is a real risk, even on a home LAN.

Three secure transport options exist:

### SSH Tunnel ⭐ (Recommended)
- Both gateways stay on `bind: loopback`
- SSH tunnel encrypts all traffic between hosts
- Smallest attack surface — no ports exposed on LAN
- See [setup.md](setup.md#option-a-ssh-tunnel--recommended-for-simple-lans) for details

### Reverse Proxy with TLS
- Caddy/nginx terminates TLS, proxies to loopback gateway
- Requires cert management (Caddy's `tls internal` simplifies this)
- Good if you already have a reverse proxy in your stack

### Tailscale Serve
- Valid TLS certs via Tailscale's built-in CA
- Works across different networks (not just LAN)
- Requires Tailscale account and tailnet membership

**For simple home LANs, SSH tunneling is recommended.** It requires no configuration changes to the gateway, no TLS certificates, and provides encryption by default.

## Comparison to Channel Security

| Aspect | Telegram/Discord | LobsterLAN |
|--------|------------------|------------|
| Network path | Internet (Telegram servers) | LAN only |
| Auth | Platform account + bot token | Gateway bearer token |
| Encryption | TLS to platform servers | Optional TLS (LAN) |
| Latency | 200-500ms | 1-5ms |
| Rate limits | Platform-imposed | Self-managed |
| Audit trail | Platform logs | Local gateway logs |
| Identity | Platform user ID | Gateway token + agent header |

LobsterLAN wins on latency, control, and privacy. Channel-based communication wins on simplicity and built-in delivery guarantees.
