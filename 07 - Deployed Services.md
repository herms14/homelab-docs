---
updated: 2026-10-19
---
# Deployed Services

> **Internal Documentation** - Contains service URLs and access details.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [08 - Arr Media Stack](08%20-%20Arr%20Media%20Stack.md) | [09 - Traefik Reverse Proxy](09%20-%20Traefik%20Reverse%20Proxy.md)

---

## Service Overview

Docker services are deployed via Docker Compose, managed by Ansible from ansible-controller01. The Roundtable portal is a native systemd service on codex-agent-lxc.

| Category | Host | Services |
|----------|------|----------|
| Reverse Proxy | traefik-lxc (LXC 203, 192.168.40.20) | Traefik (with OTEL tracing) |
| Identity | authentik-lxc (LXC 204, 192.168.40.21) | Authentik |
| Photos | immich-vm01 | Immich |
| DevOps | gitlab-vm01 | GitLab CE |
| Media | docker-lxc-media (LXC 205, 192.168.40.11) | Arr Stack (14 services) |
| Smart Home | homeassistant-lxc (LXC 206, 192.168.40.25) | Home Assistant |
| Dashboard | docker-lxc-glance (LXC 200, 192.168.40.12) | Glance Dashboard, Media Stats API, Reddit Manager |
| Utilities | docker-vm-core-utilities01 | n8n, Paperless, Life Progress API, Speedtest Tracker |
| Productivity | docker-vm-core-utilities01 | BentoPDF, Reactive Resume, Karakeep |
| Media Tools | docker-vm-core-utilities01 | Wizarr, Tracearr |
| Monitoring | docker-vm-core-utilities01 | Uptime Kuma, Prometheus, Grafana |
| Observability | docker-vm-core-utilities01 | OTEL Collector, Jaeger, Demo App |
| Update Management | docker-vm-core-utilities01 | Sentinel bot (SSH-based checks + Discord reaction approval — see [19 - Watchtower Updates](19%20-%20Watchtower%20Updates.md)) |
| Discord Bots | docker-vm-core-utilities01 | Sentinel Bot (consolidated) |
| Container Metrics | Both Docker hosts | Docker Stats Exporter |
| Dashboard (secondary) | homepage-lxc (LXC 209, 192.168.40.27) | Homepage |
| Changelog Tracker | chronicle-lxc (LXC 207, 192.168.40.15) | Homelab Chronicle |
| AI Agent Portal | codex-agent-lxc (LXC 211, 192.168.40.16) | Roundtable (`olympus-portal.service`, native Python) |
| Study Notebook | study-lxc (LXC 212, 192.168.40.38) | Study (`study.service`, FastAPI on :8090; vault sync `study-sync.timer`) |
| Secrets Management | <redacted> (LXC 213, 192.168.40.39) | OpenBao 2.7.1 (`openbao.service` :8200 TLS, `openbao-unseal.service`, audit summary timer); UI https://bao.hrmsmrflrii.xyz (LAN/Tailscale only) — see [65 - OpenBao Secrets Management](65%20-%20OpenBao%20Secrets%20Management.md) |
| Network Observability | docker-vm-core-utilities01 | VLAN Flow (`vlan-flow.service`, native Python) — Omada ACL decision map + syslog feed |

> **Note**: Traefik, Authentik, and Media services migrated from VMs to LXC containers on January 7, 2026. See [30 - LXC Migration Tutorial](30%20-%20LXC%20Migration%20Tutorial.md) for details.

---

## Service URLs (HTTPS)

> [!info] Internet access (2026-10-19)
> Roundtable (+ Lab Atlas), Glance, VLAN Flow, Immich (`photos`), Paperless, Study and the Authentik sign-in page are also reachable from the internet. They go through Cloudflare Tunnel `homelab-public` (connector on CT203), and Cloudflare Access lets in only <email> through Google sign-in. Every other URL below is LAN/Tailscale only. See [09 - Traefik Reverse Proxy › Internet access (Cloudflare Tunnel + Access) - 2026-10-19](09%20-%20Traefik%20Reverse%20Proxy.md#internet-access-cloudflare-tunnel--access---2026-10-19).

### Infrastructure

| Service | URL | Backend |
|---------|-----|---------|
| Proxmox Cluster | https://proxmox.hrmsmrflrii.xyz | 192.168.20.21:8006 |
| Proxmox Node01 | https://node01.hrmsmrflrii.xyz | 192.168.20.20:8006 |
| Proxmox Node02 | https://node02.hrmsmrflrii.xyz | 192.168.20.21:8006 |
| Proxmox Node03 | https://node03.hrmsmrflrii.xyz | 192.168.20.22:8006 |
| Traefik Dashboard | https://traefik.hrmsmrflrii.xyz | localhost:8080 |
| Pi-hole | https://pihole.hrmsmrflrii.xyz | 192.168.90.53:80 |
| Omada Controller | https://omada.hrmsmrflrii.xyz | 192.168.0.101:443 (controller 6.3.0.45; was wrongly .103, another TP-Link device, until 2026-10-19). Login page `/independent/index.html#login` |
| Synology NAS (DSM) | https://nas.hrmsmrflrii.xyz (LAN/Tailscale only, added 2026-10-19) | 192.168.20.32:5001 (self-signed, not verified) |

### Core Services

| Service | URL | Backend |
|---------|-----|---------|
| Authentik (SSO) | https://auth.hrmsmrflrii.xyz | 192.168.40.21:9000 |
| Immich (Photos) | https://photos.hrmsmrflrii.xyz | 192.168.40.22:2283 |
| GitLab | https://gitlab.hrmsmrflrii.xyz | 192.168.40.23:80 |
| Home Assistant | https://ha.hrmsmrflrii.xyz | 192.168.40.25:8123 |
| Homepage | https://homepage.hrmsmrflrii.xyz | 192.168.40.27:3000 |
| Homelab Chronicle | https://chronicle.hrmsmrflrii.xyz | 192.168.40.15:3000 |
| Roundtable Agent Portal | https://roundtable.hrmsmrflrii.xyz | 192.168.40.16:8088 |
| Hermes Lab Atlas (diagrams, portal route `/atlas`; kept live by Miriel `atlas.py` + `atlas-sweep.timer`) | https://roundtable.hrmsmrflrii.xyz/atlas | 192.168.40.16:8088/atlas |
| Roundtable Overview (default page; all lab links + live status, `/api/links`; themes via header ◐ picker or `?theme=bento\|mini\|frost\|pastel\|glass`) | https://roundtable.hrmsmrflrii.xyz/ | 192.168.40.16:8088/ |
| Study Notebook (Authentik app `study`, provider 50) | https://study.hrmsmrflrii.xyz | 192.168.40.38:8090 |
| VLAN Flow (Authentik app `vlanflow`, provider 49) | https://vlanflow.hrmsmrflrii.xyz | 192.168.40.13:8787 |

Roundtable's 2026-10-11 portal updates add a searchable voice picker, device-local microphone settings (system default, specific input, local-only test and live replacement), and browser-only 2.5D avatar depth, lighting, reactive motion and a no-call character preview. New native macOS/Windows app work is deferred; browser pop-out still depends on supported browsers and an open originating tab. The portal remains behind Traefik/Authentik. See portal, avatar and microphone guide.

The later character/portal release adds **29 stable fictional profiles**; homelab agents retain their Elden Ring
identities. The owner's subsequent artwork correction restores **22 exact previous avatar selections** and
SVG portrait fallback. Recorded image-usage estimate is $0.887035, not a verified provider invoice.
Original restoration completed (#87); the later owner-requested direct-reference correction is deployed
with fresh `-matched.jpg` / `-matched-full.png` URLs. All 29 current Discord portraits verified and correction
audit #102 closed; polished original files unchanged. Correction image-usage estimate $3.771315, not an invoice.
**Dependency map** (`/#map`) remains distinct from hierarchy, with reporting overlay, visible failures and retry.
**Listen now** restores paused live input; intentional speaker playback is not reported as microphone failure.
The raw HTTP/IP URL is **view-only**: voice needs the signed-in HTTPS portal. No DNS, SSO or corporate-control
changes; real hardware and the corporate browser were not available for acceptance.
Original character/portal backup: `~/olympus/backups/characters-portal-20261011-0d0193cf/`.
Artwork-restoration source/SQLite backup and manifest: `~/olympus/backups/avatar-restore-20261011-0d0193cf/`.

**Hierarchy status (2026-10-11): fully deployed after explicit owner authorization.** 29 catalog agents,
29 reporting agents and four leads: Gideon (COO/EA), Loretta, Ranni and Corhyn. Morgott is PM under Ranni;
Hewg owns patch delivery through the unchanged SRE executor, with Corhyn accountable for oversight.
address allocation or network change in that rollout. The subsequent IPAM safeguard repair replaces the
overlapping service pool with `.36-.49` candidates outside unchanged DHCP, and adds complete controller
reservation/history/binding reads plus Proxmox interface evidence. The initial untagged row is confirmed
internal mesh backhaul using complete history and reciprocal live AP topology, with both AP IPs excluded.
All three pools now have 14 candidates; the real allocator passed with live evidence in an isolated ledger.
No production reservation or new service was created. Existing service IPs and leases remain unchanged.
Live hierarchy and capabilities;

**Hierarchy page (fully active):** **Agents → Hierarchy** (`/#hierarchy`) displays 29 reporting agents/four leads,
active profile links and no pending badges. `/api/hierarchy` is `configured`, reading the live registry; the former
page-only pending snapshot is backed up and removed. Direct LAN: `http://192.168.40.16:8088/#hierarchy`.
Current portal/registry/source hashes are recorded in the latest matched-art/teaching release manifest;
the preceding personal-reporting release has its own historical manifest.
Personal-reporting backup: `~/olympus/backups/personal-reporting-20261011-0d0193cf/`;
Full code/unit/SQLite backup: `~/olympus/backups/roundtable-20261011-0d0193cf-full/`; rollout audit GitLab #62.
Original rollout browser/API/service/channel/boundary/backlog acceptance completed, including a shared-incident evidence fix

**Teaching/persona release (2026-10-11):** **Voice → Learn in a real terminal** opens a Mac/Windows
Opening it does not start paid voice. Dedicated teaching calls have no hidden ordinary execution tools.
Writes retain owner-only Discord approval/exact verification/rollback/documentation; Hewg's visual configuration
exercise is limited to a new fixed `/var/lib/roundtable-learning` directory, while other builds/patches use
existing reviewed workflows. No unrestricted shell or local Terminal/PowerShell helper.
and converses naturally without repeated AI boilerplate; direct identity questions are answered honestly.
Eight owned sources/HTML/registry and fourteen new artwork files deployed after idle/hash guards and
source/SQLite backup; only boss/portal restarted. Actual authenticated read-only kernel lesson accepted;
no infrastructure test write or paid voice test. Source suite: 73 tests, 201 parent-browser checks,
20 terminal browser checks and native fake-input coverage. Real physical Mac/Windows voice remains unverified.
Backup/manifest/acceptance: `~/olympus/backups/matched-teaching-20261011-0d0193cf/`.
Final `acceptance.json` is **complete**: original restoration #87, personal direct reporting #95,
and recovery boundaries are listed in the current release reference.
The earlier pending page, initial IPAM blocker and Discord cooldown are historical milestones, not active
release blockers. No directory/configuration was created merely to test teaching.


### Media Services

| Service | URL | Backend |
|---------|-----|---------|
| Jellyfin | https://jellyfin.hrmsmrflrii.xyz | 192.168.40.11:8096 |
| Radarr | https://radarr.hrmsmrflrii.xyz | 192.168.40.11:7878 |
| Sonarr | https://sonarr.hrmsmrflrii.xyz | 192.168.40.11:8989 |
| Lidarr | https://lidarr.hrmsmrflrii.xyz | 192.168.40.11:8686 |
| Prowlarr | https://prowlarr.hrmsmrflrii.xyz | 192.168.40.11:9696 |
| Bazarr | https://bazarr.hrmsmrflrii.xyz | 192.168.40.11:6767 |
| Overseerr | https://overseerr.hrmsmrflrii.xyz | 192.168.40.11:5055 |
| Jellyseerr | https://jellyseerr.hrmsmrflrii.xyz | 192.168.40.11:5056 |
| Tdarr | https://tdarr.hrmsmrflrii.xyz | 192.168.40.11:8265 |
| Autobrr | https://autobrr.hrmsmrflrii.xyz | 192.168.40.11:7474 |
| Deluge | https://deluge.hrmsmrflrii.xyz | 192.168.40.11:8112 |
| SABnzbd (idle: torrent-only since 2026-10-16) | https://sabnzbd.hrmsmrflrii.xyz | 192.168.40.11:8081 |
| MeTube | https://metube.hrmsmrflrii.xyz | 192.168.40.11:8082 |

### Utility Services

| Service           | URL                               | Backend            |
| ----------------- | --------------------------------- | ------------------ |
| Paperless-ngx     | https://paperless.hrmsmrflrii.xyz | 192.168.40.13:8000 |
| Glance Dashboard  | https://glance.hrmsmrflrii.xyz    | 192.168.40.12:8080 |
| n8n Automation    | https://n8n.hrmsmrflrii.xyz       | 192.168.40.13:5678 |
| Speedtest Tracker | https://speedtest.hrmsmrflrii.xyz | 192.168.40.13:3000 |

### Productivity & Tools

| Service | URL | Backend |
|---------|-----|---------|
| BentoPDF | https://bentopdf.hrmsmrflrii.xyz | 192.168.40.13:5055 |
| Reactive Resume | https://resume.hrmsmrflrii.xyz | 192.168.40.13:5057 |
| Karakeep | https://karakeep.hrmsmrflrii.xyz | 192.168.40.13:3005 |

### Media Tools

| Service | URL | Backend |
|---------|-----|---------|
| Wizarr | https://wizarr.hrmsmrflrii.xyz | 192.168.40.13:5690 |
| Tracearr | https://tracearr.hrmsmrflrii.xyz | 192.168.40.13:3002 |

### Monitoring Services

| Service | URL | Backend |
|---------|-----|---------|
| Uptime Kuma | https://uptime.hrmsmrflrii.xyz | 192.168.40.13:3001 |
| Prometheus | https://prometheus.hrmsmrflrii.xyz | 192.168.40.13:9090 |
| Grafana | https://grafana.hrmsmrflrii.xyz | 192.168.40.13:3030 |

### Observability Services

| Service | URL | Backend |
|---------|-----|---------|
| Jaeger | https://jaeger.hrmsmrflrii.xyz | 192.168.40.13:16686 |
| OTEL Collector | Internal (gRPC/HTTP) | 192.168.40.13:4317/4318 |
| Demo App | https://demo.hrmsmrflrii.xyz | 192.168.40.13:8080 |

---

## Traefik Reverse Proxy

**Host**: traefik-lxc, CT203 (192.168.40.20)
**Version**: v3.7.12 (upgraded 2026-08-29)
**Status**: Migrated to LXC January 7, 2026

See [09 - Traefik Reverse Proxy](09%20-%20Traefik%20Reverse%20Proxy.md) for detailed configuration.

---

## Authentik Identity Provider

**Host**: authentik-lxc, CT204 (192.168.40.21)
**Version**: 2026.8.0 (upgraded 2026-08-29)
**Status**: Migrated to LXC January 7, 2026

| Port | URL | Purpose |
|------|-----|---------|
| 9000 | http://192.168.40.21:9000 | Web interface & API |
| 9443 | https://192.168.40.21:9443 | Secure web interface |

### Initial Setup

1. Navigate to http://192.168.40.21:9000/if/flow/initial-setup/
2. Create admin account (default username: `akadmin`)

---

## Immich Photo Management

**Host**: immich-vm01 (192.168.40.22)
**Status**: Deployed December 19, 2025
**Version**: v3.3.0 (updated 2026-10-10 after failed-rollback recovery; server + ML track `:release` — v3.3.1 pending SRE approval — see [57 - Immich VectorChord Migration and v3 Upgrade](57%20-%20Immich%20VectorChord%20Migration%20and%20v3%20Upgrade.md))

| Port | URL | Purpose |
|------|-----|---------|
| 2283 | http://192.168.40.22:2283 | Web interface & API |

### Mobile App Setup

- Server URL: `http://192.168.40.22:2283/api`

---

## GitLab CE DevOps Platform

**Host**: gitlab-vm01 (192.168.40.23)
**Status**: Deployed December 19, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 80 | http://192.168.40.23 | Web UI |
| 443 | https://192.168.40.23 | Secure web |
| 2222 | ssh://git@192.168.40.23:2222 | Git SSH |

### Initial Setup

Get initial root password:
```bash
ssh hermes-admin@192.168.40.23 "sudo docker exec gitlab grep 'Password:' /etc/gitlab/initial_root_password"
```

> Password file deleted after 24 hours!

---

## n8n Workflow Automation

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 19, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 5678 | http://192.168.40.13:5678 | Workflow editor |

---

## Speedtest Tracker

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 22, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 3000 | http://192.168.40.13:3000 | Web interface |

### Features

- Scheduled speed tests (Ookla Speedtest CLI)
- Historical results with graphs
- Download, upload, and ping tracking
- Multi-server support
- Dark theme interface
- SQLite database for lightweight storage

### Schedule

Tests run automatically every 6 hours (configurable via `SPEEDTEST_SCHEDULE` cron expression).

### Initial Setup

1. Navigate to https://speedtest.hrmsmrflrii.xyz
2. Default login: `<email>` / `password`
3. Change credentials immediately in Settings

### Management

```bash
# View logs
ssh hermes-admin@192.168.40.13 "docker logs speedtest-tracker"

# Trigger manual test
ssh hermes-admin@192.168.40.13 "docker exec speedtest-tracker php artisan app:ookla-speedtest"
```

**GitHub**: https://github.com/alexjustesen/speedtest-tracker

---

## Glance Dashboard

**Host**: lxc-glance (192.168.40.12) - LXC Container 200
**Status**: Migrated to LXC December 30, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 8080 | http://192.168.40.12:8080 | Dashboard |

Data API: **Lab Feed** on codex-agent-lxc :8790 (`labfeed.service`, token-protected read-only JSON for Glance and the agents). Redesigned 2026-10-18; see [23 - Glance Dashboard › 2026-10-18 Redesign (live)](23%20-%20Glance%20Dashboard.md#2026-10-18-redesign-live).

> **Note**: Glance runs on an LXC container with Docker. The docker-compose.yml requires `security_opt: apparmor=unconfined` due to AppArmor restrictions in LXC environments.

### Features

- Service health monitoring (all Proxmox nodes, K8s cluster)
- Stock market tracking (BTC, MSFT, AAPL, SPY)
- RSS feeds (r/homelab, r/selfhosted)
- Network device status
- Media stack monitoring
- **Life Progress Widget** - Year/Month/Day/Life progress bars with daily quotes

### Configuration

Config: `/opt/glance/config/glance.yml`
Assets: `/opt/glance/assets/`

> **Note**: Glance v0.7.0+ requires config directory mount (`./config:/app/config`), not single file mount.

---

## Life Progress API

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 22, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 5051 | http://192.168.40.13:5051/progress | Progress API |

### Features

- Calculates year/month/day/life progress percentages
- Serves daily motivational quotes (30 quotes, rotates by day)
- Used by Glance dashboard widget

Compose healthcheck uses Python's standard-library urllib for an IPv4 `/health` request; the previous `curl` probe referenced a tool absent from the image. Compose pins the pre-repair running image by ID.

### Configuration

Config: `/opt/life-progress/app.py`
- `BIRTH_DATE`: February 14, 1989
- `TARGET_AGE`: 75 years

**GitHub**: https://github.com/herms14/life-progress-api

See [21 - Application Configurations](21%20-%20Application%20Configurations.md) for detailed setup and customization.

---

## Reddit Manager

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 22, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 5053 | http://192.168.40.13:5053 | Management UI & API |

### Purpose

Flask API that provides dynamic subreddit management for Glance dashboard. Allows adding/removing subreddits via web UI and fetches Reddit posts with thumbnails.

### Features

- Dynamic subreddit management (add/remove via UI)
- Thumbnail support for posts with images
- Grouped view by subreddit with headers
- Sort options: Hot, New, Top
- Parallel fetching for fast response (~2 seconds)
- 5-minute caching to reduce Reddit API calls

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Management UI |
| GET | `/api/subreddits` | List subreddits |
| POST | `/api/subreddits` | Add subreddit |
| DELETE | `/api/subreddits/<name>` | Remove subreddit |
| GET | `/api/settings` | Get sort/view settings |
| POST | `/api/settings` | Update settings |
| GET | `/api/feed` | Get feed (grouped or combined) |

### Default Subreddits

homelab, selfhosted, linux, devops, kubernetes, docker

### Troubleshooting

**Timeout errors in Glance**: Reddit Manager uses parallel fetching. Response should be ~2 seconds. Test with:
```bash
time curl http://192.168.40.13:5053/api/feed
```

**Ansible**: `~/ansible/reddit-manager/deploy-reddit-manager.yml`

---

## Monitoring Stack

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 21, 2025

Complete monitoring solution with uptime monitoring, metrics collection, and visualization.

| Service | Port | URL | Purpose |
|---------|------|-----|---------|
| Uptime Kuma | 3001 | http://192.168.40.13:3001 | Uptime monitoring & status pages |
| Prometheus | 9090 | http://192.168.40.13:9090 | Metrics collection & time-series DB |
| Grafana | 3030 | http://192.168.40.13:3030 | Metrics visualization & dashboards |

### Initial Setup

**Uptime Kuma**:
1. Navigate to https://uptime.hrmsmrflrii.xyz
2. Create admin account on first access

**Grafana**:
1. Navigate to https://grafana.hrmsmrflrii.xyz
2. Login: `admin` / `admin`
3. Change password immediately
4. Add Prometheus data source: `http://192.168.40.13:9090`

See [17 - Monitoring Stack](17%20-%20Monitoring%20Stack.md) for detailed configuration.

---

## Observability Stack

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 21, 2025

Full OpenTelemetry distributed tracing with Jaeger visualization.

| Service | Port | URL | Purpose |
|---------|------|-----|---------|
| OTEL Collector | 4317/4318 | Internal | Trace receiver/processor |
| Jaeger | 16686 | http://192.168.40.13:16686 | Distributed tracing UI |
| Demo App | 8080 | http://192.168.40.13:8080 | OTEL testing application |

### Trace Flow

```
Traefik → OTEL Collector → Jaeger → Grafana
```

See [18 - Observability Stack](18%20-%20Observability%20Stack.md) for detailed configuration.

---

## Sentinel Discord Bot

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Live — consolidated bot, replaced the old separate Argus/Chronos/Athena/Mnemosyne bots in January 2026

All homelab Discord slash commands (Proxmox status, VM/container control, media requests, update checks, GitLab issues, Claude task queue) now live in one bot. See [24 - Discord Bots](24%20-%20Discord%20Bots.md) for the full command list and cog architecture.

Media requests: `/download <title>` in `#cinema-lounge` (was `#media-downloads`, archived 2026-10-18) adds the title straight to Radarr/Sonarr (profile `Any`, season picker for TV), starts the search and reports back if nothing was found, with Keep watching / Remove buttons. Rebuilt 2026-10-07.

---

## Download Monitor

**Host**: docker-lxc-media (192.168.40.11)
**Status**: Deployed December 22, 2025

| Port | Purpose |
|------|---------|
| 5052 | Flask webhook receiver |

### Features

- Real-time download completion notifications
- Poster images embedded in Discord messages
- Supports Radarr (movies) and Sonarr (TV shows)

### Discord Channel

- **Channel**: `#media-downloads` (archived 2026-10-18; the monitor is not running - no container or :5052 listener)

### Radarr/Sonarr Configuration

Add webhook:
- URL: `http://download-monitor:5052/webhook/radarr` or `/webhook/sonarr`
- Trigger: On Download / On Upgrade

---

## MeTube

**Host**: docker-lxc-media (192.168.40.11)
**Status**: Deployed January 2, 2026

| Port | Purpose |
|------|---------|
| 8082 | Web UI |

### Features

- YouTube video/playlist downloader
- Web-based interface - paste URL and download
- Automatic playlist folder organization
- English subtitles auto-download
- MP4 output format (merged)

### Configuration

- Config: `/opt/metube/docker-compose.yml`
- Download Path: `/mnt/media/YouTube Videos` (NAS)
- URL: https://metube.hrmsmrflrii.xyz

### Usage

1. Open https://metube.hrmsmrflrii.xyz
2. Paste YouTube video or playlist URL
3. Click Add
4. Videos download to NAS automatically

---

## YouTube Stats API

**Host**: docker-lxc-media (192.168.40.11)
**Status**: Deployed January 2, 2026

| Port | Purpose |
|------|---------|
| 5060 | REST API |

### Features

- Provides YouTube download statistics for Glance dashboard
- Scans MeTube download folder
- Groups videos by playlist folders

### Endpoints

| Endpoint | Description |
|----------|-------------|
| `/api/stats` | Total videos, size, download status |
| `/api/recent` | Last 10 downloaded videos |
| `/api/queue` | Current download queue |
| `/api/playlists` | Videos grouped by playlist |
| `/health` | Health check |

### Glance Integration

The Media page displays:
- YouTube Downloads widget (stats)
- Recent YouTube Videos widget (last downloads)

---

## Docker Stats Exporter

**Hosts**: docker-vm-core-utilities01, docker-lxc-media
**Status**: Deployed December 22, 2025

| Port | Purpose |
|------|---------|
| 9417 | Prometheus metrics endpoint |

### Metrics

- `container_cpu_usage_percent`
- `container_memory_usage_bytes`
- `container_network_rx_bytes` / `container_network_tx_bytes`
- `container_running` (1/0)

### Grafana Dashboard

Container metrics displayed in "Container Monitoring" dashboard.

---

## Karakeep (AI Bookmark Manager)

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 30, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 3005 | http://192.168.40.13:3005 | Web interface |

### Features

- AI-powered bookmark tagging and organization
- Full-text search with Meilisearch
- Automatic screenshot capture via headless Chrome
- Browser extension support
- RSS feed generation

### Architecture

Karakeep requires multiple containers:

| Container | Purpose |
|-----------|---------|
| karakeep | Main application (Next.js) |
| karakeep-meilisearch | Full-text search engine |
| karakeep-chrome | Headless Chrome for screenshots |

### Management

```bash
# View logs
ssh hermes-admin@192.168.40.13 "docker logs karakeep"

# Restart all containers
ssh hermes-admin@192.168.40.13 "cd /opt/karakeep && docker compose restart"
```

**Documentation**: https://docs.karakeep.app

---

## Wizarr (Media Server Invitations)

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 30, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 5690 | http://192.168.40.13:5690 | Web interface |

### Features

- Generate invitation links for Jellyfin/Plex users
- Customizable user onboarding flow
- Automatic library access configuration
- Expiring invitations with usage limits
- Discord integration for notifications

### Integration with Jellyfin

Wizarr connects to Jellyfin at `http://192.168.40.11:8096` to create users and assign library permissions.

### Management

```bash
# View logs
ssh hermes-admin@192.168.40.13 "docker logs wizarr"

# Restart
ssh hermes-admin@192.168.40.13 "cd /opt/wizarr && docker compose restart"
```

**Documentation**: https://docs.wizarr.dev

Compose healthcheck uses an IPv4 `wget` GET probe; the original probe was incompatible with the runtime image/address binding. Compose pins the pre-repair running image by ID.

---

## Tracearr (Media Tracking)

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 30, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 3002 | http://192.168.40.13:3002 | Web interface |

### Features

- Media tracking and statistics for Jellyfin
- Watch history and analytics
- User activity monitoring
- Media consumption reports

### Management

```bash
# View logs
ssh hermes-admin@192.168.40.13 "docker logs tracearr"

# Restart
ssh hermes-admin@192.168.40.13 "cd /opt/tracearr && docker compose restart"
```

---

## Paperless-ngx (Document Management)

**Host**: docker-vm-core-utilities01 (192.168.40.13)
**Status**: Deployed December 30, 2025

| Port | URL | Purpose |
|------|-----|---------|
| 8000 | http://192.168.40.13:8000 | Web interface |

### Features

- Automatic document scanning and OCR
- Full-text search across all documents
- Tagging and categorization
- Correspondent and document type management
- Email consumption for automatic import

### Architecture

Paperless requires Redis for caching and task queue:

| Container | Purpose |
|-----------|---------|
| paperless | Main application (Django) |
| paperless-redis | Redis cache and task queue |

### Folder Structure

| Path | Purpose |
|------|---------|
| `/opt/paperless/data` | Database and settings |
| `/opt/paperless/media` | Processed documents |
| `/opt/paperless/consume` | Drop folder for new documents |
| `/opt/paperless/export` | Backup export folder |

### Management

```bash
# View logs
ssh hermes-admin@192.168.40.13 "docker logs paperless"

# Create superuser
ssh hermes-admin@192.168.40.13 "docker exec -it paperless python3 manage.py createsuperuser"

# Restart
ssh hermes-admin@192.168.40.13 "cd /opt/paperless && docker compose restart"
```

---

## Homepage Dashboard

**Host**: homepage-lxc, CT209 (192.168.40.27)
**Status**: Live

| Port | URL | Purpose |
|------|-----|---------|
| 3000 | https://homepage.hrmsmrflrii.xyz | Dashboard |

Config: `/opt/homepage/config/` (settings.yaml, services.yaml, widgets.yaml, bookmarks.yaml).

---

## VLAN Flow (Network Observability)

**Host**: docker-vm-core-utilities01 (192.168.40.13) — native systemd service `vlan-flow.service` (Python standard library, `DynamicUser`, no Docker)
**Status**: Live — source `agents/vlan-flow/`, deployed with its `update.sh` (backs up `/opt/vlan-flow`, runs tests, restarts)
**Public generic version**: https://github.com/herms14/vlan-flow (since 2026-10-16; networks/senders/hosts in `config.json`, Docker or systemd install, no secrets). The live copy here still uses the homelab-specific source.

| Port | URL | Purpose |
|------|-----|---------|
| 8787/tcp | https://vlanflow.hrmsmrflrii.xyz (Traefik + Authentik) · http://192.168.40.13:8787 | Dashboard + `/api/state`, `/api/trace`, `/healthz` |
| 5514/udp | — | Omada syslog intake (controller 192.168.0.101 + gateway interfaces only) |

- Persists only explicit `[ACL]` allow/deny decisions (SQLite `/var/lib/vlan-flow/events.sqlite3`, 7 days / 100k rows). **No ACL rules exist yet**, so the decision map is empty until Gateway ACL rules are created with Logging — see [64 - VLAN ACL Rule Set](64%20-%20VLAN%20ACL%20Rule%20Set.md).
- The **Network activity** panel shows the latest 300 syslog messages of any type (client online/offline, DHCP, devices), in memory only.
- `vlan-flow-admin-route.service` adds `192.168.10.10/32 via 192.168.40.1` to work around the Docker subnet overlap on this host (Task #54).
- Accepted Host headers: the IP:port, localhost and `vlanflow.hrmsmrflrii.xyz` (`--hosts`); others get 403.

---

## Homelab Chronicle

**Host**: chronicle-lxc, CT207 (192.168.40.15)
**Status**: Live

| Port | URL | Purpose |
|------|-----|---------|
| 3000 | https://chronicle.hrmsmrflrii.xyz | Web interface |


Compose healthcheck uses Node's HTTP client over IPv4 for `/api/events` and requires HTTP 200; the previous `wget` probe referenced a tool absent from the production image. Compose pins the pre-repair running image by ID.

---

## Related Documentation

- [08 - Arr Media Stack](08%20-%20Arr%20Media%20Stack.md) - Media stack details
- [09 - Traefik Reverse Proxy](09%20-%20Traefik%20Reverse%20Proxy.md) - SSL and routing
- [17 - Monitoring Stack](17%20-%20Monitoring%20Stack.md) - Monitoring infrastructure
- [18 - Observability Stack](18%20-%20Observability%20Stack.md) - Distributed tracing
- [19 - Watchtower Updates](19%20-%20Watchtower%20Updates.md) - Interactive container updates
- [06 - Ansible Automation](06%20-%20Ansible%20Automation.md) - Deployment playbooks
- 11 - Credentials - API keys and passwords
- [15 - New Service Onboarding Guide](15%20-%20New%20Service%20Onboarding%20Guide.md) - Adding new services
