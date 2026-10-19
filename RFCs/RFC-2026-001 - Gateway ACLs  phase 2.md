---
created: 2026-10-15
updated: 2026-10-15
type: project
status: seed
project-status: done
tags: [rfc, change-management, netops]
rfc-status: "Implemented"
approval: 32
---
# RFC-2026-001 · Gateway ACLs → phase 2

| | |
|---|---|
| **Status** | Implemented |
| **Performed by** | 🐎 Torrent (Network Engineer · Proxmox, Omada, Firewalls & ACLs) |
| **Governance** | Ranni |
| **Approver** | Master Hermes (✅ on Roundtable approval #32) |
| **Filed** | 2026-10-15 22:08 |

## What will be done

Torrent syncs the Torrent-managed Omada Gateway ACLs (LAN→LAN only) to phase 2 of `acl_ruleset.json` v1 (`355d806099625141`), creating rules in order after the needed IP and IP-port groups.

## What will change

6 rules at phase 2 (6 new) + 1 IP groups, 2 IP-port groups.
`T050-ALLOW-Admin-Infra` ✅ permit G_ADMIN_PC → VLAN 1, VLAN 20, VLAN 40, VLAN 90, VLAN 91 ALL
`<redacted>` ✅ permit G_ADMIN_PC → VLAN 80 TCPUDP 3389
`<redacted>` ✅ permit VLAN 10 → VLAN 60 ALL
`<redacted>` ✅ permit VLAN 60 → VLAN 10 TCP 3400,3401,3500
`<redacted>` ✅ permit VLAN 10 → VLAN 1, VLAN 20, VLAN 40, VLAN 90 ALL
`T250-DENY-to-Internal` ⛔ deny VLAN 1, VLAN 20, VLAN 30, VLAN 40, VLAN 50, VLAN 70, VLAN 80, VLAN 90, VLAN 91 → VLAN 10 ALL
Internet traffic is never filtered; rules Torrent did not create are untouched. During the rebuild there are a few seconds with no Torrent rules (fail-open).

## Recovery plan

Automatic: health checks right after applying (controller, Pi-hole DNS, internet, SSH node01, Grafana/VLAN Flow); any failure, ❌, or no ✅ on the 'Keep these ACLs?' message within 15 min rebuilds the previous phase (0). Manual: `@torrent acl off` disables every Torrent rule at once. Last resort: Omada → Settings → Network Security → ACL → Gateway ACL, disable the `T###-` rules. ALLOW-Admin-Infra stays first so this PC keeps controller access.

## Risk

A missing permit could block a service between VLANs until rolled back; return traffic depends on the gateway's stateful handling, which the owner verifies from VLAN 10 during the confirmation window.

## Closure

**Implemented** — closed by Ranni on 2026-10-15 22:14.

```
2026-10-15 22:08:54 [netops] acl: removed 0 managed rules
2026-10-15 22:08:54 [netops] acl: created group G_ADMIN_PC
2026-10-15 22:08:55 [netops] acl: created group TP060
2026-10-15 22:08:55 [netops] acl: created group TP120
2026-10-15 22:08:56 [netops] acl: created 6 rules in order

```

Register: [RFC Register](RFC%20Register.md)
