# Gated Container Updates (via Sentinel Bot)

> **Internal Documentation** - Contains Discord bot configuration details.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [07 - Deployed Services](07%20-%20Deployed%20Services.md) | [24 - Discord Bots](24%20-%20Discord%20Bots.md) | [12 - Troubleshooting](12%20-%20Troubleshooting.md)

---

## Overview


Sentinel checks all tracked containers for available image updates via SSH + `docker pull`, then requires **Discord reaction approval** before updating anything. Nothing updates automatically without a 👍.

| Setting | Value |
|---------|-------|
| Mechanism | SSH `docker pull` diff check (not Docker Hub webhooks) |
| Approval | Discord reaction (👍 all / numbered per-container) |
| Auto-check schedule | Daily at 7:00 PM (`scheduler.py` → `daily_update_report`) |
| Manual check | `/check` slash command |
| Manual update | `/update <container>` slash command |
| Config | `/opt/sentinel-bot/config.py` on docker-vm-core-utilities01 (192.168.40.13) |

---

## Architecture

```
Sentinel Bot (docker-vm-core-utilities01, cogs/updates.py + cogs/scheduler.py)
        │
        │  SSH (hermes-admin@homelab_ed25519) to each host in CONTAINER_HOSTS
        ▼
docker inspect <container> → docker pull <image> → compare digest
        │
        ▼
Discord #container-updates channel
  "🆕 Update available for sonarr"
  👍 → runs `docker compose pull && docker compose up -d`
       in COMPOSE_DIRS[container] over SSH
  👎 → skip
```

This replaces the older Watchtower+webhook design entirely — there is no `WATCHTOWER_NOTIFICATION_URL` or separate webhook receiver anymore for LXC hosts.

---

## Discord Bot Commands

| Command | Description |
|---------|-------------|
| `/check` | Scan all tracked containers for available updates |
| `/update <container>` | Update one specific container |
| `help` | Show available commands |

### Notification Format

**Update Available:**
```
🆕 There is a new update available for **sonarr**
📦 Current: `current digest`
🚀 New: `lscr.io/linuxserver/sonarr (new digest)`

React with 👍 to update or 👎 to skip!
```

---

## Coverage (as of 2026-08-29)

`CONTAINER_HOSTS` / `COMPOSE_DIRS` in `config.py` were corrected 2026-08-29 to match the real infrastructure (see [Homelab Changelog](Homelab%20Changelog.md) for the full list of path fixes). Currently tracked:

- **docker-vm-core-utilities01** (.13): grafana, prometheus, uptime-kuma, speedtest-tracker, n8n, jaeger, cadvisor, snmp-exporter, pve-exporter, sentinel-bot, karakeep, tracearr, paperless, pbs-exporter, and others
- **docker-lxc-media** (.11): full arr-stack (jellyfin, radarr, sonarr, lidarr, prowlarr, bazarr, jellyseerr, overseerr, tdarr, autobrr, deluge, sabnzbd), metube
- **traefik-lxc** (.20), **authentik-lxc** (.21): traefik, authentik-server, authentik-worker
- **homeassistant-lxc** (.25): homeassistant, matter-server
- **homepage-lxc** (.27): homepage
- **chronicle-lxc** (.15): homelab-chronicle
- **immich-vm01** (.22): immich-server, immich-ml



---

## Common Commands

```bash
# Check Sentinel bot status
ssh docker-vm-core-utilities01 "docker ps --filter name=sentinel-bot"

# View logs
ssh docker-vm-core-utilities01 "docker logs sentinel-bot --tail 50"

# Restart after a config.py change
ssh docker-vm-core-utilities01 "cd /opt/sentinel-bot && docker compose restart"
```

---

## Related Documentation

- [24 - Discord Bots](24%20-%20Discord%20Bots.md) - Full Sentinel architecture and cog list
- [07 - Deployed Services](07%20-%20Deployed%20Services.md) - All services
- [12 - Troubleshooting](12%20-%20Troubleshooting.md) - Issue resolutions
