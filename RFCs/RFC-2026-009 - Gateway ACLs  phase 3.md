---
created: 2026-10-16
updated: 2026-10-16
type: project
status: seed
project-status: done
tags: [rfc, change-management, netops]
rfc-status: "Implemented"
approval: 45
---
# RFC-2026-009 · Gateway ACLs → phase 3

| | |
|---|---|
| **Status** | Implemented |
| **Performed by** | 🐎 Torrent (Network Engineer · Proxmox, Omada, Firewalls & ACLs) |
| **Governance** | Ranni |
| **Approver** | Master Hermes (✅ on Roundtable approval #45) |
| **Filed** | 2026-10-16 10:50 |

## What will be done

Torrent syncs the Torrent-managed Omada Gateway ACLs (LAN→LAN only) to phase 3 of `acl_ruleset.json` v5 (`9f0643ac92a949e6`), creating rules in order after the needed IP and IP-port groups.

## What will change

9 rules at phase 3 (0 new) + 2 IP groups, 2 IP-port groups.

Internet traffic is never filtered; rules Torrent did not create are untouched. During the rebuild there are a few seconds with no Torrent rules (fail-open).

## Recovery plan

Automatic: health checks right after applying (controller, Pi-hole DNS, internet, SSH node01, Grafana/VLAN Flow); any failure, ❌, or no ✅ on the 'Keep these ACLs?' message within 15 min rebuilds the previous phase (3). Manual: `@torrent acl off` disables every Torrent rule at once. Last resort: Omada → Settings → Network Security → ACL → Gateway ACL, disable the `T###-` rules. ALLOW-Admin-Infra stays first so this PC keeps controller access.

## Risk

A missing permit could block a service between VLANs until rolled back; return traffic depends on the gateway's stateful handling, which the owner verifies from VLAN 10 during the confirmation window.

## Closure

**Implemented** — closed by Ranni on 2026-10-16 10:55.

```
2026-10-16 10:50:40 [netops] acl: removed 9 managed rules
2026-10-16 10:50:40 [netops] acl: removed unused group TP120
2026-10-16 10:50:41 [netops] acl: created 9 rules in order

```

Register: [RFC Register](RFC%20Register.md)
