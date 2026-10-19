---
created: 2026-10-15
updated: 2026-10-15
type: project
status: seed
project-status: done
tags: [rfc, change-management, netops]
rfc-status: "Failed"
approval: 35
---
# RFC-2026-004 · Gateway ACLs → phase 4

| | |
|---|---|
| **Status** | Failed |
| **Performed by** | 🐎 Torrent (Network Engineer · Proxmox, Omada, Firewalls & ACLs) |
| **Governance** | Ranni |
| **Approver** | Master Hermes (✅ on Roundtable approval #35) |
| **Filed** | 2026-10-15 22:45 |

## What will be done

Torrent syncs the Torrent-managed Omada Gateway ACLs (LAN→LAN only) to phase 4 of `acl_ruleset.json` v2 (`410b1be596a311d1`), creating rules in order after the needed IP and IP-port groups.

## What will change

41 rules at phase 4 (27 new) + 12 IP groups, 12 IP-port groups.
`T020-ALLOW-Omada-Adopt` ✅ permit VLAN 90 → G_OMADA_CTRL TCPUDP 29810-29817
`T030-ALLOW-Omada-Manage` ✅ permit G_OMADA_CTRL → VLAN 90 ALL
`<redacted>` ✅ permit VLAN 90 → G_SYSLOG UDP 514,5514
`<redacted>` ✅ permit G_OMADA_CTRL → G_SYSLOG UDP 514,5514
`T090-ALLOW-Backup-All` ✅ permit G_BACKUP → VLAN 40, VLAN 80, VLAN 90 ALL
`T100-ALLOW-NFS-Clients` ✅ permit G_NFS_CLIENTS → G_NAS TCPUDP 111,2049
`<redacted>` ✅ permit VLAN 40 → VLAN 80 ALL
`<redacted>` ✅ permit VLAN 80 → VLAN 40 ALL
`<redacted>` ✅ permit G_ANSIBLE → VLAN 40, VLAN 90 ALL
`<redacted>` ✅ permit G_ANSIBLE → G_OMADA_CTRL TCP 443
`T171-ALLOW-SRE-Agent` ✅ permit G_SRE → VLAN 20, VLAN 90 ALL
`T172-ALLOW-SRE-Omada` ✅ permit G_SRE → G_OMADA_CTRL ALL
`<redacted>` ✅ permit G_PVE → VLAN 40, VLAN 90 ALL
`<redacted>` ✅ permit G_MONITOR → VLAN 20, VLAN 90 TCP 22,80,443,5000-5001,8006,8007,9100,9101
`<redacted>` ✅ permit G_MONITOR → VLAN 20, VLAN 90 UDP 161
`<redacted>` ✅ permit G_MONITOR → VLAN 20, VLAN 90 ICMP
`<redacted>` ✅ permit G_MONITOR → G_OMADA_CTRL TCP 443
`T200-ALLOW-Helios` ✅ permit G_HELIOS → G_PVE, G_NAS, G_OMADA_CTRL TCP 22,443,5000-5001,8006
`T210-ALLOW-Dashboards` ✅ permit G_DASH → G_PVE, G_BACKUP, G_NAS, G_PIHOLE TCP 80,443,5000-5001,8006,8007
`<redacted>` ✅ permit G_TRAEFIK → G_PVE, G_BACKUP, G_PIHOLE, G_OMADA_CTRL TCP 80,443,8006,8007
`T320-DENY-to-HyperV` ⛔ deny VLAN 1, VLAN 10, VLAN 20, VLAN 90, VLAN 91 → VLAN 80 ALL
`T330-DENY-Internal-Rest` ⛔ deny VLAN 10 → VLAN 91 ALL
`T340-DENY-Prod-Rest` ⛔ deny VLAN 40 → VLAN 1, VLAN 20, VLAN 90, VLAN 91 ALL
`T350-DENY-Homelab-Rest` ⛔ deny VLAN 20 → VLAN 1, VLAN 40, VLAN 90, VLAN 91 ALL
`T360-DENY-Mgmt-Rest` ⛔ deny VLAN 90 → VLAN 1, VLAN 20, VLAN 40, VLAN 91 ALL
`T370-DENY-HyperV-Rest` ⛔ deny VLAN 80 → VLAN 1, VLAN 20, VLAN 90, VLAN 91 ALL
`T380-DENY-Default-Rest` ⛔ deny VLAN 1 → VLAN 20, VLAN 40, VLAN 90, VLAN 91 ALL
Internet traffic is never filtered; rules Torrent did not create are untouched. During the rebuild there are a few seconds with no Torrent rules (fail-open).

## Recovery plan

Automatic: health checks right after applying (controller, Pi-hole DNS, internet, SSH node01, Grafana/VLAN Flow); any failure, ❌, or no ✅ on the 'Keep these ACLs?' message within 15 min rebuilds the previous phase (3). Manual: `@torrent acl off` disables every Torrent rule at once. Last resort: Omada → Settings → Network Security → ACL → Gateway ACL, disable the `T###-` rules. ALLOW-Admin-Infra stays first so this PC keeps controller access.

## Risk

A missing permit could block a service between VLANs until rolled back; return traffic depends on the gateway's stateful handling, which the owner verifies from VLAN 10 during the confirmation window.

## Closure

**Failed** — closed by Ranni on 2026-10-15 22:47.

```
Traceback (most recent call last):
  File "/home/hermes-admin/olympus/netops.py", line 646, in <module>
    main()
  File "/home/hermes-admin/olympus/netops.py", line 626, in main
    acl_apply(int(a.arg2), a.proposal)
  File "/home/hermes-admin/olympus/netops.py", line 396, in acl_apply
    raise PermissionError("Rule set changed after approval; plan again")
PermissionError: Rule set changed after approval; plan again

```

Register: [RFC Register](RFC%20Register.md)
