---
banner: "cybersecurity.jpg"
updated: 2026-10-18
---
# Homelab Infrastructure Index


## 🚨 Emergency Quick Reference

**Something's down? Start here.**

| If this is broken... | Check |
|---|---|
| Everything (all `*.hrmsmrflrii.xyz` sites down) | Traefik (`traefik-lxc`, 192.168.40.20) — `ssh traefik-lxc "docker ps"` |
| Login/SSO on any site | Authentik (`authentik-lxc`, 192.168.40.21) |
| Can't SSH anywhere | [16 - SSH Configuration](16%20-%20SSH%20Configuration.md) — key is `~/.ssh/homelab_ed25519` |
| DNS / can't resolve `*.hrmsmrflrii.xyz` internally | Pi-hole (`pihole`, 192.168.90.53) |
| Media (Jellyfin/Arr stack) | `docker-lxc-media`, 192.168.40.11 |
| Dashboards (Glance/Homepage) | `docker-lxc-glance` (.12) / `homepage-lxc` (.27) |
| Backups | PBS at 192.168.20.50:8007 — see [40 - PBS Disaster Recovery](40%20-%20PBS%20Disaster%20Recovery.md) |
| A specific service | [07 - Deployed Services](07%20-%20Deployed%20Services.md) has the full host/port map |

**Passwords/tokens**: 11 - Credentials
**Full node/LXC/VM map**: [10 - IP Address Map](10%20-%20IP%20Address%20Map.md)
**Known issues + fixes**: [12 - Troubleshooting](12%20-%20Troubleshooting.md)
**What changed recently**: [Homelab Changelog](Homelab%20Changelog.md)

---

## Quick Links

### Infrastructure
- [01 - Network Architecture](01%20-%20Network%20Architecture.md) - VLANs, physical topology, switch configs
- [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) - Nodes, VM/LXC standards, templates
- [03 - Storage Architecture](03%20-%20Storage%20Architecture.md) - NFS, Synology, storage pools

### Deployment
- [05 - Terraform Configuration](05%20-%20Terraform%20Configuration.md) - IaC modules and deployment
- [06 - Ansible Automation](06%20-%20Ansible%20Automation.md) - Playbooks, inventory, Discord notifications

### Services
- [07 - Deployed Services](07%20-%20Deployed%20Services.md) - All Docker services with ports (single source of truth)
- [08 - Arr Media Stack](08%20-%20Arr%20Media%20Stack.md) - Jellyfin, Radarr, Sonarr, etc.
- [09 - Traefik Reverse Proxy](09%20-%20Traefik%20Reverse%20Proxy.md) - SSL, routing, configuration

### Reference
- [10 - IP Address Map](10%20-%20IP%20Address%20Map.md) - Complete IP allocation
- 11 - Credentials - All passwords, API keys, tokens
- [12 - Troubleshooting](12%20-%20Troubleshooting.md) - Common issues and resolutions
- [13 - Service Configuration Guide](13%20-%20Service%20Configuration%20Guide.md) - Detailed service setup with command explanations
- [14 - Authentik Google SSO Setup](14%20-%20Authentik%20Google%20SSO%20Setup.md) - Google SSO configuration for all services
- [15 - New Service Onboarding Guide](15%20-%20New%20Service%20Onboarding%20Guide.md) - Complete workflow for adding new services
- [16 - SSH Configuration](16%20-%20SSH%20Configuration.md) - SSH keys, config file, host aliases
- [41 - Omada ACL Implementation Guide](41%20-%20Omada%20ACL%20Implementation%20Guide.md) - Network ACL rules (2026-01 draft)
- [64 - VLAN ACL Rule Set](64%20-%20VLAN%20ACL%20Rule%20Set.md) - Gateway ACLs per VLAN, managed by Torrent (**all disabled since 2026-10-16** after the phase 4 outage)
- [RFC Register](RFCs/RFC%20Register.md) - Requests for Change filed by agents, closed by Ranni

### Monitoring & Automation
- [17 - Monitoring Stack](17%20-%20Monitoring%20Stack.md) - Uptime Kuma, Prometheus, Grafana
- [18 - Observability Stack](18%20-%20Observability%20Stack.md) - OpenTelemetry, Jaeger, distributed tracing
- [19 - Watchtower Updates](19%20-%20Watchtower%20Updates.md) - Gated container update workflow via Sentinel bot
- [20 - GitLab CI-CD Automation](20%20-%20GitLab%20CI-CD%20Automation.md) - Automated service onboarding pipeline
- [21 - Application Configurations](21%20-%20Application%20Configurations.md) - Detailed app setup with command explanations
- [23 - Glance Dashboard](23%20-%20Glance%20Dashboard.md) - Dashboard, Media Stats widget, custom API
- Hermes Lab Atlas - Interactive architecture, network, services, agents and hardware diagrams (HTML, PDF export)
- [24 - Discord Bots](24%20-%20Discord%20Bots.md) - Sentinel unified homelab bot

### Apps
- [56 - Claude Code Statusline Setup](56%20-%20Claude%20Code%20Statusline%20Setup.md)
- [57 - Immich VectorChord Migration and v3 Upgrade](57%20-%20Immich%20VectorChord%20Migration%20and%20v3%20Upgrade.md)
- [58 - Helios API Reference](58%20-%20Helios%20API%20Reference.md)
- [65 - OpenBao Secrets Management](65%20-%20OpenBao%20Secrets%20Management.md) - OpenBao on CT213, per-agent scoped tokens through the Roundtable broker, Varys the keeper (never sees values)
- 59 - Autonomous Patch Management Agent Pipeline

### Tutorials
- [26 - Tutorials Index](26%20-%20Tutorials%20Index.md) - Complete list of all tutorials
- [29 - Immich Authentik SSO Tutorial](29%20-%20Immich%20Authentik%20SSO%20Tutorial.md) - OAuth/SSO for Immich
- [30 - LXC Migration Tutorial](30%20-%20LXC%20Migration%20Tutorial.md) - Migrating services from VM to LXC
- [39 - PBS Deployment Tutorial](39%20-%20PBS%20Deployment%20Tutorial.md) - Proxmox Backup Server LXC deployment
- [40 - PBS Disaster Recovery](40%20-%20PBS%20Disaster%20Recovery.md) - Complete PBS recovery when drives fail
- [42 - Immich Backup and Restore](42%20-%20Immich%20Backup%20and%20Restore.md) - Immich database backup and disaster recovery
- [43 - GitOps Complete Tutorial](43%20-%20GitOps%20Complete%20Tutorial.md) - GitOps theory, design patterns, and implementation (includes architecture, pipeline, deployment, troubleshooting)
- [44 - Glance GitOps Onboarding Tutorial](44%20-%20Glance%20GitOps%20Onboarding%20Tutorial.md) - Hands-on GitOps migration with Glance
- [47 - PBS Monitoring](47%20-%20PBS%20Monitoring.md) - PBS backup monitoring in Glance/Grafana

### Active TODOs
- [TODO - Homelab Improvements 2026-05](TODO%20-%20Homelab%20Improvements%202026-05.md) - Prioritized improvement backlog
- [TODO - Cloudflare Tunnel Setup](TODO%20-%20Cloudflare%20Tunnel%20Setup.md) - Expose Jellyfin/Jellyseerr to internet

### Meta / Agent System
- 62 - Roundtable Technical Manual - How the Roundtable agents are built, governed and deployed
- Agents - Hub for every agent (blog writer, image artist, patch pipeline, slash-command skills) and how to run each one
- 60 - Roundtable Agent Team - Current 29-agent/four-lead Roundtable runtime, artwork, Discord routes and recovery
- Avatar-guided terminal learning - Hewg/Torrent/Radahn, real bounded SSH in Mac/Windows browsers, learner controls and owner-approved narrow configuration
- Homelab Agent Documentation - How the Claude/slash-command agent system works
- Homelab Blog Writer Agent / Image Artist Agent - Codex blog agents on codex-agent-lxc (`blog-writer` from PowerShell, live preview in the browser; `blog-preview` to reread pulled drafts)
- [Homelab Changelog](Homelab%20Changelog.md) - Infrastructure change log (most current record in this vault)

### Narrative / Historical
- [Book - The Complete Homelab Guide](Book%20-%20The%20Complete%20Homelab%20Guide.md) - Step-by-step tutorial from trip photos to production infrastructure
- `<redacted>` - Docs superseded by the numbered reference set above (Technical Manual, Master Wiki, pre-K8s-decommission docs, pre-consolidation bot docs, etc.) — kept for history, not current reference
- `Archived/` and `_archived-2026-05-azure/` - Fully retired infrastructure (old NAS, Azure/Windows hybrid)

---

## Infrastructure Summary

| Metric | Value |
|--------|-------|
| Proxmox Nodes | 3 (MorpheusCluster + Qdevice) |
| Virtual Machines | 5 (ansible-controller01, docker-vm-core-utilities01, gitlab-runner-vm01, gitlab-vm01, immich-vm01) |
| LXC Containers | 11 (see [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md)) |
| Docker Containers | 40+ |
| Kubernetes | Decommissioned 2026-05-23 |
| Azure/Hybrid | Decommissioned 2026-05-25 |
| Observability | Full stack (OTEL, Jaeger, Prometheus, Grafana) |

## Service URLs

| Service | Internal URL | External URL |
|---------|--------------|--------------|
| Proxmox | https://192.168.20.20:8006 | https://proxmox.hrmsmrflrii.xyz |
| PBS | https://192.168.20.50:8007 | https://pbs.hrmsmrflrii.xyz |
| Traefik | http://192.168.40.20:8080 | https://traefik.hrmsmrflrii.xyz |
| Authentik | http://192.168.40.21:9000 | https://auth.hrmsmrflrii.xyz |
| Immich | http://192.168.40.22:2283 | https://photos.hrmsmrflrii.xyz |
| GitLab | http://192.168.40.23:80 | https://gitlab.hrmsmrflrii.xyz |
| Jellyfin | http://192.168.40.11:8096 | https://jellyfin.hrmsmrflrii.xyz |
| Glance | http://192.168.40.12:8080 | https://glance.hrmsmrflrii.xyz |
| Homepage | http://192.168.40.27:3000 | https://homepage.hrmsmrflrii.xyz |
| Home Assistant | http://192.168.40.25:8123 | https://ha.hrmsmrflrii.xyz |
| Uptime Kuma | http://192.168.40.13:3001 | https://uptime.hrmsmrflrii.xyz |
| Prometheus | http://192.168.40.13:9090 | https://prometheus.hrmsmrflrii.xyz |
| Grafana | http://192.168.40.13:3030 | https://grafana.hrmsmrflrii.xyz |
| n8n | http://192.168.40.13:5678 | https://n8n.hrmsmrflrii.xyz |
| Paperless-ngx | http://192.168.40.13:8000 | https://paperless.hrmsmrflrii.xyz |
| Karakeep | http://192.168.40.13:3005 | https://karakeep.hrmsmrflrii.xyz |
| Roundtable | http://192.168.40.16:8088 (view-only without sign-in) | https://roundtable.hrmsmrflrii.xyz |
| VLAN Flow | http://192.168.40.13:8787 | https://vlanflow.hrmsmrflrii.xyz |

See [07 - Deployed Services](07%20-%20Deployed%20Services.md) for the complete list (40+ containers).

---

## Related External Docs

- [GitHub Repository](https://github.com/herms14/Proxmox-TerraformDeployments) - Public sanitized docs
- [GitHub Wiki](https://github.com/herms14/Proxmox-TerraformDeployments/wiki) - Beginner-friendly guides
- [GitLab Wiki](https://gitlab.hrmsmrflrii.xyz/hrmsmrflrii/homelab-project/-/wikis/home) - Synced technical wiki

---

*Last updated: 2026-10-11*
