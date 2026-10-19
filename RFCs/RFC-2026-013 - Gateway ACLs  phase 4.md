---
created: 2026-10-16
updated: 2026-10-16
type: project
status: seed
project-status: active
tags: [rfc, change-management, netops]
rfc-status: "Approved — implementing"
approval: 49
---
# RFC-2026-013 · Gateway ACLs → phase 4

| | |
|---|---|
| **Status** | Approved — implementing |
| **Performed by** | 🐎 Torrent (Network Engineer · Proxmox, Omada, Firewalls & ACLs) |
| **Governance** | Ranni |
| **Approver** | Master Hermes (✅ on Roundtable approval #49) |
| **Filed** | 2026-10-16 11:16 |

## What will be done

Torrent syncs the Torrent-managed Omada Gateway ACLs (LAN→LAN only) to phase 4 of `acl_ruleset.json` v7 (`f4faad410f6caee8`), creating rules in order after the needed IP and IP-port groups.

## What will change

35 rules at phase 4 (0 new) + 12 IP groups, 11 IP-port groups.

Internet traffic is never filtered; rules Torrent did not create are untouched. During the rebuild there are a few seconds with no Torrent rules (fail-open).

## Recovery plan

Automatic: health checks right after applying (controller, Pi-hole DNS, internet, SSH node01, Grafana/VLAN Flow); any failure, ❌, or no ✅ on the 'Keep these ACLs?' message within 15 min rebuilds the previous phase (4). Manual: `@torrent acl off` disables every Torrent rule at once. Last resort: Omada → Settings → Network Security → ACL → Gateway ACL, disable the `T###-` rules. ALLOW-Admin-Infra stays first so this PC keeps controller access.

## Risk

A missing permit could block a service between VLANs until rolled back; return traffic depends on the gateway's stateful handling, which the owner verifies from VLAN 10 during the confirmation window.

## Closure

_Open — Ranni closes this automatically when the approved task ends._

Register: [RFC Register](RFC%20Register.md)
