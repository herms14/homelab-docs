---
tags:
  - todo
  - homelab
  - infrastructure
  - maintenance
created: 2026-05-20
updated: 2026-09-15
status: in-progress
---

# TODO: Homelab Improvements — May 2026

> Improvement backlog identified after node01 NVMe failure and recovery on 2026-05-20.

[← Back to Index](00%20-%20Homelab%20Index.md) | [📋 Changelog](Homelab%20Changelog.md)

---

## 🔴 High Priority

### 1. Fix Proxmox Backup Server (PBS)
- **Status:** ✅ Resolved 2026-05-23 — CT100 was stopped, started via `pct start 100`; web UI verified 200 OK
- **Reference:** [39 - PBS Deployment Tutorial](39%20-%20PBS%20Deployment%20Tutorial.md) | [40 - PBS Disaster Recovery](40%20-%20PBS%20Disaster%20Recovery.md)
- Automated backup schedule (last remaining sub-task here) is still open — see item #10 below.

---

### 2. Fix NFS Soft Mounts on node02 and node03
- **Status:** ✅ Resolved (2026-05-23 — done as part of NAS NIC upgrade)
- **Fixed on:** node01 (2026-05-20), node02 + node03 (2026-05-23)
- All nodes now use `soft,timeo=30,retrans=3,noatime,nfsvers=3,_netdev` and point to 192.168.20.32

---

### 3. Immich: Migrate Postgres pgvecto-rs → VectorChord
- **Status:** ✅ Resolved 2026-08-26 — migrated to `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`, Immich upgraded 2.7.5 → 3.1.0 in the same pass
- **VM:** immich-vm01 (192.168.40.22)
- **Reference:** [42 - Immich Backup and Restore](42%20-%20Immich%20Backup%20and%20Restore.md) | [57 - Immich VectorChord Migration and v3 Upgrade](57%20-%20Immich%20VectorChord%20Migration%20and%20v3%20Upgrade.md) (full record of what was done)

---

## 🟡 Medium Priority

### 4. Investigate node03 Cluster Membership (NR Status)
- **Status:** ⚠️ Shows NR (Not Reachable) in pvecm
- **Issue:** `pvecm status` on node02 shows node03 with `NR` flag
- **Impact:** Corosync ring may be degraded; cluster health uncertain

**Tasks:**
- [ ] SSH into node03: check corosync status: `systemctl status corosync`
- [ ] Run `pvecm status` from node03 directly
- [ ] Check corosync logs: `journalctl -u corosync --since "1 hour ago"`
- [ ] If ring is broken, rejoin corosync ring

---

### 5. Install node_exporter on node03
- **Status:** ⚠️ Not installed
- **Issue:** node03 metrics missing from Grafana Proxmox Cluster Health dashboard
- **Installed on:** node01 (2026-05-20), node02 (existing)

**Tasks:**
- [ ] Download node_exporter v1.7.0 to node03
- [ ] Install to `/usr/local/bin/node_exporter`
- [ ] Create systemd service (same flags as node01/node02)
- [ ] Enable and start: `systemctl enable --now node_exporter`
- [ ] Verify Prometheus scrapes `192.168.20.22:9100`

**Install commands (same as node01):**
```bash
cd /tmp
curl -sL https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz -o node_exporter.tar.gz
tar xzf node_exporter.tar.gz
cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/
chmod +x /usr/local/bin/node_exporter
```

---

### 6. Fix Omada Controller Prometheus Exporter
- **Status:** ✅ Resolved — wrong host was configured (.103 was a switch, not the controller); correct host is 192.168.0.101:443. Exporter healthy, metrics flowing.
- **Reference:** [41 - Omada ACL Implementation Guide](41%20-%20Omada%20ACL%20Implementation%20Guide.md)

---

### 7. Fix docker-stats-media Exporter
- **Status:** ❌ Down (Prometheus target = 0)
- **Issue:** `docker-stats-media docker-media` scrape target failing
- **Impact:** Container metrics for media stack missing from Grafana
- **Host:** docker-lxc-media (192.168.40.11)

**Tasks:**
- [ ] SSH into docker-lxc-media: `ssh hermes-admin@192.168.40.11`
- [ ] Check docker stats exporter container status
- [ ] Restart or redeploy the exporter

---

## 🟢 Nice to Have

### 8. Update Docker Images Inside Containers

---

### 9. OPNsense Migration — DNS Records
- **Issue:** All 45 `*.hrmsmrflrii.xyz` custom DNS records need to move from Pi-hole to OPNsense Unbound
- **Blocker:** OPNsense hardware is racked, configured (bare-metal, single-arm LAN at `192.168.90.60`), and healthy on its own subnet, but still not reachable from other VLANs (Kratos) as of 2026-09-15 despite gateway/route/firewall-rule fixes applied through the GUI. Needs one more debugging pass before DNS config can start.

**Tasks:**
- [x] OPNsense available and accessible — from same-subnet hosts only so far; cross-VLAN access still broken
- [ ] OPNsense reachable from all VLANs (in progress)
- [ ] Add all host overrides to OPNsense Unbound (Services → Unbound DNS → Host Overrides)
- [ ] Update DHCP server to hand out OPNsense IP as DNS instead of Pi-hole (192.168.90.53)
- [ ] Verify all `*.hrmsmrflrii.xyz` domains resolve from client
- [ ] Optionally keep Pi-hole for ad-blocking only (upstream DNS → OPNsense)

---

### 10. Schedule Regular Automated Backups
- **Status:** ⏳ Pending (blocked by item #1 — PBS must be fixed first)
- **Issue:** No automated backup job configured; last backups Feb 16, 2026

**Tasks:**
- [ ] Fix PBS first (item #1)
- [ ] Create backup job in Proxmox: Datacenter → Backup → Add
  - All nodes, all containers/VMs
  - Schedule: Weekly (Sunday 02:00)
  - Storage: PBS `main` datastore
  - Retention: Keep last 4 weekly backups
- [ ] Verify backup completes successfully
- [ ] Set up Discord notification on backup completion/failure

---

## Completed (2026-05-20)

- ✅ node01 NVMe diagnosed as dead (hardware controller failure confirmed)
- ✅ node01 fresh Proxmox VE install — rejoined MorpheusCluster
- ✅ node01 NFS fstab fixed to soft mounts (prevents future freezes)
- ✅ node01 vmbr0 VLAN-aware bridge configured (`bridge-vlan-aware yes`, `bridge-vids 2-4094`)
- ✅ node01 all 6 LXC containers restored from Feb 16 backups (CT200-203, CT205-206)
- ✅ node01 packages upgraded to latest (Proxmox VE 9.1.19, kernel 6.17.2-1-pve)
- ✅ node01 enterprise repos replaced with no-subscription repos
- ✅ node01 node_exporter v1.7.0 installed — Prometheus scraping restored
- ✅ All 10 cluster containers updated (OS packages)
- ✅ Grafana anonymous access fixed (`GF_AUTH_ANONYMOUS_ORG_NAME=Main Org.`)

---

*Created: 2026-05-20 | Managed by Claude Code Homelab Agent*
