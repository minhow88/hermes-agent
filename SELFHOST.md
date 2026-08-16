# Self-Hosting Hermes Agent — Security & Privacy Guide

This document covers security hardening, privacy controls, and operational
guidance for self-hosting Hermes Agent on your own infrastructure.

## License

Hermes Agent is **MIT licensed** — you are free to use, modify, distribute,
and self-host it without restrictions. See [LICENSE](./LICENSE).

---

## Quick Start (Hardened)

```bash
# Build locally (don't pull from a registry if you want full control)
docker build -t hermes-agent:selfhost .

# Run with the hardened compose file
HERMES_UID=$(id -u) HERMES_GID=$(id -g) \
  docker compose -f docker-compose.selfhost.yml up -d
```

Access the dashboard via SSH tunnel (never expose directly):
```bash
ssh -L 9119:localhost:9119 your-server
# Then open http://localhost:9119 in your browser
```

---

## Privacy & Telemetry Summary

| Component | Default Behavior | Network Calls | How to Disable |
|-----------|-----------------|---------------|----------------|
| Update checker | Contacts `api.github.com` | Yes (GitHub API) | `HERMES_DISABLE_UPDATE_CHECK=1` |
| NeMo Relay plugin | **Disabled** (opt-in only) | No (local files) | Don't enable it |
| OTLP health export | **Disabled** by default | Only if operator sets endpoint | Don't set `monitoring.export.otlp.enabled` |
| Shared metrics | Local SQLite + JSON files | No | Already local-only |
| Skills Hub | Contacts GitHub API for skill search | Yes (when used) | Don't use `hermes skills search` |
| Lazy dependency installs | Contacts PyPI on first backend use | Yes (pip/uv) | `HERMES_DISABLE_LAZY_INSTALLS=1` |

### Disabling All Network Phone-Home

Set these environment variables to prevent any unsolicited outbound connections:

```bash
# In your .env or docker-compose environment:
HERMES_DISABLE_UPDATE_CHECK=1       # No GitHub API calls for update check
HERMES_DISABLE_LAZY_INSTALLS=1      # No PyPI calls for optional backends
```

The hardened docker-compose (`docker-compose.selfhost.yml`) sets these by default.

---

## Docker Compose Profiles

The `docker-compose.selfhost.yml` provides two profiles:

### Hardened (default) — Maximum security, minimal capabilities

```bash
docker compose -f docker-compose.selfhost.yml up -d
```

Drops ALL Linux capabilities and re-adds only 4 needed for the s6-overlay boot
sequence. The agent can still write files, run shell commands, and use Docker-in-Docker.

### Permissive — Full agent functionality, relaxed capabilities

```bash
docker compose -f docker-compose.selfhost.yml --profile permissive up -d
```

Keeps Docker's default capability set (minus the most dangerous ones like
`SYS_ADMIN` and `SYS_MODULE`). Use this when the agent's terminal tool needs
`ping`, raw sockets, `strace`, or other system-level tools.

### What works in each profile

| Functionality | Hardened | Permissive |
|---------------|:--------:|:----------:|
| Writing/reading skills & files | ✅ | ✅ |
| Running shell commands (ls, grep, python, git, etc.) | ✅ | ✅ |
| Docker-in-Docker terminal backend | ✅ | ✅ |
| All LLM calls, tools, cron, messaging | ✅ | ✅ |
| Dashboard & gateway | ✅ | ✅ |
| `ping` / raw sockets (NET_RAW) | ❌ | ✅ |
| `strace` / debugging (SYS_PTRACE) | ❌ | ✅ |
| Kill any process (KILL) | ❌ | ✅ |
| Bind ports < 1024 (NET_BIND_SERVICE) | ❌ | ✅ |
| Network interface changes (NET_ADMIN) | ❌ | ❌ |
| Mount filesystems (SYS_ADMIN) | ❌ | ❌ |
| Load kernel modules (SYS_MODULE) | ❌ | ❌ |

### Linux Capabilities Reference

Full table of all Linux capabilities, what they do, and their risk level:

| Capability | Description | Risk |
|------------|-------------|------|
| `AUDIT_CONTROL` | Enable/disable kernel auditing; change audit filter rules | High |
| `AUDIT_READ` | Read audit log via multicast netlink socket | Low |
| `AUDIT_WRITE` | Write records to kernel auditing log | Medium |
| `BLOCK_SUSPEND` | Employ features that can block system suspend | Low |
| `BPF` | Use bpf() syscall; create BPF maps/programs | High |
| `CHECKPOINT_RESTORE` | Checkpoint/restore (CRIU) operations | High |
| `CHOWN` ⭐ | Make arbitrary changes to file UIDs and GIDs | Medium |
| `DAC_OVERRIDE` ⭐ | Bypass file read/write/execute permission checks | High |
| `DAC_READ_SEARCH` | Bypass file read permission + directory read/execute | Medium |
| `FOWNER` | Bypass permission checks on file ops (chmod, etc.) | Medium |
| `FSETID` | Don't clear set-user/group-ID bits when file modified | Low |
| `IPC_LOCK` | Lock memory (mlock, mlockall, mmap, shmctl) | Low |
| `IPC_OWNER` | Bypass permission checks for IPC operations | Medium |
| `KILL` | Send signals to any process (bypass permission checks) | Medium |
| `LEASE` | Establish leases on arbitrary files (fcntl) | Low |
| `LINUX_IMMUTABLE` | Set FS_APPEND_FL and FS_IMMUTABLE_FL inode flags | Low |
| `MAC_ADMIN` | Override Mandatory Access Control (Smack/SELinux admin) | High |
| `MAC_OVERRIDE` | Override MAC policy; Smack label access | High |
| `MKNOD` | Create special files using mknod() | Medium |
| `NET_ADMIN` | Network admin (interfaces, firewall, routing, etc.) | High |
| `NET_BIND_SERVICE` | Bind a socket to privileged ports (below 1024) | Low |
| `NET_BROADCAST` | Make socket broadcasts and listen to multicasts | Low |
| `NET_RAW` | Use RAW and PACKET sockets (ping, raw networking) | Medium |
| `PERFMON` | Performance monitoring (perf_event_open, BPF) | Medium |
| `SETFCAP` | Set file capabilities (xattr) | High |
| `SETGID` ⭐ | Manipulate process GIDs; forge GID in socket creds | Medium |
| `SETPCAP` | Add/remove capabilities from the bounding set | High |
| `SETUID` ⭐ | Manipulate process UIDs; forge UID in socket creds | High |
| `SYS_ADMIN` | Sysadmin: mount, sethostname, quotactl, pivot_root, etc. | **Critical** |
| `SYS_BOOT` | Use reboot() and kexec_load() | **Critical** |
| `SYS_CHROOT` | Use chroot(); change mount namespaces | Medium |
| `SYS_MODULE` | Load and unload kernel modules | **Critical** |
| `SYS_NICE` | Set process scheduling priority; nice, ionice, affinity | Low |
| `SYS_PACCT` | Use acct() (process accounting on/off) | Low |
| `SYS_PTRACE` | Trace/inspect any process (ptrace, strace, gdb) | High |
| `SYS_RAWIO` | Perform I/O port operations (iopl, ioperm) | **Critical** |
| `SYS_RESOURCE` | Override resource limits (ulimits, disk quotas, etc.) | Medium |
| `SYS_TIME` | Set system clock; set real-time hardware clock | Medium |
| `SYS_TTY_CONFIG` | Use vhangup(); privileged virtual terminal ioctls | Low |
| `SYSLOG` | Privileged syslog(2) operations; view dmesg | Medium |
| `WAKE_ALARM` | Trigger something that will wake the system | Low |

⭐ = Included in the hardened profile (minimum for s6-overlay boot)

**Docker's default set** (what containers get without any `cap_drop`/`cap_add`):
`AUDIT_WRITE`, `CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `FSETID`, `KILL`, `MKNOD`,
`NET_BIND_SERVICE`, `NET_RAW`, `SETFCAP`, `SETGID`, `SETPCAP`, `SETUID`, `SYS_CHROOT`

---

## Security Hardening Checklist

### 1. Container Security

- [ ] Use `docker-compose.selfhost.yml` (applies all below automatically)
- [ ] Don't use `network_mode: host` (use explicit port binds)
- [ ] Drop all capabilities, add only what's needed (`CHOWN`, `SETUID`, `SETGID`, `DAC_OVERRIDE`)
- [ ] Set `no-new-privileges:true`
- [ ] Set memory/CPU limits to prevent resource exhaustion
- [ ] Use named Docker volumes (not host bind mounts with broad permissions)

### 2. Network Security

- [ ] Dashboard bound to `127.0.0.1` only — use SSH tunnels for remote access
- [ ] Never use `--insecure` flag on the dashboard in production
- [ ] Never set `API_SERVER_HOST=0.0.0.0` without a strong `API_SERVER_KEY`
- [ ] If exposing any port, put it behind a reverse proxy with TLS + auth
- [ ] Set `GATEWAY_ALLOW_ALL_USERS=false` (default)
- [ ] Configure explicit allowlists for any messaging platform adapter

### 3. Credential Management

- [ ] `.env` file permissions: `chmod 600 ~/.hermes/.env`
- [ ] Never commit `.env` to version control
- [ ] Use separate API keys with minimal permissions
- [ ] Rotate keys periodically
- [ ] For SSH terminal backend: use dedicated keys, not your personal SSH key

### 4. Access Control

- [ ] Set per-platform allowlists (`TELEGRAM_ALLOWED_USERS`, `DISCORD_ALLOWED_USERS`, etc.)
- [ ] Never set `GATEWAY_ALLOW_ALL_USERS=true` on internet-facing deployments
- [ ] Use DM pairing mode for new user onboarding (most restrictive)
- [ ] Review and approve skills before installation (`hermes skills` shows what's loaded)

### 5. Data Isolation

- [ ] Use the Docker/SSH terminal backend instead of local (isolates agent shell access)
- [ ] Set `HERMES_WRITE_SAFE_ROOT` to restrict file write operations
- [ ] Keep the data volume separate from your host home directory
- [ ] Back up `~/.hermes/` (or `/opt/data` in Docker) — it contains session history

---

## Recommended Reverse Proxy Setup

If you must expose the dashboard remotely, use a reverse proxy with authentication:

### Nginx + Basic Auth Example

```nginx
server {
    listen 443 ssl;
    server_name hermes.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/hermes.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/hermes.yourdomain.com/privkey.pem;

    auth_basic "Hermes Agent";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://127.0.0.1:9119;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # WebSocket support (for live dashboard updates)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Tailscale / WireGuard (Recommended)

The simplest secure remote access is a VPN:
```bash
# On your server
tailscale up

# Access dashboard at http://<tailscale-ip>:9119
# No port exposure to the public internet needed
```

---

## Air-Gapped Deployment

For fully air-gapped environments:

1. Build the Docker image on an internet-connected machine:
   ```bash
   docker build -t hermes-agent:selfhost .
   docker save hermes-agent:selfhost | gzip > hermes-agent.tar.gz
   ```

2. Transfer to the air-gapped host and load:
   ```bash
   docker load < hermes-agent.tar.gz
   ```

3. All optional backends are baked into the Docker image (the `[all]` extra),
   so no runtime PyPI access is needed for core functionality.

4. Set both disable flags:
   ```bash
   HERMES_DISABLE_UPDATE_CHECK=1
   HERMES_DISABLE_LAZY_INSTALLS=1
   ```

---

## Monitoring (Optional, Self-Hosted)

If you want observability without sending data externally, deploy a local
OpenTelemetry Collector + Grafana stack and point Hermes at it:

```yaml
# In config.yaml under monitoring:
monitoring:
  gateway_health_export:
    enabled: true
  export:
    otlp:
      enabled: true
      endpoint: "http://localhost:4318"  # Your local OTEL collector
```

This keeps all telemetry data on your own infrastructure.

---

## Updates

With update checking disabled, check for updates manually:
```bash
# From your host (not inside the container)
git -C /path/to/hermes-agent fetch origin main
git -C /path/to/hermes-agent log --oneline HEAD..origin/main
```

Or check the GitHub releases page: https://github.com/NousResearch/Hermes-Agent/releases
