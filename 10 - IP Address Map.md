---
updated: 2026-10-19
---
# IP Address Map

> **Internal Documentation** - Complete IP allocation for all VLANs.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [01 - Network Architecture](01%20-%20Network%20Architecture.md) | [07 - Deployed Services](07%20-%20Deployed%20Services.md)

---

## Current new-service policy - 2026-10-11

Diallos now considers **`.36-.49` on VLANs 20, 40 and 90** for new services, instead of
the unsafe `.150-.199` pool. These are **candidates, not verified free addresses**.
Live Omada DHCP remains **`.50-.254`** on all three VLANs; gateways, leases, VLANs and
existing service addresses were not changed.

Allocation checks the complete controller network/current-client/history inventory, all DHCP
reservations and IP/MAC bindings (including disabled entries), Proxmox interface MAC/IP configuration,
host inventory and atomic service reservations. Offline CT209's interface proves its `.40.27`
address even when Omada's wired history has no IP. Older recorded addresses are also excluded.
Missing last-seen IP evidence is not replaced with an unanswered ping.

**Repair completed:** the initially unresolved `BA-20-51-C1-EA-A7` row is an internal Omada
mesh-backhaul association, not a separate IP-addressed host. Its complete history uses the First Floor
EAP's internal mesh SSID, absent from configured user WLANs; live controller evidence confirms an online
mesh-enabled root and unique reciprocal Outdoor EAP wireless link. Both AP IPs remain excluded.
This is recorded as Layer-2 transport evidence, not a guessed IP, VLAN-0-to-VLAN-1 assumption or
MAC-prefix exemption. Ordinary/ambiguous clients, changed mesh topology and incomplete evidence still block.

Live authoritative inventory returns **14 candidates per VLAN (42 total)**. The real
reserve/verify/activate flow passed with fresh network evidence and ICMP in an isolated validation ledger.
Production reservations remain zero; no service was created. `IPAM-DESIGN-20261011` and GitLab
work item **#83** are completed. Allocations still require fresh evidence, an atomic reservation and approval.

The historical expansion tables below are **not current allocation authority**. Any part at `.50`
or above overlaps DHCP and must not be used for a new static service without a separately reviewed
controller reservation or DHCP change. Existing PBS `.20.50`, test VM `.40.150` and Pi-hole `.90.53`
were preserved; this safeguard rollout does not resolve their static-inside-DHCP risks or the
separate PBS squatter task #65.


---

## VLAN 20 - Homelab Infrastructure (192.168.20.0/24)

### Proxmox Cluster

| IP | Hostname | Purpose |
|----|----------|---------|
| 192.168.20.1 | Gateway | VLAN 20 Gateway |
| 192.168.20.20 | node01 | Proxmox Node (VM Host) |
| 192.168.20.21 | node02 | Proxmox Node (LXC Host) |
| 192.168.20.22 | node03 | Proxmox Node (K8s Host) |

### Automation & Storage

| IP | Hostname | Purpose |
|----|----------|---------|
| 192.168.20.30 | ansible-controller01 | Ansible automation |
| 192.168.20.31 | synology-nas (eth0, 1GbE) | Synology NAS — management / legacy interface |
| 192.168.20.32 | synology-nas (eth1, 10GbE NIC) | **Synology NAS — primary NFS interface (2.5G switch)** |

> K8s cluster decommissioned — all K8s VMs (controllers .32-.34, workers .40-.45) deleted.

### Historical range plan - superseded by the current policy above

| Range | Purpose |
|-------|---------|
| 192.168.20.33-39 | Future infrastructure hosts |
| 192.168.20.40-99 | Future VMs / expansion |
| 192.168.20.100-199 | LXC containers |
| 192.168.20.200-254 | Future VMs |

---

## VLAN 40 - Production Services (192.168.40.0/24)

### Docker Hosts & LXC Containers

| IP | Hostname | Purpose |
|----|----------|---------|
| 192.168.40.1 | Gateway | VLAN 40 Gateway |
| 192.168.40.5 | linux-syslog-server01 | Centralized logging |
| 192.168.40.10 | ~~docker-vm-core-utilities01~~ | **DECOMMISSIONED** - Replaced by 192.168.40.12 & 192.168.40.13 |
| 192.168.40.11 | docker-vm-media01 | Arr Media Stack |
| 192.168.40.12 | lxc-glance (LXC 200) | Glance Dashboard |
| 192.168.40.13 | docker-vm-core-utilities-1 | n8n, Paperless, Speedtest, Monitoring Stack, Karakeep, Wizarr, Tracearr |
| 192.168.40.14 | helios-lxc (LXC 210) | Helios homelab CLI/API control plane — moved here from ansible-controller01 2026-10-03 |
| 192.168.40.16 | codex-agent-lxc (LXC 211) | Agent runner: patch pipeline (orchestrator + `sre-agent.service`) and blog-studio (`blog-writer`, `blog-artist`, `blog-preview` on 127.0.0.1:8765, reached via SSH tunnel only) |
| 192.168.40.17 | docker-lxc-bots (LXC 201) | Idle bot LXC — **re-IPed from .14 on 2026-10-05** (was conflicting with helios-lxc) |

### Core Application Services

| IP | Hostname | Purpose |
|----|----------|---------|
| 192.168.40.20 | traefik-lxc (LXC 203) | Reverse Proxy |
| 192.168.40.21 | authentik-lxc (LXC 204) | Identity/SSO |
| 192.168.40.22 | immich-vm01 | Photo Management |
| 192.168.40.23 | gitlab-vm01 | DevOps Platform |
| 192.168.40.24 | gitlab-runner-vm01 | GitLab CI/CD Runner |
| 192.168.40.25 | homeassistant-lxc (LXC 206) | Home Assistant Smart Home |
| 192.168.40.27 | homepage-lxc (LXC 209) | Homepage Dashboard — on node01 |
| 192.168.40.38 | study-lxc (LXC 212) | Study notebook (`study.service` :8090, https://study.hrmsmrflrii.xyz) and vault sync — IPAM reservation `study-lxc` active since 2026-10-17 |
| 192.168.40.39 | openbao-lxc (LXC 213) | OpenBao secrets store (:<redacted> TLS, https://bao.hrmsmrflrii.xyz), node exporter :9100 — IPAM reservation `openbao-vault` active since 2026-10-18; see [65 - OpenBao Secrets Management](65%20-%20OpenBao%20Secrets%20Management.md) |

### Historical range plan - superseded by the current policy above

| Range | Purpose |
|-------|---------|
| 192.168.40.16 | codex-agent-lxc (LXC 211) | Dedicated Codex CLI agent-runner for the patch-management pipeline — added 2026-10-04, see 59 - Autonomous Patch Management Agent Pipeline. Since 2026-10-07 also hosts blog-studio (`~/blog-studio`), see Agents |
| 192.168.40.15, 17-19 | Additional Docker hosts |
| 192.168.40.28-39 | Monitoring & additional services |
| 192.168.40.40-254 | Future services |

---

## VLAN 90 - Management (192.168.90.0/24)

| IP | Device | Purpose |
|----|--------|---------|
| 192.168.90.1 | Gateway | VLAN 90 Gateway |
| 192.168.90.2 | SG3210 | Core Switch |
| 192.168.90.3 | SG2210P | Morpheus Switch |
| 192.168.90.10 | EAP610 | Living Room WiFi AP |
| 192.168.90.11 | EAP603 | Outdoor WiFi AP |
| 192.168.90.12 | EAP225 | Computer Room WiFi AP |
| 192.168.90.51 | ES20GP | Atreus Switch |
| 192.168.90.53 | pihole (CT202) | Secondary DNS (Pi-hole + Unbound) |
| 192.168.90.60 | homelab-dns01 | **Primary DNS** (Technitium), bare-metal mini PC, static (was OPNsense until 2026-10-18) |

---

## VLAN 91 - Firewall (192.168.91.0/24)

| IP | Device | Purpose |
|----|--------|---------|
| 192.168.91.1 | Gateway | VLAN 91 Gateway |
| ~~192.168.91.30~~ | OPNsense | **Stale plan**: OPNsense was at **192.168.90.60** (VLAN 90, bare metal, Morpheus Rack Switch port 6); that box is homelab-dns01 since 2026-10-18 |

---

## Other VLANs

### VLAN 10 - Internal LAN

- 192.168.10.0/24
- Workstations, NAS access

### VLAN 30 - IoT

- 192.168.30.0/24
- IoT WiFi devices

### VLAN 50 - Guest

- 192.168.50.0/24
- Guest WiFi access

---

## Used and free addresses per VLAN (live)

The Roundtable portal → 🧭 IP Map shows a card per VLAN with used, free and total usable addresses. The static range and the DHCP pool are counted separately. Click a card to filter the table.

- An address counts as **used** when a live source has it (Proxmox, an Omada device or VLAN gateway, a DHCP reservation, a Roundtable host, a Traefik backend or an IPAM reservation), or when an Omada client holding it is connected now.
- Clients the controller only remembers, and retired or not-seen rows, don't count.
- The snapshot refreshes every 15 min (`ipmap-watch.timer`); an open page re-reads it every 60 s.
- Snapshot on 2026-10-19 (used of 254): VLAN 1 2, VLAN 10 7, VLAN 20 18, VLAN 30 8, VLAN 40 19, VLAN 50 1, VLAN 80 1, VLAN 90 11.

## Related Documentation

- [01 - Network Architecture](01%20-%20Network%20Architecture.md) - VLAN configuration
- [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) - Node details
- [07 - Deployed Services](07%20-%20Deployed%20Services.md) - Service details

## Roundtable IP registry (maintained by Diallos)

*Auto-maintained by Diallos from Proxmox, Omada, IPAM, Traefik and agent records. Updated 2026-10-19 07:27. Edit the tables above, not this section; it is rewritten whenever an address or its service changes (checked every 15 minutes) and daily. Two copies: sorted by IP address and by name. Live view with sorting and search: Roundtable portal → 🧭 IP Map.*

> [!list]+ Sorted by IP address
> | IP | Service | Purpose | URL | Status |
> |----|---------|---------|-----|--------|
> | 192.168.0.1 | Core Router | Core router and default gateway. *(AI suggestion)* |  | undocumented |
> | 192.168.0.101 |  |  | https://omada.hrmsmrflrii.xyz | undocumented |
> | 192.168.10.1 | Gateway (Internal VLAN) | Gateway for the internal VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.10.10 | KratosPC | PC named KratosPC on the internal VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.10.31 | CallimachusNAS - Internal 1 | NAS named CallimachusNAS - Internal 1 on the internal VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.20.1 | Gateway | VLAN 20 Gateway |  | ok |
> | 192.168.20.20 | node01 | Proxmox Node (VM Host) | https://node01.hrmsmrflrii.xyz | ok |
> | 192.168.20.21 | node02 | Proxmox Node (LXC Host) | https://node02.hrmsmrflrii.xyz, https://proxmox.hrmsmrflrii.xyz | ok |
> | 192.168.20.22 | node03 | Proxmox Node (K8s Host) | https://node03.hrmsmrflrii.xyz | ok |
> | 192.168.20.30 | ansible-controller01 | Ansible automation |  | ok |
> | 192.168.20.31 | synology-nas (eth0, 1GbE) | Synology NAS — management / legacy interface |  | ok |
> | 192.168.20.32 | synology-nas (eth1, 10GbE NIC) | **Synology NAS — primary NFS interface (2.5G switch)** | https://nas.hrmsmrflrii.xyz | ok |
> | 192.168.20.35 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.20.36 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.20.40 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.41 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.42 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.43 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.44 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.45 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.50 | pbs-server | PBS server in LXC 100 on node03, serving pbs.hrmsmrflrii.xyz. *(AI suggestion)* | https://pbs.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.51 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.30.1 | Gateway (IoT VLAN) | Gateway for the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.40.1 | Gateway | VLAN 40 Gateway |  | ok |
> | 192.168.40.5 | linux-syslog-server01 | Centralized logging |  | not-seen |
> | 192.168.40.10 | docker-vm-core-utilities01 | **DECOMMISSIONED** - Replaced by 192.168.40.12 & 192.168.40.13 | https://lagident.hrmsmrflrii.xyz | retired-but-seen |
> | 192.168.40.11 | docker-lxc-media | Arr Media Stack | https://autobrr.hrmsmrflrii.xyz, https://bazarr.hrmsmrflrii.xyz | ok |
> | 192.168.40.12 | docker-lxc-glance | Glance Dashboard | https://demo.hrmsmrflrii.xyz, https://glance.hrmsmrflrii.xyz | ok |
> | 192.168.40.13 | docker-vm-core-utilities01 | n8n, Paperless, Speedtest, Monitoring Stack, Karakeep, Wizarr, Tracearr | https://bentopdf.hrmsmrflrii.xyz, https://chronicle.hrmsmrflrii.xyz | ok |
> | 192.168.40.14 | helios-lxc | Helios homelab CLI/API control plane — moved here from ansible-controller01 2026-10-03 |  | ok |
> | 192.168.40.15 | chronicle-lxc | Proxmox LXC 207 on node02. *(AI suggestion)* |  | undocumented |
> | 192.168.40.16 | codex-agent-lxc | Agent runner: patch pipeline (orchestrator + `sre-agent.service`) and blog-studio (`blog-writer`, `blog-artist`, `blog-preview` on 127.0.0.1:8765, reached via S | https://olympus.hrmsmrflrii.xyz, https://roundtable.hrmsmrflrii.xyz | ok |
> | 192.168.40.17 | docker-lxc-bots | Idle bot LXC — **re-IPed from .14 on 2026-10-05** (was conflicting with helios-lxc) |  | ok |
> | 192.168.40.20 | traefik-lxc | Reverse Proxy |  | ok |
> | 192.168.40.21 | authentik-lxc | Identity/SSO | https://auth.hrmsmrflrii.xyz | ok |
> | 192.168.40.22 | immich-vm01 | Photo Management | https://photos.hrmsmrflrii.xyz | ok |
> | 192.168.40.23 | gitlab-vm01 | DevOps Platform | https://gitlab.hrmsmrflrii.xyz | ok |
> | 192.168.40.24 | gitlab-runner-vm01 | GitLab CI/CD Runner |  | ok |
> | 192.168.40.25 | homeassistant-lxc | Home Assistant Smart Home | https://ha.hrmsmrflrii.xyz | ok |
> | 192.168.40.27 | homepage-lxc | Homepage Dashboard — on node01 | https://homepage.hrmsmrflrii.xyz | ok |
> | 192.168.40.36 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.40.37 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.40.39 | openbao-lxc (CT213) | OpenBao secrets store for the Roundtable (Varys); API/UI :8200 TLS, node exporter :9100 | https://bao.hrmsmrflrii.xyz, https://192.168.40.39:8200/ui | ok |
> | 192.168.50.1 | Gateway (Guest VLAN) | Gateway for the guest VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.60.52 | SonosZP | Sonos device on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.60.54 | SonosZP | Sonos device on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.60.55 | SonosZP | Sonos device on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.60.57 | Hermes-s-S26-Ultra | S26 Ultra device named Hermes-s-S26-Ultra on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.60.58 | SonosZP | Sonos device on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.80.1 | Gateway (HyperV VLAN) | Gateway for the HyperV VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.90.1 | Gateway | VLAN 90 Gateway |  | ok |
> | 192.168.90.2 | SG3210 | Core Switch |  | not-seen |
> | 192.168.90.3 | SG2210P | Morpheus Switch |  | not-seen |
> | 192.168.90.10 | EAP610 | Living Room WiFi AP |  | not-seen |
> | 192.168.90.11 | EAP603 | Outdoor WiFi AP |  | not-seen |
> | 192.168.90.12 | EAP225 | Computer Room WiFi AP |  | not-seen |
> | 192.168.90.20 | OC300 Controller | OC300 controller. *(AI suggestion)* |  | undocumented |
> | 192.168.90.50 | Atreus Switch | Network switch named Atreus Switch. *(AI suggestion)* |  | undocumented |
> | 192.168.90.51 | ES20GP | Atreus Switch |  | ok |
> | 192.168.90.52 | First Floor EAP | First-floor EAP. *(AI suggestion)* |  | undocumented |
> | 192.168.90.53 | pihole | Secondary DNS (Pi-hole + Unbound) | https://pihole.hrmsmrflrii.xyz | ok |
> | 192.168.90.54 | Core Switch | Core network switch. *(AI suggestion)* |  | undocumented |
> | 192.168.90.55 | Outdoor EAP | Outdoor EAP. *(AI suggestion)* |  | undocumented |
> | 192.168.90.56 | Second Floor EAP | Second-floor EAP. *(AI suggestion)* |  | undocumented |
> | 192.168.90.58 | Zephyrus Switch | Network switch named Zephyrus Switch. *(AI suggestion)* |  | undocumented |
> | 192.168.90.60 | homelab-dns01 | **Primary DNS** (Technitium), bare-metal mini PC, static (was OPNsense until 2026-10-18) | https://dns.hrmsmrflrii.xyz | ok |
> | 192.168.91.1 | Gateway | VLAN 91 Gateway |  | not-seen |
> | 192.168.91.30 | OPNsense | **Stale plan**: OPNsense was at **192.168.90.60** (VLAN 90, bare metal, Morpheus Rack Switch port 6); that box is homelab-dns01 since 2026-10-18 |  | not-seen |

> [!list]- Sorted by name
> | IP | Service | Purpose | URL | Status |
> |----|---------|---------|-----|--------|
> | 192.168.20.30 | ansible-controller01 | Ansible automation |  | ok |
> | 192.168.90.50 | Atreus Switch | Network switch named Atreus Switch. *(AI suggestion)* |  | undocumented |
> | 192.168.40.21 | authentik-lxc | Identity/SSO | https://auth.hrmsmrflrii.xyz | ok |
> | 192.168.10.31 | CallimachusNAS - Internal 1 | NAS named CallimachusNAS - Internal 1 on the internal VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.40.15 | chronicle-lxc | Proxmox LXC 207 on node02. *(AI suggestion)* |  | undocumented |
> | 192.168.40.16 | codex-agent-lxc | Agent runner: patch pipeline (orchestrator + `sre-agent.service`) and blog-studio (`blog-writer`, `blog-artist`, `blog-preview` on 127.0.0.1:8765, reached via S | https://olympus.hrmsmrflrii.xyz, https://roundtable.hrmsmrflrii.xyz | ok |
> | 192.168.0.1 | Core Router | Core router and default gateway. *(AI suggestion)* |  | undocumented |
> | 192.168.90.54 | Core Switch | Core network switch. *(AI suggestion)* |  | undocumented |
> | 192.168.40.17 | docker-lxc-bots | Idle bot LXC — **re-IPed from .14 on 2026-10-05** (was conflicting with helios-lxc) |  | ok |
> | 192.168.40.12 | docker-lxc-glance | Glance Dashboard | https://demo.hrmsmrflrii.xyz, https://glance.hrmsmrflrii.xyz | ok |
> | 192.168.40.11 | docker-lxc-media | Arr Media Stack | https://autobrr.hrmsmrflrii.xyz, https://bazarr.hrmsmrflrii.xyz | ok |
> | 192.168.40.10 | docker-vm-core-utilities01 | **DECOMMISSIONED** - Replaced by 192.168.40.12 & 192.168.40.13 | https://lagident.hrmsmrflrii.xyz | retired-but-seen |
> | 192.168.40.13 | docker-vm-core-utilities01 | n8n, Paperless, Speedtest, Monitoring Stack, Karakeep, Wizarr, Tracearr | https://bentopdf.hrmsmrflrii.xyz, https://chronicle.hrmsmrflrii.xyz | ok |
> | 192.168.90.12 | EAP225 | Computer Room WiFi AP |  | not-seen |
> | 192.168.90.11 | EAP603 | Outdoor WiFi AP |  | not-seen |
> | 192.168.90.10 | EAP610 | Living Room WiFi AP |  | not-seen |
> | 192.168.90.51 | ES20GP | Atreus Switch |  | ok |
> | 192.168.90.52 | First Floor EAP | First-floor EAP. *(AI suggestion)* |  | undocumented |
> | 192.168.20.1 | Gateway | VLAN 20 Gateway |  | ok |
> | 192.168.40.1 | Gateway | VLAN 40 Gateway |  | ok |
> | 192.168.90.1 | Gateway | VLAN 90 Gateway |  | ok |
> | 192.168.91.1 | Gateway | VLAN 91 Gateway |  | not-seen |
> | 192.168.50.1 | Gateway (Guest VLAN) | Gateway for the guest VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.80.1 | Gateway (HyperV VLAN) | Gateway for the HyperV VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.10.1 | Gateway (Internal VLAN) | Gateway for the internal VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.30.1 | Gateway (IoT VLAN) | Gateway for the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.40.24 | gitlab-runner-vm01 | GitLab CI/CD Runner |  | ok |
> | 192.168.40.23 | gitlab-vm01 | DevOps Platform | https://gitlab.hrmsmrflrii.xyz | ok |
> | 192.168.40.14 | helios-lxc | Helios homelab CLI/API control plane — moved here from ansible-controller01 2026-10-03 |  | ok |
> | 192.168.60.57 | Hermes-s-S26-Ultra | S26 Ultra device named Hermes-s-S26-Ultra on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.40.25 | homeassistant-lxc | Home Assistant Smart Home | https://ha.hrmsmrflrii.xyz | ok |
> | 192.168.90.60 | homelab-dns01 | **Primary DNS** (Technitium), bare-metal mini PC, static (was OPNsense until 2026-10-18) | https://dns.hrmsmrflrii.xyz | ok |
> | 192.168.40.27 | homepage-lxc | Homepage Dashboard — on node01 | https://homepage.hrmsmrflrii.xyz | ok |
> | 192.168.40.22 | immich-vm01 | Photo Management | https://photos.hrmsmrflrii.xyz | ok |
> | 192.168.20.40 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.41 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.42 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.43 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.44 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.20.45 | k8s-test.hrmsmrflrii.xyz host | Host serving k8s-test.hrmsmrflrii.xyz. *(AI suggestion)* | https://k8s-test.hrmsmrflrii.xyz | undocumented |
> | 192.168.10.10 | KratosPC | PC named KratosPC on the internal VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.40.5 | linux-syslog-server01 | Centralized logging |  | not-seen |
> | 192.168.20.20 | node01 | Proxmox Node (VM Host) | https://node01.hrmsmrflrii.xyz | ok |
> | 192.168.20.21 | node02 | Proxmox Node (LXC Host) | https://node02.hrmsmrflrii.xyz, https://proxmox.hrmsmrflrii.xyz | ok |
> | 192.168.20.22 | node03 | Proxmox Node (K8s Host) | https://node03.hrmsmrflrii.xyz | ok |
> | 192.168.90.20 | OC300 Controller | OC300 controller. *(AI suggestion)* |  | undocumented |
> | 192.168.40.39 | openbao-lxc (CT213) | OpenBao secrets store for the Roundtable (Varys); API/UI :8200 TLS, node exporter :9100 | https://bao.hrmsmrflrii.xyz, https://192.168.40.39:8200/ui | ok |
> | 192.168.91.30 | OPNsense | **Stale plan**: OPNsense was at **192.168.90.60** (VLAN 90, bare metal, Morpheus Rack Switch port 6); that box is homelab-dns01 since 2026-10-18 |  | not-seen |
> | 192.168.90.55 | Outdoor EAP | Outdoor EAP. *(AI suggestion)* |  | undocumented |
> | 192.168.20.50 | pbs-server | PBS server in LXC 100 on node03, serving pbs.hrmsmrflrii.xyz. *(AI suggestion)* | https://pbs.hrmsmrflrii.xyz | undocumented |
> | 192.168.90.53 | pihole | Secondary DNS (Pi-hole + Unbound) | https://pihole.hrmsmrflrii.xyz | ok |
> | 192.168.90.56 | Second Floor EAP | Second-floor EAP. *(AI suggestion)* |  | undocumented |
> | 192.168.90.3 | SG2210P | Morpheus Switch |  | not-seen |
> | 192.168.90.2 | SG3210 | Core Switch |  | not-seen |
> | 192.168.60.52 | SonosZP | Sonos device on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.60.54 | SonosZP | Sonos device on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.60.55 | SonosZP | Sonos device on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.60.58 | SonosZP | Sonos device on the IoT VLAN. *(AI suggestion)* |  | undocumented |
> | 192.168.20.31 | synology-nas (eth0, 1GbE) | Synology NAS — management / legacy interface |  | ok |
> | 192.168.20.32 | synology-nas (eth1, 10GbE NIC) | **Synology NAS — primary NFS interface (2.5G switch)** | https://nas.hrmsmrflrii.xyz | ok |
> | 192.168.40.20 | traefik-lxc | Reverse Proxy |  | ok |
> | 192.168.20.35 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.20.36 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.20.51 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.40.36 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.40.37 | unknown device | Unidentified static client; check the switch port or Omada *(AI suggestion)* |  | undocumented |
> | 192.168.90.58 | Zephyrus Switch | Network switch named Zephyrus Switch. *(AI suggestion)* |  | undocumented |
> | 192.168.0.101 |  |  | https://omada.hrmsmrflrii.xyz | undocumented |

