---
created: 2026-10-15
updated: 2026-10-18
type: project
status: seed
project-status: active
tags: [homelab, networking, security, omada, acl]
related: ["[01 - Network Architecture](01%20-%20Network%20Architecture.md)", "[10 - IP Address Map](10%20-%20IP%20Address%20Map.md)", "[41 - Omada ACL Implementation Guide](41%20-%20Omada%20ACL%20Implementation%20Guide.md)"]
---
# VLAN ACL Rule Set

> [!WARNING] 2026-10-16 ~12:05: all Torrent ACLs disabled by the owner
> Phase 4 (applied 10:57, rebuilt after the failed v6 attempt at 11:10) blocked VLAN 90 <-> Omada controller
> (devices disconnected) and VLAN 40/IoT DNS to Pi-hole (slow apps); `ALLOW-DNS-PiHole`, `ALLOW-Omada-Adopt` and
> `ALLOW-Omada-Manage` did not match. No internet traffic was blocked. Before re-enabling: compare the live rule order
> with `acl_ruleset.json`, and add VLAN 90 -> controller and VLAN 40/30 DNS health checks that also run after rollbacks.
> The status notes below are history. Evidence: Homelab Changelog 2026-10-16.

> **2026-10-16: rule set v5.** The owner removed VLANs 70 (VPN) and 91 (Firewall) and the old speaker VLAN 60
> (those devices are on a separate network, outside this lab). Omada dropped the rules that only referenced them;
> 9 managed rules remain live. v5 (`agents/olympus/acl_ruleset.json`, 37 rules, deployed to CT211) removes those
> rules plus `DENY-Internal-Rest`, and strips 60/70/91 from every other rule; the tables below match v5. Live
> `T050-ALLOW-Admin-Infra` still holds VLAN 91's dangling network ID until Torrent's next rollout.

> **Status (2026-10-15 22:27): phase 3 live** (approval #33, [RFC-2026-002](RFCs/RFC%20Register.md) Implemented, rule set v2, 14 rules): Guest and IoT isolation active; Guest may use Pi-hole DNS only. Earlier: **phase 2 live 22:14.** Torrent applied the admin, VLAN 10 and deny-into-VLAN-10 rules (`T050-ALLOW-Admin-Infra`,
> `<redacted>`, `<redacted>`, `T250-DENY-to-Internal`) under approval #32 and
> [RFC-2026-001](RFCs/RFC-2026-001%20-%20Gateway%20ACLs%20%20phase%202.md) (Implemented). VLAN Flow logged the denies into VLAN 10
> and the admin PC's permits, so the ER605 enforces them (the controller's `aclDisable` flag did not matter).
> **Phase 4 is pending** (service dependencies, Hyper-V, agents, backup and the remaining per-VLAN denies).
> Rules marked **DECIDE** stay disabled until the owner decides. Replaces the 2026-01 draft in
> [41 - Omada ACL Implementation Guide](41%20-%20Omada%20ACL%20Implementation%20Guide.md).

Related: [01 - Network Architecture](01%20-%20Network%20Architecture.md) · [10 - IP Address Map](10%20-%20IP%20Address%20Map.md) · [07 - Deployed Services](07%20-%20Deployed%20Services.md) · VLAN Flow dashboard: https://vlanflow.hrmsmrflrii.xyz

## Who applies it: Torrent

Torrent (Roundtable network agent) configures these rules himself from `agents/olympus/acl_ruleset.json`, the
machine-readable copy of this note. Each phase is posted as an approval; after your ✅ he applies it, runs health
checks and waits 15 minutes for a second ✅ before keeping it; otherwise he rolls back. RFCs are filed automatically ([RFC Register](RFCs/RFC%20Register.md)). Commands in #roundtable-hold:
`@torrent probe the ACL API` (once) · `@torrent apply phase 2|3|4` · `@torrent acl status` · `@torrent acl off`.
Future changes: ask Torrent; he proposes an edit to the rule set and applies it the same way. If this note and the
JSON disagree, the JSON is what is on the controller. See 60 - Roundtable Agent Team.

## Rollout history

| When (Manila) | Approval | Result |
|---|---|---|
| 2026-10-15 21:33 | #29 probe | ✅ Controller body format learned with disabled throw-away rules |
| 2026-10-15 21:35 | #30 phase 2 | ❌ Omada refused the 15th IP group (limit 16, 2 pre-existing). No rules created; tool fixed |
| 2026-10-15 21:41 | #31 phase 2 | ↩️ Applied and healthy, rolled back at 21:57: "Keep these ACLs?" got no ✅ |
| 2026-10-15 22:08 | #32 phase 2 · RFC-2026-001 | ✅ Applied, confirmed and kept |
| 2026-10-15 22:24 | #33 phase 3 · RFC-2026-002 | ✅ 14 rules applied (v2: Guest → Pi-hole DNS only), confirmed and kept |

**Two ✅ per rollout:** one on the approval (starts the change) and one on Torrent's "🌈 Keep these ACLs?" message
after testing from VLAN 10 (keeps it). No second ✅ within 15 minutes means automatic rollback.

## Guest: Pi-hole DNS only (owner decision 2026-10-15)

Guest gets **Pi-hole 192.168.90.53 as primary DHCP DNS** (8.8.8.8 secondary). On the owner's request, rule 1 now
includes Guest: guests may reach **only 192.168.90.53 on TCP/UDP 53**, and `DENY-Guest-Out` blocks everything else
toward every internal VLAN (rule set v2). Guests keep normal internet access and Pi-hole filtering. No DHCP change is
needed. Effective once phase 3 is applied.

## Controller limits

- The ER605/controller holds **16 IP groups** in total; `IPGroup_Any` and an older `OPNsense_DNS` (192.168.90.30)
  use two. Torrent creates IP groups only for sources and port-less destinations, and identical destination+port
  sets share one IP-port group: phase 2 needs 1 + 2, phase 4 needs 12 + 12.
- During a rebuild there are a few seconds with no Torrent rules (fail-open), never a fail-closed lockout.

## Why some permits don't log

> **Owner decision 2026-10-15 (rule set v4):** logging was switched on in the Omada portal for `T010-ALLOW-DNS-PiHole`
> and `<redacted>`, and adopted into
> `acl_ruleset.json` v4 so Torrent keeps it. Expect VLAN Flow (7 days / 100k events) to cycle much faster. Make future
> hand edits the same way, or Torrent's next rollout undoes them: `@torrent acl status` now reports any drift.

Every deny logs (enforced by Torrent's safety checks). Permits log only where access is powerful: the admin PC and
RDP, IoT managers, backup, Ansible, the SRE agent, Proxmox/Tailscale and Helios. High-volume permits (all of VLAN 10
into services, DNS, NFS, monitoring, Traefik backends, Hyper-V ↔ Production) don't log: they would flood
VLAN Flow (7 days / 100k events), push out the deny evidence and add router load, and `ALLOW-Syslog` logging would
loop on itself. To log one temporarily, ask Torrent; it goes through approval and an RFC.

## Owner requirements (non-negotiable)

1. **This PC (Kratos, `192.168.10.10`)** manages and reaches Homelab (20), Production (40) and Management (90).
2. Those VLANs **cannot reach VLAN 10**.
3. **IoT (30) is completely isolated**, except selected VLAN 10 devices (Samsung phone) that manage IoT devices.
4. IoT devices stay reachable and manageable **from the internet** (vendor cloud apps).
5. **Backup reaches all services** to back them up.
6. **Agents** can manage their own responsibilities.
8. **Guest (50) is completely isolated.**
9. **Hyper-V (80) and Production (40) talk to each other.**
10. **RDP to Hyper-V (80) only from this PC.**
11. **Every VLAN 10 device** reaches the media stack, agents, network management, DNS and all services (VLANs 1, 20, 40, 90). This is **one-way**: nothing starts a connection into VLAN 10 unless that service must call back (added 2026-10-15).

## How Omada Gateway ACLs behave (read before building)

- **Gateway ACLs only see routed traffic between VLANs.** Traffic inside one VLAN (node ↔ NAS on VLAN 20, Traefik ↔ apps on VLAN 40) never reaches the gateway, so the old "VLAN 40 internal" / "Proxmox inter-node" rules are unnecessary.
- **First match wins, top to bottom.** No match = permitted, so each VLAN needs an explicit deny at the end.
- **Return traffic — verify first.** Requirement 2 (deny 20/40/90 → 10) must block only *new* connections, or replies to Kratos are dropped too. If the rule editor shows **Advanced → States**, set deny rules to **New** only. Phase 1 below tests this before anything else depends on it.
- **Logging** feeds [VLAN Flow](https://vlanflow.hrmsmrflrii.xyz) (Omada syslog → `192.168.40.13:5514`). Enable it on every **deny** and on the admin/agent/backup **permits**; not on DNS (too noisy).

## IP groups

Settings → Profiles → Groups → IP Group.

| Group | Members | Notes |
|---|---|---|
| `G_ADMIN_PC` | 192.168.10.10 | Kratos — static, outside DHCP |
| `G_IOT_MANAGERS` | Samsung phone IP, 192.168.10.10 | **DECIDE**: the phone needs a DHCP reservation and **"Use device MAC"** for the Internal SSID (Android randomizes MACs per network) |
| `G_PIHOLE` | 192.168.90.53, 192.168.90.60 | Both DNS servers since rule set v8 (2026-10-18): Pi-hole and Technitium (homelab-dns01). v8 also allows 5380 (Technitium UI) in ALLOW-Dashboards/ALLOW-Traefik-Backends and 9274 (exporter) in ALLOW-Monitoring-TCP. Not rolled out: all Torrent ACLs are disabled |
| `G_OMADA_CTRL` | 192.168.0.101 | |
| `G_PVE` | 192.168.20.20–22 | node01 is also the Tailscale subnet router |
| `G_NAS` | 192.168.20.32 | .31 NIC is dead |
| `G_BACKUP` | 192.168.20.50 | PBS — **fix Task #65 first** (see prerequisites) |
| `G_ANSIBLE` | 192.168.20.30 | Ansible + Omada exporter |
| `G_SRE` | 192.168.40.16 | codex-agent-lxc: SRE executor/Roundtable — owner granted full access (Task #63) |
| `G_MONITOR` | 192.168.40.13 | Prometheus, Uptime Kuma, Sentinel, VLAN Flow, syslog intake |
| `G_HELIOS` | 192.168.40.14 | Helios control plane |
| `G_DASH` | 192.168.40.12, 192.168.40.27 | Glance, Homepage widgets |
| `G_TRAEFIK` | 192.168.40.20 | Reverse proxy |
| `G_NFS_CLIENTS` | 192.168.40.11, 192.168.40.22 | Media stack, Immich |
| `G_SYSLOG` | 192.168.40.5, 192.168.40.13 | Central syslog, VLAN Flow |
| `G_RUNNER` | 192.168.40.24 | GitLab runner |

## Rule set (Gateway ACL, LAN → LAN, in this order)

Port groups: `P_OMADA` = TCP+UDP 29810–29817 · `P_MON` = TCP 22, 8006, 8007, 9100, 9101, 5000–5001 + UDP 161 · `P_RDP` = TCP+UDP 3389 · `P_NFS` = TCP+UDP 111, 2049.

### A. Core plumbing

| # | Name | Action | Source | Destination | Protocol / ports | Log |
|---|---|---|---|---|---|---|
| 1 | `ALLOW-DNS-PiHole` | Permit | Networks 1,10,20,30,40,50,80,90 | `G_PIHOLE` | TCP/UDP 53 | Yes (v4) |
| 2 | `ALLOW-Omada-Adopt` | Permit | Network 90 | `G_OMADA_CTRL` | `P_OMADA` | No |
| 3 | `ALLOW-Omada-Manage` | Permit | `G_OMADA_CTRL` | Network 90 | All | No |
| 4 | `ALLOW-Syslog` | Permit | Network 90, `G_OMADA_CTRL` | `G_SYSLOG` | UDP 514, 5514 | No |

Guest (50) is included in rule 1 (owner decision 2026-10-15): Pi-hole DNS is the only internal service guests can reach.

### B. Admin PC (requirements 1, 10)

| # | Name | Action | Source | Destination | Protocol / ports | Log |
|---|---|---|---|---|---|---|
| 5 | `ALLOW-Admin-Infra` | Permit | `G_ADMIN_PC` | Networks 1, 20, 40, 90 | All | Yes |
| 6 | `ALLOW-Admin-RDP-HyperV` | Permit | `G_ADMIN_PC` | Network 80 | `P_RDP` | Yes |

### C. IoT management (requirements 3, 4)

| # | Name | Action | Source | Destination | Protocol / ports | Log |
|---|---|---|---|---|---|---|
| 7 | `ALLOW-IoT-Managers` | Permit | `G_IOT_MANAGERS` | Network 30 | All | Yes |
| 8 | `ALLOW-HA-IoT` *(DECIDE)* | Permit | 192.168.40.25 | Network 30 | All | Yes |

Internet management needs **no inbound rule**: IoT devices keep outbound internet (their cloud accounts), and the vendor apps work from anywhere. Do **not** port-forward to VLAN 30. Rule 8 only matters if Home Assistant controls IoT devices locally; leaving it out keeps IoT fully isolated, as the requirement states.

### D. Backup and storage (requirement 5)

| # | Name | Action | Source | Destination | Protocol / ports | Log |
|---|---|---|---|---|---|---|
| 9 | `ALLOW-Backup-All` | Permit | `G_BACKUP` | Networks 40, 80, 90 | All | Yes |
| 10 | `ALLOW-NFS-Clients` | Permit | `G_NFS_CLIENTS` | `G_NAS` | `P_NFS` | No |

Proxmox guest backups run node → PBS inside VLAN 20, so no ACL is involved. Rule 9 covers application-level backups. Hyper-V VMs on Kratos are **not** protected by PBS; that needs a separate Windows-side backup.

### F. Hyper-V ↔ Production (requirement 9)

| # | Name | Action | Source | Destination | Protocol / ports | Log |
|---|---|---|---|---|---|---|
| 13 | `ALLOW-Prod-to-HyperV` | Permit | Network 40 | Network 80 | All | No |
| 14 | `ALLOW-HyperV-to-Prod` | Permit | Network 80 | Network 40 | All | No |

Risk to note: anything in VLAN 80 can reach every production host, including codex-agent-lxc, which holds SSH keys to the whole lab. Narrow rule 14 to specific hosts and ports once the Hyper-V workload is known.

### G. Agents and automation (requirement 6)

| # | Name | Action | Source | Destination | Protocol / ports | Log |
|---|---|---|---|---|---|---|
| 15 | `ALLOW-Ansible-Targets` | Permit | `G_ANSIBLE` | Networks 40, 90 | All | Yes |
| 16 | `ALLOW-Ansible-Omada` | Permit | `G_ANSIBLE` | `G_OMADA_CTRL` | TCP 443 | No |
| 17 | `ALLOW-SRE-Agent` | Permit | `G_SRE` | Networks 20, 90; `G_OMADA_CTRL` | All | Yes |
| 18 | `ALLOW-PVE-Tailscale` | Permit | `G_PVE` | Networks 40, 90 | All | Yes |
| 19 | `ALLOW-Monitoring` | Permit | `G_MONITOR` | Networks 20, 90; `G_OMADA_CTRL` | `P_MON`, ICMP | No |
| 20 | `ALLOW-Helios` | Permit | `G_HELIOS` | `G_PVE`, `G_NAS`, `G_OMADA_CTRL` | TCP 22, 443, 5000–5001, 8006 | Yes |
| 21 | `ALLOW-Dashboards` | Permit | `G_DASH` | `G_PVE`, `G_BACKUP`, `G_NAS`, `G_PIHOLE` | TCP 80, 443, 5000–5001, 8006, 8007 | No |
| 22 | `ALLOW-Traefik-Backends` | Permit | `G_TRAEFIK` | `G_PVE`, `G_BACKUP`, `G_PIHOLE`, `G_OMADA_CTRL` | TCP 80, 443, 8006, 8007 | No |
| 23 | `ALLOW-Runner-Deploy` *(DECIDE)* | Permit | `G_RUNNER` | Network 20 | TCP 22 | Yes |

Rule 18: Tailscale on node01 source-NATs remote clients, so traffic from your MacBook on the tailnet arrives as `192.168.20.20`. Without this rule, remote access to VLAN 40/90 stops working. It also means any tailnet device gets node01's reach, so keep Tailscale's own ACLs tight.

### H. All VLAN 10 devices: one-way into services (requirement 11)

| # | Name | Action | Source | Destination | Protocol / ports | Log |
|---|---|---|---|---|---|---|
| 24 | `ALLOW-Internal-Services` | Permit | Network 10 | Networks 1, 20, 40, 90 | All | No |

Any VLAN 10 device can open media, agents, network management, DNS and every other service, by name or by IP. Replies come back, but nothing in those VLANs can **start** a connection into VLAN 10 (rule 25). Hyper-V RDP (rule 6) and IoT (rule 7) stay limited to their groups.

**Reverse exceptions (things that must call back into VLAN 10):**

| Need | Rule | Status |
|---|---|---|
| Helios → Kratos WinRM for Hyper-V (Task #44) | would need `G_HELIOS` → `G_ADMIN_PC` TCP 5985/5986 | Not included — #44 is parked |
| Roundtable voice server on Kratos :8765 | none — the browser on Kratos connects to `ws://127.0.0.1:8765` | No rule needed |
| Jellyfin/DLNA casting *to* a VLAN 10 TV or speaker | would need media host → device | Not included — add only if used |

Add any further exception here as a narrow host-and-port permit **above** rule 25, never as a VLAN-wide allow.

### I. Default denies (enable Logging; States = New where offered)

| # | Name | Action | Source | Destination |
|---|---|---|---|---|
| 25 | `DENY-to-Internal` | Deny | Networks 1, 20, 30, 40, 50, 80, 90 | Network 10 |
| 26 | `DENY-IoT-Out` | Deny | Network 30 | Networks 1, 20, 40, 50, 80, 90 |
| 27 | `DENY-Guest-Out` | Deny | Network 50 | Networks 1, 10, 20, 30, 40, 80, 90 |
| 30 | `DENY-to-IoT` | Deny | All other networks | Network 30 |
| 32 | `DENY-to-HyperV` | Deny | Networks 1, 10, 20, 90 | Network 80 |
| 34 | `DENY-Prod-Rest` | Deny | Network 40 | Networks 1, 20, 90 |
| 35 | `DENY-Homelab-Rest` | Deny | Network 20 | Networks 1, 40, 90 |
| 36 | `DENY-Mgmt-Rest` | Deny | Network 90 | Networks 1, 20, 40 |
| 37 | `DENY-HyperV-Rest` | Deny | Network 80 | Networks 1, 20, 90 |
| 38 | `DENY-Default-Rest` | Deny | Network 1 | Networks 20, 40, 90 |

Internet access is untouched; these are LAN → LAN rules only.

## Prerequisites (blockers)

1. **Task #65: PBS address squatter is the Zephyrus switch.** Its MAC `78:20:51:8A:E3:7B` (Zephyrus Switch, SG3210X-M2) is the device that held `192.168.20.50`. On 2026-10-15 the gateway also handed it `192.168.30.53` on IoT, so its management interface is taking DHCP leases on several VLANs. Give it a single static management IP on VLAN 90, and add an IP-MAC binding for PBS. Otherwise rule 9 grants backup-level access to whatever holds `.20.50`.
3. **Confirm the use of VLANs 70 (VPN, 192.168.70.0/24), 80 (Hyper-V, 192.168.80.0/24) and 91 (Firewall, 192.168.91.0/24)**: all three exist on the controller (checked 2026-10-15), but their current use is undocumented. 70 and 80 are not in [01 - Network Architecture](01%20-%20Network%20Architecture.md), and Kratos's switch port tags VLAN 20 for Hyper-V, not 80. VLAN 91 looks empty because OPNsense actually sits at `192.168.90.60` on VLAN 90; the rules keep 91 so they stay correct if it is ever used.
4. **Samsung phone**: DHCP reservation and fixed MAC (see `G_IOT_MANAGERS`).
5. **Helios Hyper-V support (Task #44)** needs WinRM *into* Kratos on VLAN 10, which requirement 2 forbids. Leave #44 parked, or run Helios's Hyper-V calls from Kratos instead.

## Rollout (phased, with checks)

1. **Groups only.** Create all IP and port groups. Nothing changes yet.
2. **Admin and VLAN 10 first: rules 5, 6, 24, then 25.** From Kratos, verify SSH to node01, Proxmox UI, Pi-hole UI, the Omada controller and `vlanflow.hrmsmrflrii.xyz`. **If Kratos loses replies after rule 25, the gateway is matching statelessly. Disable rule 25 and stop** until return traffic can be handled (state matching, or routing inter-VLAN through OPNsense).
3. **Untrusted VLANs: rules 1, 26–28, 30, 31.** Test the phone → IoT app, guest isolation.
4. **Services: rules 2–4, 7–24, then 32–38.** Watch VLAN Flow's **Denied only** view for 24 hours. Every unexpected deny is a missing permit.
5. **Rollback:** disable rules bottom-up (deny rules first). Rule 5 stays enabled throughout so Kratos is never locked out of the controller.

## Verification checklist

| Test | Expected |
|---|---|
| Kratos → `ssh node01`, `https://192.168.20.20:8006`, `http://192.168.90.53/admin` | Works |
| node01 / any VLAN 40 host → `ping 192.168.10.10` | Blocked (logged deny, rule 25) |
| Any VLAN 10 phone/laptop → Jellyfin, Roundtable, Grafana, Proxmox UI, Pi-hole, Omada (by name and IP) | Works, and keeps working after rule 25 (proves replies pass) |
| Kratos → `mstsc /v:<VLAN 80 host>` | Works; RDP from any other device fails |
| Samsung phone → IoT app (local and on mobile data) | Works |
| Any other VLAN 10 device → IoT device IP | Blocked |
| Guest phone → `192.168.40.20`, `192.168.90.53` | Blocked; internet works |
| PBS → a VLAN 40 host | Works |
| Grafana node and PBS panels, Glance and Homepage widgets, Sentinel `/check` | Still populated |
| Tailscale MacBook → `https://grafana.hrmsmrflrii.xyz`, `ssh 192.168.40.13` | Works |
