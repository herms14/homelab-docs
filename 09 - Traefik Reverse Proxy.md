# Traefik Reverse Proxy

> **Internal Documentation** - Contains SSL configuration and Cloudflare API details.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [07 - Deployed Services](07%20-%20Deployed%20Services.md) | 11 - Credentials | [18 - Observability Stack](18%20-%20Observability%20Stack.md)

---

## Overview

**Host**: traefik-lxc, CT203 (192.168.40.20)
**Version**: Traefik v3.7.12 (upgraded 2026-08-29, was v3.2)
**Status**: Migrated to LXC January 7, 2026

| Port | Purpose |
|------|---------|
| 80 | HTTP (redirects to HTTPS) |
| 443 | HTTPS traffic |
| 8080 | Dashboard |
| 8082 | Prometheus metrics |

---

## SSL Configuration

### Let's Encrypt via Cloudflare DNS-01

| Setting | Value |
|---------|-------|
| Domain | `hrmsmrflrii.xyz` |
| Certificate Type | Wildcard (*.hrmsmrflrii.xyz) |
| Challenge | DNS-01 via Cloudflare API |
| Email | <email> |
| Storage | `/opt/traefik/certs/acme.json` |

### Cloudflare Configuration

- **Account**: <email>
- **API Token**: *(stored in Traefik config)*
- **Zone**: hrmsmrflrii.xyz

---

## Storage Layout

```
/opt/traefik/
├── docker-compose.yml
├── config/
│   ├── traefik.yml          # Static config
│   └── dynamic/
│       └── services.yml      # Dynamic service routes
├── certs/
│   └── acme.json            # Let's Encrypt certificates
└── logs/
    └── traefik.log
```

---

## OpenTelemetry Tracing

**Status**: Enabled December 21, 2025

Traefik sends distributed traces to the OTEL Collector for visualization in Jaeger.

### Configuration

```yaml
# In traefik.yml static config
tracing:
  otlp:
    http:
      endpoint: "http://192.168.40.13:4318/v1/traces"
  serviceName: "traefik"
  sampleRate: 1.0  # 100% sampling

metrics:
  prometheus:
    buckets: [0.1, 0.3, 1.2, 5.0]
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
    entryPoint: metrics

entryPoints:
  metrics:
    address: ":8082"
```

### Trace Flow

```
User Request → Traefik → OTEL Collector → Jaeger
                 ↓
            Prometheus (metrics)
```

### Viewing Traces

1. Navigate to https://jaeger.hrmsmrflrii.xyz
2. Select "traefik" from service dropdown
3. Click "Find Traces"
4. Click on a trace to see span details

### Metrics Endpoint

Prometheus scrapes Traefik metrics at: `http://192.168.40.20:8082/metrics`

> See [18 - Observability Stack](18%20-%20Observability%20Stack.md) for complete observability configuration.

---

## Pre-configured Routes

| Hostname | Backend Service |
|----------|----------------|
| auth.hrmsmrflrii.xyz | Authentik (192.168.40.21:9000) |
| photos.hrmsmrflrii.xyz | Immich (192.168.40.22:2283) |
| gitlab.hrmsmrflrii.xyz | GitLab (192.168.40.23:80) |
| jellyfin.hrmsmrflrii.xyz | Jellyfin (192.168.40.11:8096) |
| radarr.hrmsmrflrii.xyz | Radarr (192.168.40.11:7878) |
| sonarr.hrmsmrflrii.xyz | Sonarr (192.168.40.11:8989) |
| lidarr.hrmsmrflrii.xyz | Lidarr (192.168.40.11:8686) |
| prowlarr.hrmsmrflrii.xyz | Prowlarr (192.168.40.11:9696) |
| bazarr.hrmsmrflrii.xyz | Bazarr (192.168.40.11:6767) |
| n8n.hrmsmrflrii.xyz | n8n (192.168.40.13:5678) |
| proxmox.hrmsmrflrii.xyz | Proxmox (192.168.20.21:8006) |
| uptime.hrmsmrflrii.xyz | Uptime Kuma (192.168.40.13:3001) |
| prometheus.hrmsmrflrii.xyz | Prometheus (192.168.40.13:9090) |
| grafana.hrmsmrflrii.xyz | Grafana (192.168.40.13:3030) |
| jaeger.hrmsmrflrii.xyz | Jaeger (192.168.40.13:16686) |
| demo.hrmsmrflrii.xyz | Demo App (192.168.40.12:8080) |

---

## Adding New Services

Edit `/opt/traefik/config/dynamic/services.yml`:

```yaml
http:
  routers:
    new-service:
      rule: "Host(`newservice.hrmsmrflrii.xyz`)"
      service: new-service
      entryPoints:
        - websecure
      tls:
        certResolver: letsencrypt

  services:
    new-service:
      loadBalancer:
        servers:
          - url: "http://192.168.40.XX:PORT"
```

Changes auto-reload (no restart needed).

---

## Management Commands

```bash
# SSH to Traefik host
ssh hermes-admin@192.168.40.20

# View logs
cd /opt/traefik && sudo docker compose logs -f

# Restart Traefik
cd /opt/traefik && sudo docker compose restart

# Update Traefik
cd /opt/traefik && sudo docker compose pull && sudo docker compose up -d
```

---

## Ansible Deployment

Playbook: `~/ansible/traefik/deploy-traefik.yml`

```bash
# From ansible-controller01
ansible-playbook traefik/deploy-traefik.yml
```

---

## Internet access (Cloudflare Tunnel + Access) - 2026-10-19

Some routes are also reachable from the internet, without port forwarding and without changing Traefik:

- **Tunnel** `homelab-public` (remote-managed in Cloudflare). Its connector, `cloudflared-homelab-public.service`, runs on this LXC (CT203) and sends each request to `https://192.168.40.20:443` with the original hostname as SNI and Host. Routers, certificates and Authentik forward auth apply exactly as on the LAN. Unknown hostnames get 404.
- **Public hostnames:** `roundtable` (incl. `/atlas`), `glance`, `vlanflow`, `photos`, `paperless`, `auth`. Study has its own tunnel.
- **Login wall:** every hostname has a Cloudflare Access app (Google only, 24 h). Only <email> is allowed.
- **Split horizon:** Technitium still answers LAN and Tailscale clients with 192.168.40.20, so those never pass through Access.
- **Adding a hostname:** add it to `APPS` in `agents/cloudflare/cloudflare_public.py`, then run `access`, `tunnel`, `dns` and `check`, in that order. Also add it to `HOSTS` in `agents/cloudflare/public-apps-check.sh` (installed at `/usr/local/bin/`) and to the Watch `homelab` site's `host` list.
- **Kill switch:** `sudo systemctl stop cloudflared-homelab-public` on CT203. LAN access is unaffected.
- **Monitoring:** `public-apps-check.timer` (every 2 min) → Uptime Kuma "Tunnel + Access login wall" → Prometheus `PublicAppsPathDown`. See [17 - Monitoring Stack](17%20-%20Monitoring%20Stack.md).
- **Known issue:** `/opt/traefik/logs/access.log` isn't rotated (7.9 GB on 2026-10-19, Task #102).

---

## Related Documentation

- [18 - Observability Stack](18%20-%20Observability%20Stack.md) - Distributed tracing with Jaeger
- [17 - Monitoring Stack](17%20-%20Monitoring%20Stack.md) - Prometheus and Grafana
- [07 - Deployed Services](07%20-%20Deployed%20Services.md) - All service URLs
- [01 - Network Architecture](01%20-%20Network%20Architecture.md) - DNS configuration
- [06 - Ansible Automation](06%20-%20Ansible%20Automation.md) - Deployment playbooks
- 11 - Credentials - Cloudflare API token

