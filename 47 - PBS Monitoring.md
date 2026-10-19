# PBS Monitoring

> [!NOTE] 2026-10-19
> GC schedules added (`daily` 04:30 daily, `main` Sat 06:00). Check `proxmox-backup-manager garbage-collection list` for last-run state.

> Prometheus monitoring for Proxmox Backup Server with Grafana dashboards and Glance integration.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [18 - Observability Stack](18%20-%20Observability%20Stack.md) | [03 - Storage Architecture](03%20-%20Storage%20Architecture.md)

---

## Overview

PBS monitoring provides visibility into:
- Backup job status and snapshot counts
- Datastore storage usage (daily/main)
- PBS host resource utilization

## Architecture

```
PBS (192.168.20.50)
        ↓ API (port 8007)
PBS Exporter (192.168.40.13:9101)
        ↓ Scrape
Prometheus (192.168.40.13:9090)
        ↓ Query
Grafana Dashboard + Glance Backup Page
```

## Components

### PBS Server

| Property | Value |
|----------|-------|
| LXC | 100 on node03 |
| IP | 192.168.20.50 |
| Web UI | https://192.168.20.50:8007 |
| Version | 3.4 |

### Datastores

| Datastore | Storage | Capacity | Purpose |
|-----------|---------|----------|---------|
| `daily` | Kingston NVMe | 1TB | Fast daily backups |
| `main` | Seagate HDD | 4TB | Weekly/monthly archives |

### PBS Exporter

| Property | Value |
|----------|-------|
| Container | pbs-exporter |
| Image | ghcr.io/natrontech/pbs-exporter:latest |
| Host | docker-vm-core-utilities01 |
| Port | 9101 |
| Config | /opt/pbs-exporter/docker-compose.yml |

## Configuration

### Docker Compose

```yaml
services:
  pbs-exporter:
    image: ghcr.io/natrontech/pbs-exporter:latest
    container_name: pbs-exporter
    environment:
      PBS_ENDPOINT: "https://192.168.20.50:8007"
      PBS_USERNAME: "backup@pbs"
      PBS_API_TOKEN_NAME: "pve"
      PBS_API_TOKEN: "<secret>"
      PBS_INSECURE: "true"
    ports:
      - "9101:10019"
```

### Prometheus Job

```yaml
- job_name: 'pbs'
  static_configs:
    - targets: ['192.168.40.13:9101']
      labels:
        instance: 'pbs-lxc100'
  scrape_interval: 60s
```

## Key Metrics

| Metric | Description |
|--------|-------------|
| `pbs_up` | Connection status (1=connected) |
| `pbs_size{datastore}` | Total size in bytes |
| `pbs_used{datastore}` | Used bytes |
| `pbs_available{datastore}` | Available bytes |
| `pbs_snapshot_count{datastore}` | Backup count |
| `pbs_host_cpu_usage` | CPU usage (0-1) |
| `pbs_host_memory_*` | Memory metrics |
| `pbs_host_load*` | Load averages |

## Access

| Resource | URL |
|----------|-----|
| PBS Web UI | https://192.168.20.50:8007 |
| Grafana Dashboard | https://grafana.hrmsmrflrii.xyz/d/pbs-backup-status |
| Glance Backup Page | https://glance.hrmsmrflrii.xyz (Backup tab) |
| Metrics Endpoint | http://192.168.40.13:9101/metrics |

## Useful Queries

```promql
# Storage usage percentage
pbs_used{datastore="daily"} / pbs_size{datastore="daily"} * 100

# Total backup count
sum(pbs_snapshot_count)

# Memory usage percentage
pbs_host_memory_used / pbs_host_memory_total * 100
```

## Troubleshooting

### pbs_up = 0

1. Check exporter logs: `docker logs pbs-exporter`
2. Verify API access works with curl
3. Check network connectivity to PBS

### No Data in Dashboard

1. Verify Prometheus target is up
2. Check time range in Grafana
3. Test metrics endpoint directly

### Glance Backup tab shows stale/wrong dates despite backups actually succeeding (Fixed January 14, 2026; found again and root-caused September 14, 2026)

This is a bug in `nas-backup-status-api` (port 9102), not in PBS or the actual backup jobs. Before assuming backups failed, check `proxmox-backup-manager task list` on PBS and `vzdump` entries in `/var/log/pve/tasks/index` on the PVE node — those are ground truth.

**Root cause**: `get_backup_job_status()` in `app.py` found the "latest" snapshot via `find {path} -maxdepth 4 -type d -name '20*T*' | sort -r | head -1`. This sorts by the **full path string** (`/backup-ssd/vm/{vmid}/{timestamp}`), not by timestamp — so a VMID that sorts lexicographically last (e.g. an old decommissioned VM like `311`, which beats any current `1xx`/`2xx` VMID as a string) permanently wins, even if its snapshots are months old and everything else backed up fine last night. Fixed by sorting on `-printf '%f\t%p\n'` (timestamp basename first) then `cut -f2` back to the path — see [Homelab Changelog](Homelab%20Changelog.md) 2026-09-14 entry for the full patch and an unrelated 8-day node03/PBS outage found in the same investigation.

**Quick diagnosis**: `curl http://192.168.40.13:9102/status | jq .` and eyeball whether `job_status.daily/main.last_backup` looks implausibly old. `curl http://192.168.40.13:9102/refresh` forces an immediate re-fetch.

### Backup tab shows "unknown" status (investigated 2026-09-14, not reproduced)

Distinct from the stale-dates bug above — this is the widget showing an unrecognized/blank status rather than a wrong-but-plausible one. Investigated same-day as the fix above; every live check came back healthy (`/status` returning correct `success` values end-to-end, template matching the JSON shape, both monitor targets returning `200`). Restarted the `glance` container as a precaution but couldn't confirm a root cause since it wasn't reproducible at investigation time. See [Homelab Changelog](Homelab%20Changelog.md) 2026-09-14 (second entry) for the full diagnostic trail. If it recurs, check `docker logs -f glance` live while it's happening rather than after the fact — point-in-time checks after the report came back clean every time.

**Reaching VLAN 40 hosts for diagnosis**: if diagnosing from a client on VLAN 10 without the Kratos ACL bypass, `ansible-controller01` (192.168.20.30) works as an SSH jump host (`ssh -J ansible-controller01 docker-vm-core-utilities01`) — it has a working route to both VLAN 20 and VLAN 40.

### Backup Jobs Failing with "job errors" (Added January 14, 2026)

Check recent backup task logs on the Proxmox node:

```bash
# View recent backup tasks
cat /var/log/pve/tasks/index | grep vzdump | tail -20

# Read specific task log (get UPID from above)
cat '/var/log/pve/tasks/0/UPID:node01:...'
```

**Common causes:**

#### 1. CT/VM Locked with Backup Lock

**Error**: `CT is locked (backup)`

**Cause**: Previous backup failed and left a stale lock

**Fix**:
```bash
# For containers
pct unlock <VMID>

# For VMs
qm unlock <VMID>
```

#### 2. Permission Error on Prune

**Error**: `permission check failed - missing Datastore.Modify|Datastore.Prune on /datastore/daily`

**Cause**: The API token `backup@pbs!pve` doesn't have prune permissions. This happens when the parent user `backup@pbs` lacks `DatastoreAdmin` role on the datastore.

**Fix** (on PBS server 192.168.20.50):
```bash
# Add DatastoreAdmin to parent user
proxmox-backup-manager acl update /datastore/daily DatastoreAdmin --auth-id backup@pbs

# Verify permissions
proxmox-backup-manager user permissions backup@pbs!pve
```

**Expected output** should include:
```
Path: /datastore/daily
- Datastore.Backup (*)
- Datastore.Modify (*)
- Datastore.Prune (*)
```

#### 3. PBS Connection Failed

**Error**: `connection refused` or `timeout`

**Fix**:
```bash
# Check PBS is running
ssh root@192.168.20.50 "systemctl status proxmox-backup-proxy"

# Test connectivity from Proxmox node
curl -k https://192.168.20.50:8007/api2/json/version
```

### Verify Backup Job Configuration

```bash
# List all backup jobs
pvesh get /cluster/backup --output-format json-pretty

# Check storage configuration
pvesm status | grep pbs
```

## Files

| Location | Purpose |
|----------|---------|
| /opt/pbs-exporter/docker-compose.yml | Exporter config |
| /opt/monitoring/prometheus/prometheus.yml | Scrape config |
| dashboards/pbs-backup-status.json | Dashboard JSON |

## Administration (Updated January 12, 2026)

### Root Password

| Field | Value |
|-------|-------|
| Username | `root` (select "Linux PAM" realm) |
| Password | `<redacted>` |

> **Note**: Enter `root` in username field (not `root@pam`). The realm dropdown adds the suffix.

### ACL Permissions (Updated January 14, 2026)

| Principal | Path | Role |
|-----------|------|------|
| `backup@pbs` | `/` | Audit |
| `backup@pbs` | `/datastore/main` | DatastoreAdmin, DatastoreBackup |
| `backup@pbs` | `/datastore/daily` | DatastoreAdmin, DatastoreBackup |
| `backup@pbs!pve` | `/` | Audit |
| `backup@pbs!pve` | `/datastore/main` | DatastoreAdmin |
| `backup@pbs!pve` | `/datastore/daily` | DatastoreAdmin, DatastoreBackup |

> [!important] API Token Permission Inheritance
> PBS API tokens can only inherit permissions that their parent user has. If the token `backup@pbs!pve` needs `DatastoreAdmin` on a datastore, the parent user `backup@pbs` must ALSO have `DatastoreAdmin` on that datastore.

### Subscription Nag Removal

Edit `/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js` and flip the condition from `!== 'active'` to `=== 'active'`.

## Drive Health Monitoring

SMART health monitoring for PBS storage drives via custom API on node03.

| Property | Value |
|----------|-------|
| Endpoint | http://192.168.20.22:9101/health |
| Service | `smart-health-api.service` on node03 |
| Drives | Seagate 4TB HDD (main), Kingston 1TB NVMe (daily) |

The Drive Health Status widget is displayed on the Glance Backup page.

## NAS Backup Status API (Updated January 15, 2026)

Custom API that monitors PBS-to-NAS backup sync, job durations, and lists backups stored on Synology NAS with VM/CT names.

| Property | Value |
|----------|-------|
| Host | docker-vm-core-utilities01 (192.168.40.13) |
| Port | 9102 |
| Container | `nas-backup-status-api` |
| Config | `/opt/nas-backup-status-api/` |

### Endpoints

| Endpoint | Description |
|----------|-------------|
| `/status` | Sync status, last sync time, datastore sizes, **job durations** |
| `/backups` | List of all VMs/CTs backed up on NAS **with names** |
| `/job-status` | Just the job status portion (daily, main) |
| `/health` | Health check with cache status |
| `/refresh` | Force cache refresh |

### Response Format

#### `/status` Response
```json
{
  "status": "success",
  "last_sync": "2026-01-15 02:31:27",
  "main_size": "907G",
  "daily_size": "292G",
  "nas_sync_duration": "28m 51s",
  "nas_target": "192.168.20.32:/volume2/ProxmoxData/pbs-offsite",
  "schedule": "Daily at 2:00 AM",
  "job_status": {
    "daily": {
      "last_backup": "2026-01-14 18:33",
      "status": "success",
      "count": 90,
      "duration": "3h 39m"
    },
    "main": {
      "last_backup": "2026-01-14 17:48",
      "status": "success",
      "count": 57,
      "duration": "4h 49m"
    }
  }
}
```

#### `/backups` Response
```json
{
  "backups": [
    {"vmid": "100", "name": "pbs-server", "type": "VM", "datastore": "main", "last_backup": "2026-01-14 17:48"},
    {"vmid": "101", "name": "docker-lxc-glance", "type": "CT", "datastore": "daily", "last_backup": "2026-01-14 18:33"}
  ],
  "total_count": 23,
  "vm_count": 10,
  "ct_count": 13,
  "cached": true
}
```

### Duration Calculation

| Duration Field | Source |
|----------------|--------|
| `nas_sync_duration` | Parsed from `/var/log/pbs-nas-backup.log` start/end timestamps |
| `job_status.daily.duration` | Time from first to last backup on most recent backup day |
| `job_status.main.duration` | Time from first to last backup on most recent backup day |

### VM Name Lookup

The API includes a `VM_NAMES` dictionary that maps VMID to human-readable names:
- Falls back to `{type}-{vmid}` if VMID not in lookup table
- Names are used in Glance dashboard display

### Caching

| Setting | Value |
|---------|-------|
| Cache TTL | 5 minutes |
| Background Refresh | Every 4 minutes |
| Thread | Daemon thread with background cache warming |

### Test Commands

```bash
# Full status with durations
curl http://192.168.40.13:9102/status | jq .

# Backup list with VM names
curl http://192.168.40.13:9102/backups | jq .

# Just job status
curl http://192.168.40.13:9102/job-status | jq .

# Force cache refresh
curl http://192.168.40.13:9102/refresh

# Health check
curl http://192.168.40.13:9102/health
```

### Glance Widgets

The Backup page uses this API in multiple widgets:
- **Backup Jobs Overview** - Shows daily, main, and NAS sync status with durations
- **VM & Container Backup Status** - Lists all protected VMs/CTs with names and last backup times

### Deployment

```bash
ansible-playbook glance/deploy-nas-backup-status-api.yml
```

### Files

| File | Purpose |
|------|---------|
| `ansible/playbooks/glance/files/nas-backup-api-app.py` | Python API source |
| `ansible/playbooks/glance/deploy-nas-backup-status-api.yml` | Ansible deployment playbook |
| `/opt/nas-backup-status-api/app.py` | Deployed API on server |

## Credentials

See 11 - Credentials for PBS API token details.
