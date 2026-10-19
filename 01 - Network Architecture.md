---
banner: "01 - Network Architecture.gif"
updated: 2026-10-18
---
# Network Architecture


Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | 11 - Credentials | [10 - IP Address Map](10%20-%20IP%20Address%20Map.md)

---

## Physical Network Topology


## Network Hardware



| DEVICE NAME          | SITE NAME       | IP ADDRESS    | STATUS              | HEALTH  | MODEL                   | VERSION                          | UPTIME              |
| -------------------- | --------------- | ------------- | ------------------- | ------- | ----------------------- | -------------------------------- | ------------------- |
| Core Router          | Parang Marikina | 192.168.0.1   | Connected           | Good-10 | ER605 v2.20             | 2.3.3 Build 20251029 Rel.18054   | 17h 52m 12s         |
| Atreus Switch        | Parang Marikina | 192.168.90.50 | Connected           | Good-10 | ES206GP v1.0            | 1.0.5 Build 20260325 Rel.31583   | 8day(s) 12h 9m 50s  |
| Core Switch          | Parang Marikina | 192.168.90.55 | Connected           | Good-10 | SG3210 v3.20            | 3.20.19 Build 20260310 Rel.10715 | 6day(s) 3h 38m 23s  |
| Morpheus Rack Switch | Parang Marikina | 192.168.90.57 | Connected           | Good-10 | SG2210P v5.20           | 5.20.20 Build 20260310 Rel.10715 | 4day(s) 2h 46m 35s  |
| Zephyrus Switch      | Parang Marikina | 192.168.90.54 | Connected           | Good-10 | SG3210X-M2 v1.0         | 1.0.7 Build 20250123 Rel.42187   | 1h 21m 51s          |
| First Floor EAP      | Parang Marikina | 192.168.90.51 | Connected           | Good-10 | EAP610(US) v3.0         | 1.6.0 Build 20250507 Rel. 66340  | 8day(s) 12h 10m 15s |
| Second Floor EAP     | Parang Marikina | 192.168.90.58 | Connected           | Good-10 | EAP225(US) v4.0         | 5.1.11 Build 20250401 Rel. 50587 | 4day(s) 2h 46m 12s  |
| Outdoor EAP          | Parang Marikina | 192.168.90.52 | Connected(Wireless) | Good-9  | EAP603-Outdoor(US) v1.0 | 1.0.1 Build 20250320 Rel. 41321  | 8day(s) 19h 27m 34s |


---

## Network Controller Credentials

### Omada Cloud Controller
- **Username**: `hermes-admin`
- **Password**: `<redacted>`

### Device Passwords (Routers/EAPs)
- **Username**: `hermes-admin`
- **Password (old)**: `<redacted>`
- **Password (new)**: `<redacted>`

### ISP Gateway (Converge)
- **URL**: http://192.168.100.1/
- **Model**: EG8245H5
- **Username**: `telcoadmin`
- **Password**: `<redacted>`

---

## Complete VLAN Configuration

**2026-10-11 IPAM repair:** DHCP, VLANs, leases and existing service addresses are unchanged.
Diallos's new-service candidates are `.36-.49` on VLAN 20/40/90, outside live DHCP `.50-.254`.
The initially untagged row is now recorded as confirmed internal mesh backhaul using complete history
and reciprocal live AP topology; both AP IPs remain excluded. Ordinary unresolved clients still fail closed.
All three pools return 14 candidates; the real allocator passed against live evidence in an isolated ledger.
The older IP-map expansion ranges are not proof of availability; see [10 - IP Address Map](10%20-%20IP%20Address%20Map.md) and

| VLAN ID | Name           | Network             | Gateway          | Purpose                      | DHCP Range |
| ------- | -------------- | ------------------- | ---------------- | ---------------------------- | ---------- |
| 1       | Default        | 192.168.0.0/24      | 192.168.0.1      | Management (temporary)       | .100-.199  |
| 10      | Internal       | 192.168.10.0/24     | 192.168.10.1     | Main LAN                     | .50-.254   |
| **20**  | **Homelab**    | **192.168.20.0/24** | **192.168.20.1** | **Proxmox/VMs**              | .50-.254   |
| 30      | IoT            | 192.168.30.0/24     | 192.168.30.1     | IoT devices                  | .50-.254   |
| **40**  | **Production** | **192.168.40.0/24** | **192.168.40.1** | **Docker/Apps**              | .50-.254   |
| 50      | Guest          | 192.168.50.0/24     | 192.168.50.1     | Guest WiFi                   | .50-.254   |
| 90      | Management     | 192.168.90.0/24     | 192.168.90.1     | Network devices, Pi-hole DNS | .50-.254   |

---

## Switch Port Configurations

### Core Switch (SG3210)

| Port | Device | Mode | Native VLAN | Tagged VLANs |
|------|--------|------|-------------|--------------|
| 1 | OC300 Controller | Trunk | VLAN 1 | All VLANs |
| 2 | OPNsense Port | Access | VLAN 90 | - |
| 5 | Zephyrus Port | Access | VLAN 10 | - |
| 6 | Morpheus Rack Uplink | Trunk | VLAN 1 | 10,20,30,40,50,90 |
| 7 | Kratos PC | Trunk | VLAN 10 | 20 (Hyper-V) |
| 8 | Atreus Switch Uplink | Trunk | VLAN 1 | All VLANs |

### Morpheus Switch (SG2210P)

| Port | Device | Mode | Native VLAN | Tagged VLANs |
|------|--------|------|-------------|--------------|
| 1 | Core Switch Uplink | Trunk | VLAN 1 | All |
| 2 | **Proxmox Node 01** | Trunk | VLAN 20 | 10, 40 |
| 5 | Computer Room EAP | Trunk | VLAN 1 | All SSIDs |
| 6 | **Proxmox Node 02** | Trunk | VLAN 20 | 10, 40 |
| 7 | Synology NAS (eth0) | Access | VLAN 10 | - |
| 8 | Synology NAS (eth1) | Access | VLAN 20 | - |

### Atreus Switch (ES20GP)

| Port | Device | Mode | Native VLAN | Tagged VLANs |
|------|--------|------|-------------|--------------|
| 1 | First Floor EAP | Trunk | VLAN 1 | All SSIDs |
| 5 | Core Router | Trunk | VLAN 1 | All |
| 6 | Core Switch | Trunk | VLAN 1 | All |

---

## WiFi SSID Configuration

| SSID | VLAN | Security Key |
|------|------|--------------|
| NKD5380-Internal | 10 | `<redacted>` |
| NHN7476-Homelab | 20 | `<redacted>` |
| WOC321-IoT | 30 | `<redacted>` |
| NAZ9229-Production | 40 | `<redacted>` |
| EAD6167-Guest | 50 | `<redacted>` |
| NCP5653-Management | 90 | `<redacted>` |

### Backup SSIDs
| SSID | Security Key |
|------|--------------|
| ARA2802 | `<redacted>` |
| NIR7714-MLO-Backup | `<redacted>` |

---

## Domain & SSL Configuration

- **Domain**: `hrmsmrflrii.xyz`
- **Registrar**: GoDaddy
- **Nameservers**: Cloudflare
- **SSL**: Let's Encrypt wildcard via Cloudflare DNS-01
- **Reverse Proxy**: [09 - Traefik Reverse Proxy](09%20-%20Traefik%20Reverse%20Proxy.md)
- **Internal DNS**: Technitium on homelab-dns01 (192.168.90.60, primary) + Pi-hole (192.168.90.53, secondary) — see [DNS Servers (Technitium primary, Pi-hole secondary)](#dns-servers-technitium-primary-pi-hole-secondary)

---

## Inter-VLAN ACLs and VLAN Flow

- **Gateway ACLs** (LAN→LAN only) are owned by Torrent (Roundtable network agent): rule set
  [64 - VLAN ACL Rule Set](64%20-%20VLAN%20ACL%20Rule%20Set.md), machine copy `agents/olympus/acl_ruleset.json`. **All Torrent ACLs disabled by the owner
  2026-10-16 ~12:05**: inter-VLAN traffic is currently unfiltered. Phases 2-4 had been applied 2026-10-15/16; from ~11:07
  on 2026-10-16 phase 4 blocked Omada device management (VLAN 90 <-> controller) and VLAN 40/IoT DNS to Pi-hole, because
  the permits at the top of the rule set were not taking effect (see
  [12 - Troubleshooting › Omada devices disconnected and slow DNS after an ACL rollout (2026-10-16)](12%20-%20Troubleshooting.md#omada-devices-disconnected-and-slow-dns-after-an-acl-rollout-2026-10-16)). Verify the live
  rule order before re-enabling. Changes need a Discord approval with an RFC and a second ✅; `@torrent acl off` disables
  every Torrent rule.
- **VLAN Flow** (https://vlanflow.hrmsmrflrii.xyz, docker-vm-core-utilities01 :8787) receives Omada syslog on UDP
  5514 and maps logged ACL decisions per VLAN pair, plus a raw feed of client and DHCP events.
- **192.168.90.60** was the OPNsense box; since 2026-10-18 the same mini PC runs Ubuntu as **homelab-dns01** (Technitium primary DNS). The OPNsense install is still on its eMMC but no longer boots.
- **Controller networks not in the table above**: VLAN 80 "HyperV VLAN" 192.168.80.0/24.
- **Removed by the owner 2026-10-16**: VLAN 60, VLAN 70 "VPN VLAN" and VLAN 91 "FirewallVLAN" (verified on the
  controller; live networks are now 1, 10, 20, 30, 40, 50, 80, 90).
- **DHCP DNS** (owner, 2026-10-18): VLANs 10/20/30/40/50/80/90 hand out 192.168.90.60 then 192.168.90.53 (the 8.8.8.8 secondaries are gone). **Default (VLAN 1) still hands out only 192.168.90.53.**

## Observability Endpoints

Internal endpoints for monitoring and tracing (See [18 - Observability Stack](18%20-%20Observability%20Stack.md)):

| Service | Endpoint | Purpose |
|---------|----------|---------|
| Traefik Metrics | 192.168.40.20:8082/metrics | Prometheus scrape target |
| OTEL Collector (gRPC) | 192.168.40.13:4317 | OTLP trace receiver |
| OTEL Collector (HTTP) | 192.168.40.13:4318 | OTLP trace receiver |
| OTEL Collector Metrics | 192.168.40.13:8888/metrics | Collector internal metrics |
| OTEL Pipeline Metrics | 192.168.40.13:8889/metrics | Pipeline exporter metrics |
| Jaeger | 192.168.40.13:16686 | Trace visualization UI |
| Jaeger Metrics | 192.168.40.13:14269/metrics | Jaeger internal metrics |

---

## Remote Access (Tailscale)

Tailscale provides secure remote access to the homelab from outside the local network using WireGuard encryption. node01 is configured as a **subnet router** to enable access to all VMs and containers.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Tailscale Network                            │
│                                                                      │
│   ┌──────────────┐         ┌──────────────┐         ┌─────────────┐ │
│   │   MacBook    │◄───────►│   node01     │◄───────►│   node02    │ │
│   │ 100.90.207.58│  WireGuard│ 100.89.33.5 │         │100.96.195.27│ │
│   └──────────────┘         │ SUBNET ROUTER│         └─────────────┘ │
│                            └──────┬───────┘                         │
│                                   │                                  │
└───────────────────────────────────┼──────────────────────────────────┘
                                    │ Advertises Routes
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
            ┌───────────┐   ┌───────────┐   ┌───────────┐
            │192.168.20 │   │192.168.40 │   │192.168.90 │
            │  /24      │   │  /24      │   │  /24      │
            │ Infra     │   │ Services  │   │ Mgmt/DNS  │
            └───────────┘   └───────────┘   └───────────┘
```

### Tailscale IP Mapping

| Device | Local IP | Tailscale IP | Role |
|--------|----------|--------------|------|
| node01 | 192.168.20.20 | 100.89.33.5 | **Subnet Router** |
| node02 | 192.168.20.21 | 100.96.195.27 | Peer |
| Synology NAS | 192.168.20.31 | 100.84.128.43 | Peer (inactive) |
| MacBook Pro | - | 100.90.207.58 | Client |

### Subnet Router Configuration

node01 advertises local networks to the Tailscale network:

| Network | Purpose | Example Hosts |
|---------|---------|---------------|
| 192.168.20.0/24 | Infrastructure VLAN | Proxmox nodes, Ansible, K8s |
| 192.168.40.0/24 | Services VLAN | Docker hosts, applications |
| 192.168.90.0/24 | Management VLAN | Network devices, Pi-hole DNS (192.168.90.53) |

**node01 Configuration:**
```bash
# IP forwarding (persisted in /etc/sysctl.d/99-tailscale.conf)
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Tailscale subnet router command
tailscale up --advertise-routes=192.168.20.0/24,192.168.40.0/24,192.168.90.0/24 --accept-routes
```

### Split DNS Configuration

DNS queries for `*.hrmsmrflrii.xyz` go to both DNS servers (two nameserver entries, same domain):

| Setting | Value |
|---------|-------|
| Nameservers | 192.168.90.60 (Technitium), 192.168.90.53 (Pi-hole) |
| Restricted to domain | hrmsmrflrii.xyz |
| Override local DNS | Enabled |

Configured in **Tailscale Admin Console → DNS tab**.

### Client Configuration

**macOS:**
```bash
# CLI path (not in PATH by default)
/Applications/Tailscale.app/Contents/MacOS/Tailscale

# Accept subnet routes
/Applications/Tailscale.app/Contents/MacOS/Tailscale up --accept-routes

# Optional: Add alias to ~/.zshrc
alias tailscale="/Applications/Tailscale.app/Contents/MacOS/Tailscale"
```

**Linux/Windows:**
```bash
tailscale up --accept-routes
```

### What Works Remotely

| Access Type | Method | Example |
|-------------|--------|---------|
| SSH to any VM/container | Local IP via subnet router | `ssh 192.168.40.13` |
| Web services | Domain name via split DNS | `https://grafana.hrmsmrflrii.xyz` |
| Proxmox Web UI | Tailscale IP or local IP | `https://192.168.20.20:8006` |
| Direct container access | Local IP + port | `http://192.168.40.13:3030` |

### Remote Access Commands

```bash
# SSH via local IPs (through subnet router)
ssh hermes-admin@192.168.20.30    # Ansible controller
ssh hermes-admin@192.168.40.13    # Docker utilities

# SSH via Tailscale IPs (direct)
ssh root@100.89.33.5              # node01
ssh root@100.96.195.27            # node02

# Access services via domain (with split DNS)
curl https://grafana.hrmsmrflrii.xyz
curl https://glance.hrmsmrflrii.xyz
```

### Troubleshooting

```bash
# Verify routes are accepted
tailscale status

# Test connectivity to subnets
ping 192.168.20.1    # VLAN 20 gateway
ping 192.168.40.1    # VLAN 40 gateway
ping 192.168.90.53   # Pi-hole DNS

# Test DNS resolution
nslookup grafana.hrmsmrflrii.xyz 192.168.90.53

# Check Tailscale DNS status (macOS)
/Applications/Tailscale.app/Contents/MacOS/Tailscale dns status

# Restart Tailscale (macOS)
sudo killall Tailscale tailscaled
open -a Tailscale
```

### Security Features

| Feature | Benefit |
|---------|---------|
| WireGuard encryption | All traffic encrypted end-to-end |
| No port forwarding | No inbound ports exposed to internet |
| Device authentication | Only authorized devices can join |
| ACL control | Access controlled via Tailscale admin |
| Split DNS | DNS queries stay within encrypted tunnel |

> [!tip] Service URLs vs Tailscale
> With the subnet router configured, both service URLs (`*.hrmsmrflrii.xyz`) and local IPs work from anywhere via Tailscale.

---

## Network Maintenance

### Omada Backup
- **FTP Path**: `/OmadaConfigBackup`
- **FTP Username**: `omada-log-admin`
- **FTP Password**: `<redacted>`

### DNS Servers (Technitium primary, Pi-hole secondary)
Since 2026-10-18 every client, node, LXC and VM asks **Technitium** first and **Pi-hole** second.

| | Technitium (primary) | Pi-hole (secondary) |
|---|---|---|
| Host | homelab-dns01, bare-metal mini PC, Ubuntu 26.04, static 192.168.90.60 (no DHCP) | CT202 on node01, 192.168.90.53 |
| Version | Technitium 15.6.2 (`dns.service`) | Pi-hole 6.4.3 + Unbound 127.0.0.1:5335 |
| Admin UI | https://dns.hrmsmrflrii.xyz (Traefik, own login `admin`) / http://192.168.90.60:5380 | http://192.168.90.53/admin |
| Resolution | Recursive (no forwarders), DNSSEC validation, QNAME minimisation, serve-stale | Unbound recursive |
| Blocking | Advanced Blocking app: StevenBlack + 3 hagezi lists, Pi-hole's regex denies and allowlist; blocked = 0.0.0.0 | gravity (same lists) |
| Local names | Zone `hrmsmrflrii.xyz` (Conditional Forwarder to this-server: local A records → Traefik 192.168.40.20, everything else public); zone `lan` (pihole, homelab-dns01, FS01, FS02) | `dns.hosts`, **copied from Technitium daily** by Torrent (`dnssync.py`, `dns-sync.timer` 05:50) |
| Monitoring | Prometheus jobs `technitium` (exporter :9274) and `node-homelab-dns01` (:9100); Torrent checks both servers every 10 min; Glance Network page | Torrent, Glance |

- **Add a local name**: add an A record in Technitium (zone `hrmsmrflrii.xyz` → 192.168.40.20 for Traefik routes). Pi-hole gets it at the next `dns-sync` (or `python3 ~/olympus/dnssync.py sync` on CT211). Do not edit Pi-hole's local DNS by hand; the sync keeps Technitium's list.
- **Setup is code**: `agents/dns/technitium_setup.py` (idempotent; `/root/technitium_setup.py` on the box) and `agents/dns/technitium_exporter.py`. Admin password and API tokens: <redacted> on homelab-dns01 (root only).
- **Plan**: replace Pi-hole with a second Technitium in a new LXC, join a Technitium cluster, then move 192.168.90.53 to it.

---

## Related Documentation

- [10 - IP Address Map](10%20-%20IP%20Address%20Map.md) - Complete IP allocation
- 11 - Credentials - All access credentials
- [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) - Node configurations
- [09 - Traefik Reverse Proxy](09%20-%20Traefik%20Reverse%20Proxy.md) - SSL and routing
- [18 - Observability Stack](18%20-%20Observability%20Stack.md) - OTEL and Jaeger tracing
- [17 - Monitoring Stack](17%20-%20Monitoring%20Stack.md) - Prometheus and Grafana
