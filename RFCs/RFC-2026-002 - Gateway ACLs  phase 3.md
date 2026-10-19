---
created: 2026-10-15
updated: 2026-10-15
type: project
status: seed
project-status: done
tags: [rfc, change-management, netops]
rfc-status: "Implemented"
approval: 33
---
# RFC-2026-002 · Gateway ACLs → phase 3

| | |
|---|---|
| **Status** | Implemented |
| **Performed by** | 🐎 Torrent (Network Engineer · Proxmox, Omada, Firewalls & ACLs) |
| **Governance** | Ranni |
| **Approver** | Master Hermes (✅ on Roundtable approval #33) |
| **Filed** | 2026-10-15 22:23 |

## What will be done

Torrent syncs the Torrent-managed Omada Gateway ACLs (LAN→LAN only) to phase 3 of `acl_ruleset.json` v2 (`410b1be596a311d1`), creating rules in order after the needed IP and IP-port groups.

## What will change

14 rules at phase 3 (8 new) + 2 IP groups, 3 IP-port groups.
`T010-ALLOW-DNS-PiHole` ✅ permit VLAN 1, VLAN 10, VLAN 20, VLAN 30, VLAN 40, VLAN 50, VLAN 60, VLAN 70, VLAN 80, VLAN 90, VLAN 91 → G_PIHOLE TCPUDP 53
`T070-ALLOW-IoT-Managers` ✅ permit G_IOT_MANAGERS → VLAN 30 ALL
`T260-DENY-IoT-Out` ⛔ deny VLAN 30 → VLAN 1, VLAN 20, VLAN 40, VLAN 50, VLAN 60, VLAN 70, VLAN 80, VLAN 90, VLAN 91 ALL
`T270-DENY-Guest-Out` ⛔ deny VLAN 50 → VLAN 1, VLAN 10, VLAN 20, VLAN 30, VLAN 40, VLAN 60, VLAN 70, VLAN 80, VLAN 90, VLAN 91 ALL
`T280-DENY-Sonos-Out` ⛔ deny VLAN 60 → VLAN 1, VLAN 10, VLAN 20, VLAN 30, VLAN 40, VLAN 50, VLAN 70, VLAN 80, VLAN 90, VLAN 91 ALL
`T290-DENY-VPN-Out` ⛔ deny VLAN 70 → VLAN 1, VLAN 10, VLAN 20, VLAN 30, VLAN 40, VLAN 50, VLAN 60, VLAN 80, VLAN 90, VLAN 91 ALL
`T300-DENY-to-IoT` ⛔ deny VLAN 1, VLAN 10, VLAN 20, VLAN 40, VLAN 50, VLAN 60, VLAN 70, VLAN 80, VLAN 90, VLAN 91 → VLAN 30 ALL
`T310-DENY-to-Sonos` ⛔ deny VLAN 1, VLAN 20, VLAN 40, VLAN 70, VLAN 80, VLAN 90, VLAN 91 → VLAN 60 ALL
Internet traffic is never filtered; rules Torrent did not create are untouched. During the rebuild there are a few seconds with no Torrent rules (fail-open).

## Recovery plan

Automatic: health checks right after applying (controller, Pi-hole DNS, internet, SSH node01, Grafana/VLAN Flow); any failure, ❌, or no ✅ on the 'Keep these ACLs?' message within 15 min rebuilds the previous phase (2). Manual: `@torrent acl off` disables every Torrent rule at once. Last resort: Omada → Settings → Network Security → ACL → Gateway ACL, disable the `T###-` rules. ALLOW-Admin-Infra stays first so this PC keeps controller access.

## Risk

A missing permit could block a service between VLANs until rolled back; return traffic depends on the gateway's stateful handling, which the owner verifies from VLAN 10 during the confirmation window.

## Closure

**Implemented** — closed by Ranni on 2026-10-15 22:27.

```
2026-10-15 22:24:25 [netops] acl: removed 6 managed rules
2026-10-15 22:24:25 [netops] acl: created group G_IOT_MANAGERS
2026-10-15 22:24:25 [netops] acl: created group TP010
2026-10-15 22:24:27 [netops] acl: created 14 rules in order

```

Register: [RFC Register](RFC%20Register.md)
