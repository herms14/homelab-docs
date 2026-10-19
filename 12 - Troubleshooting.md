---
updated: 2026-10-16
---

# Troubleshooting

> **Internal Documentation** - Diagnostic commands and resolution steps organized by category.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) | [01 - Network Architecture](01%20-%20Network%20Architecture.md)

---

## Table of Contents

- [Roundtable Portal, Teaching and Discord](#roundtable-portal-teaching-and-discord)
- [Proxmox Cluster Issues](#proxmox-cluster-issues)
- [Kubernetes Issues](#kubernetes-issues)
- [Authentication Issues](#authentication-issues)
- [Container & Docker Issues](#container--docker-issues)
- [Service-Specific Issues](#service-specific-issues)
- [Network Issues](#network-issues)
- [Common Issues](#common-issues)
- [Diagnostic Commands](#diagnostic-commands)

---

## Roundtable Portal, Teaching and Discord

Use the current Roundtable guide and the signed-in
https://roundtable.hrmsmrflrii.xyz portal. Backend: codex-agent-lxc (.40.16), `olympus-portal.service`
and `boss.service`; the Sentinel Docker cogs on .40.13 are a separate backend.

| Symptom | First check / safe recovery |
|---|---|
| New artwork looks old | Reload/reopen the portal. The seven new selected URLs end in `-matched.jpg` / `-matched-full.png`; old selected artwork is intentionally unchanged. Do not regenerate paid images or restart the bot for a browser cache |
| Voice/teaching unavailable over raw HTTP/IP | Use the signed-in HTTPS URL; HTTP is view-only. Do not disable browser security, SSO or identity checks |
| Teaching pop-out missing or closed | Allow portal pop-ups and use Reopen terminal. Keep the parent portal tab open; no Electron/native helper install is needed |
| Preview appears but no real output | "Command preview — not yet executed" is not execution. Run the current step; inspect any displayed SSH/host/command error. Each command is independent, not a persistent shell |
| Write awaits approval | Browser Run requests the exact owner-only Discord approval. It does not execute before approval; inspect queue/verification/rollback rather than resubmitting or bypassing it |
| Pause during a command | It can disconnect a bounded read without guaranteeing remote termination. It never force-kills an approved configuration change; inspect final state before advancing |
| Corporate Mac reports NXDOMAIN | Homelab-side Pi-hole portal/auth records and proxy were verified healthy. Endpoint resolver policy/interception remains unresolved and **owner-paused**; no corporate-control/DNS bypass or unrequested network change |

Latest acceptance: 29 reporting agents/four leads, seven matched artwork sets/all 29 Discord icons,
are not established by the browser/text checks. No general shell or local Terminal/PowerShell control
was deployed; use existing reviewed build/patch workflows for work outside the teaching grammar.

---

## Proxmox Cluster Issues

### Corosync SIGSEGV Crash

**Resolved**: December 2025

**Symptoms**:
- `corosync.service` fails with `status=11/SEGV`
- Logs stop at: `Initializing transport (Kronosnet)`
- Node cannot join cluster

**Root Cause**: Broken NSS crypto stack (`libnss3`).

**Resolution**:
```bash
apt install --reinstall -y \
  libnss3 libnss3-tools \
  libknet1t64 libnozzle1t64 \
  corosync libcorosync-common4
```

**Verification**:
```bash
systemctl start corosync
pvecm status
journalctl -u corosync | grep crypto_nss
```

---

### Node Showing Question Mark

**Resolved**: December 2025

**Symptoms**:
- Question mark icon in Proxmox UI
- "NR" (Not Ready) status

**Diagnosis**:
```bash
ping 192.168.20.22
ssh root@192.168.20.22 "pvecm status"
```

**Resolution**:
1. Power on node if shutdown
2. Restart cluster services:
   ```bash
   systemctl restart pve-cluster && systemctl restart corosync
   ```

---

### Cloud-init VM Boot Failure

**Resolved**: December 2025

**Symptoms**:
- Console stops at: `Btrfs loaded, zoned=yes, fsverity=yes`
- VM unreachable via SSH/ping

**Root Cause**: UEFI/BIOS boot mode mismatch.

**Resolution**: Updated `modules/linux-vm/main.tf`:
```hcl
bios    = "ovmf"
machine = "q35"

efidisk {
  storage           = var.storage
  efitype           = "4m"
  pre_enrolled_keys = true
}

scsihw = "virtio-scsi-single"
```

---

### Backup Jobs Failing - VM Locked (Stale Backup Lock)

**Resolved**: January 2026

**Symptoms**:
- Backup jobs report "job errors" in email notifications
- Proxmox cluster tasks show `status: job errors` for vzdump
- Specific VM backup fails with: `ERROR: Backup of VM XXX failed - VM is locked (backup)`
- VM status shows "stopped" or "running" but has a stale lock

**Root Cause**: VMs get locked during backup operations to prevent modifications. If a backup is interrupted (node reboot, network timeout, PBS connection lost), the lock remains orphaned.

> [!warning] Common Cause
> Node reboots during scheduled backups are the most common cause of orphaned backup locks.

**Diagnosis**:
```bash
# Check cluster tasks for backup errors
ssh root@192.168.20.20 "pvesh get /cluster/tasks --output-format json | jq '.[] | select(.type==\"vzdump\" and .status!=\"OK\") | {node, status}'"

# Find all locked VMs across cluster
ssh root@192.168.20.20 "grep -l 'lock:' /etc/pve/nodes/*/qemu-server/*.conf /etc/pve/nodes/*/lxc/*.conf 2>/dev/null"

# Check specific VM lock
ssh root@<node> "qm status <vmid>"
ssh root@<node> "cat /etc/pve/nodes/<node>/qemu-server/<vmid>.conf | grep lock"
```

**Fix**:
```bash
# Unlock a VM
ssh root@<node> "qm unlock <vmid>"

# Unlock an LXC container
ssh root@<node> "pct unlock <ctid>"

# Verify lock is removed
ssh root@<node> "cat /etc/pve/nodes/<node>/qemu-server/<vmid>.conf | grep lock"
# Should return nothing (no lock)
```

**Quick Reference**:
| Command | Purpose |
|---------|---------|
| `qm unlock <vmid>` | Unlock a VM |
| `pct unlock <ctid>` | Unlock an LXC container |
| `qm status <vmid>` | Check VM status (shows if locked) |
| `pvesh get /cluster/tasks` | View recent cluster tasks |

**Prevention**:
- Monitor backup job completion emails
- After node reboots, check for orphaned locks:
  ```bash
  grep -l 'lock:' /etc/pve/nodes/*/qemu-server/*.conf /etc/pve/nodes/*/lxc/*.conf 2>/dev/null
  ```

---

### PBS-to-NAS Sync (Offsite Backup)

Daily sync of PBS backups to Synology NAS for redundant storage.

> [!info] Schedule
> Runs daily at **2:00 AM** via cron on PBS server (192.168.20.50)

**Configuration**:
| Component | Value |
|-----------|-------|
| Script | `/usr/local/bin/pbs-backup-to-nas.sh` |
| Log File | `/var/log/pbs-nas-backup.log` |
| NAS Mount | `/mnt/nas-backup` → `192.168.20.31:/volume2/ProxmoxData` |

**What Gets Synced**:
```
PBS Server                          →    Synology NAS
├── /backup (main)                  →    pbs-offsite/main/
└── /backup-ssd (daily)             →    pbs-offsite/daily/
```

**Manually Run Sync**:
```bash
ssh root@192.168.20.50 "/usr/local/bin/pbs-backup-to-nas.sh"
```

**Check Sync Status**:
```bash
# View logs
ssh root@192.168.20.50 "tail -50 /var/log/pbs-nas-backup.log"

# Check sizes
ssh root@192.168.20.50 "du -sh /mnt/nas-backup/pbs-offsite/*"
```

**Troubleshooting**:
- **"Backup already running"**: Remove stale lock: `rm /var/run/pbs-nas-backup.lock`
- **"Failed to mount NAS"**: Check NFS: `showmount -e 192.168.20.31`

---

## Kubernetes Issues

### kubectl Connection Refused

**Resolved**: December 20, 2025

**Symptoms**: On non-primary controllers (controller02, controller03):
```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

**Root Cause**: Kubeconfig not set up on secondary controller nodes after `kubeadm join`.

**Fix**:
```bash
ssh hermes-admin@192.168.20.32 "cat ~/.kube/config" | ssh hermes-admin@192.168.20.33 "mkdir -p ~/.kube && cat > ~/.kube/config && chmod 600 ~/.kube/config"
ssh hermes-admin@192.168.20.32 "cat ~/.kube/config" | ssh hermes-admin@192.168.20.34 "mkdir -p ~/.kube && cat > ~/.kube/config && chmod 600 ~/.kube/config"
```

**Verification**:
```bash
for ip in 192.168.20.32 192.168.20.33 192.168.20.34; do
  echo "=== $ip ==="
  ssh hermes-admin@$ip "kubectl get nodes --no-headers | head -3"
done
```

**Prevention**: Add kubeconfig distribution to Kubernetes Ansible playbook.

---

## Authentication Issues

### Authentik ForwardAuth "Not Found" Error

**Resolved**: December 21, 2025

**Symptoms**: When accessing services protected by Authentik ForwardAuth (Grafana, Prometheus, Jaeger), users receive a "not found" error instead of being redirected to login.

**Root Cause**: The Authentik **Embedded Outpost had no providers assigned**. Proxy providers and applications were created, but never bound to the outpost that handles ForwardAuth requests from Traefik.

**Diagnosis**:
```bash
ssh hermes-admin@192.168.40.21 "sudo docker exec authentik-server ak shell -c \"
from authentik.outposts.models import Outpost
outpost = Outpost.objects.get(name='authentik Embedded Outpost')
print(f'Providers: {list(outpost.providers.values_list(\"name\", flat=True))}')
\""
# Empty list = problem
```

**Fix**:
```bash
ssh hermes-admin@192.168.40.21 "sudo docker exec authentik-server ak shell -c \"
from authentik.providers.proxy.models import ProxyProvider
from authentik.outposts.models import Outpost

providers = list(ProxyProvider.objects.all())
outpost = Outpost.objects.get(name='authentik Embedded Outpost')
for p in providers:
    outpost.providers.add(p)
outpost.save()
print(f'Added {len(providers)} providers to outpost')
\""
```

**Verification**:
```bash
# Should return 302 (redirect to login)
curl -s -k -o /dev/null -w "%{http_code}" https://grafana.hrmsmrflrii.xyz
```

**Prevention**:
1. Always assign new providers to the Embedded Outpost in Authentik Admin UI
2. Include outpost assignment in blueprints
3. Verify outpost has providers assigned before testing

---

### Authentik "Permission Denied - Internal Users Only"

**Resolved**: December 24, 2025

**Symptoms**: User logs in via Google/GitHub OAuth and sees "Permission denied - Interface can only be accessed by internal users"

**Root Cause**: OAuth sources create users with `type=external` by default. The Authentik Admin Interface (`/if/admin/`) is restricted to internal users by design.

> [!info] User Type Behavior
> | User Type | App Access | Admin UI Access |
> |-----------|------------|-----------------|
> | `internal` | ✅ Yes | ✅ Yes |
> | `external` | ✅ Yes | ❌ No |
>
> **External users CAN access regular apps** (Grafana, Jellyfin, etc.) - this restriction only applies to the Authentik Admin Interface.

**This is expected behavior** - most OAuth users should remain `external`. Only change to `internal` if admin access is specifically needed.

**Diagnosis** (check user types):
```bash
ssh hermes-admin@192.168.40.21 "sudo docker exec authentik-server ak shell -c \"
from authentik.core.models import User
for u in User.objects.all():
    print(f'{u.username}: type={u.type}')
\""
```

**Fix** (only if admin access is required):
```bash
ssh hermes-admin@192.168.40.21 "sudo docker exec authentik-server ak shell -c \"
from authentik.core.models import User
user = User.objects.get(username='USERNAME_HERE')
user.type = 'internal'
user.save()
print(f'Changed {user.username} to internal')
\""
```

---

### GitLab SSO "Failed to open tcp connection" (DNS Resolution)

**Resolved**: December 24, 2025

**Symptoms**: Clicking "Authentik" SSO button on GitLab login page shows:
```
Could not authenticate you from OpenIDConnect because "Failed to open tcp connection to auth.hrmsmrflrii.xyz:443 (getaddrinfo: name or service not known)".
```

**Root Cause**: GitLab VM had incorrect DNS configuration. Netplan was configured with DNS server `192.168.20.1` instead of `192.168.91.30` (OPNsense). The VM couldn't resolve internal domain names.

**Diagnosis**:
```bash
# Check DNS resolution from GitLab VM
ssh hermes-admin@192.168.40.23 "nslookup auth.hrmsmrflrii.xyz"
# Returns: NXDOMAIN = DNS misconfigured

# Check current DNS configuration
ssh hermes-admin@192.168.40.23 "resolvectl status | grep 'DNS Server'"
```

**Fix**:
```bash
# Update netplan DNS to correct server
ssh hermes-admin@192.168.40.23 "sudo sed -i 's/192.168.20.1/192.168.91.30/g' /etc/netplan/50-cloud-init.yaml"

# Apply changes
ssh hermes-admin@192.168.40.23 "sudo netplan apply"

# Verify DNS now works
ssh hermes-admin@192.168.40.23 "nslookup auth.hrmsmrflrii.xyz"
# Should return: 192.168.40.20 (Traefik IP)
```

**Verification**:
```bash
# Test from Docker container
ssh hermes-admin@192.168.40.23 "docker exec gitlab nslookup auth.hrmsmrflrii.xyz"
```

> [!tip] Prevention
> Ensure all VLAN 40 VMs use DNS `192.168.91.30` (OPNsense). Check Terraform `nameserver` variable in `main.tf` vm_groups.

---

### GitLab SSO "Invalid client" (Client Secret Mismatch)

**Resolved**: December 24, 2025

**Symptoms**: After fixing DNS, clicking "Authentik" SSO button shows:
```
Could not authenticate you from OpenIDConnect because "Invalid client :: client authentication failed (e.g., unknown client, no client authentication included, or unsupported authentication method)".
```

**Root Cause**: The client secret in GitLab's docker-compose.yml was truncated/incorrect:
- Wrong character: `<redacted>` (digit "1") instead of `<redacted>` (lowercase "L")
- Truncated: Missing the second half of the secret string

**Diagnosis**:
```bash
# Get correct secret from Authentik database
ssh hermes-admin@192.168.40.21 "docker exec authentik-postgres psql -U authentik -d authentik -c \"
SELECT p.name, o.client_id, o.client_secret
FROM authentik_core_provider p
JOIN authentik_providers_oauth2_oauth2provider o ON p.id = o.provider_ptr_id
WHERE p.name ILIKE '%gitlab%';\""

# Compare with GitLab config
ssh hermes-admin@192.168.40.23 "grep 'secret:' /opt/gitlab/docker-compose.yml"
```

**Fix**:
```bash
# Update GitLab docker-compose.yml with correct secret
ssh hermes-admin@192.168.40.23 "sudo nano /opt/gitlab/docker-compose.yml"
# Fix the 'secret:' value to match Authentik database

# Restart GitLab (takes 2-3 minutes to initialize)
ssh hermes-admin@192.168.40.23 "cd /opt/gitlab && sudo docker compose down && sudo docker compose up -d"
```

**Verification**:
1. Navigate to https://gitlab.hrmsmrflrii.xyz
2. Click "Authentik" button
3. Complete Authentik login
4. Should redirect back to GitLab logged in

> [!warning] Prevention
> When copying OAuth secrets:
> - Verify full string length matches
> - Watch for "1" (one) vs "l" (lowercase L) confusion
> - Test OIDC immediately after configuration

---

### Jellyfin SSO Button Not Showing on Login Page

**Resolved**: December 24, 2025

**Symptoms**: Jellyfin login page shows only username/password fields, no "Sign in with Authentik" button despite SSO-Auth plugin being installed.

**Root Cause**: SSO-Auth plugin does not automatically add a login button. It must be manually added via Branding settings.

**Fix**:
1. Go to Dashboard → General → Branding
2. In "Login disclaimer" field, add:
```html
<a href="/sso/OID/start/authentik" style="display: block; width: 100%; padding: 12px; margin-top: 10px; background-color: #00a4dc; color: white; text-align: center; text-decoration: none; border-radius: 4px; font-weight: bold;">Sign in with Authentik</a>
```
3. Click Save

**Verification**: Refresh login page - button should appear below password field.

---

### Jellyfin SSO invalid_grant Error

**Resolved**: December 24, 2025

**Symptoms**: After Authentik login, redirected back with:
```
Error logging in: Error redeeming code: invalid_grant
The provided authorization grant or refresh token is invalid...
```

**Root Cause**: Scheme mismatch. Jellyfin behind reverse proxy generates HTTP redirect URIs, but authorization was done with HTTPS.

**Fix**:
1. Go to Dashboard → Plugins → SSO-Auth → Settings
2. Select provider and click "Load Provider"
3. Set **Scheme Override** to `https`
4. Click "Save Provider"
5. Restart Jellyfin: `docker restart jellyfin`

**Verification**: Test SSO login - should complete successfully.

> [!tip] Prevention
> Always configure Scheme Override to `https` for services behind TLS-terminating reverse proxies.

---

### Jellyseerr OIDC Not Working with Latest Image

**Resolved**: December 24, 2025

**Symptoms**:
- OIDC environment variables configured but no SSO button appears
- API shows `openIdProviders: []` (empty array)
- No errors in logs, OIDC just silently not working

**Root Cause**: The `latest` Jellyseerr image does not include native OIDC support. OIDC is only available in the `preview-OIDC` branch image.

**Diagnosis**:
```bash
# Check current image
ssh hermes-admin@192.168.40.11 "docker inspect jellyseerr --format='{{.Config.Image}}'"

# Check API for OIDC providers
ssh hermes-admin@192.168.40.11 "curl -s http://localhost:5056/api/v1/settings/public | jq '.openIdProviders'"
```

**Resolution**:
```bash
# Update docker-compose.yml to use preview-OIDC image
ssh hermes-admin@192.168.40.11 "sudo sed -i 's|fallenbagel/jellyseerr:latest|fallenbagel/jellyseerr:preview-OIDC|' /opt/arr-stack/docker-compose.yml"

# Recreate container
ssh hermes-admin@192.168.40.11 "cd /opt/arr-stack && sudo docker compose up -d --force-recreate jellyseerr"
```

Then configure OIDC via UI: **Settings → Users → Configure OpenID Connect**

> [!warning] Important
> OIDC configuration via environment variables does NOT work. Must use the UI settings.

---

### Jellyseerr SSO Redirect URI Error

**Resolved**: December 24, 2025

**Symptoms**: After clicking SSO button and authenticating with Authentik, user is redirected back with error:
```
Redirect URI Error
The request fails due to a missing, invalid, or mismatching redirection URI
```

**Root Cause**: Two issues:
1. Jellyseerr uses a new callback format: `/login?provider=authentik&callback=true`
2. Behind reverse proxy, Jellyseerr generates `http://` URIs instead of `https://`

**Diagnosis**:
```bash
# Check Authentik logs for the actual redirect URI being sent
ssh hermes-admin@192.168.40.21 "sudo docker logs authentik-server --tail 50 2>&1 | grep -i redirect"
```

**Resolution**: Add both HTTP and HTTPS redirect URIs with regex matching in Authentik:

```bash
ssh hermes-admin@192.168.40.21 "sudo docker exec authentik-server ak shell -c \"
from authentik.providers.oauth2.models import OAuth2Provider
from authentik.providers.oauth2.constants import RedirectURIMatchingMode
from authentik.providers.oauth2.models import RedirectURI

provider = OAuth2Provider.objects.get(name='jellyseerr-oidc-provider')
new_uris = [
    RedirectURI(matching_mode=RedirectURIMatchingMode.REGEX, url=r'https://jellyseerr\.hrmsmrflrii\.xyz/login\?provider=authentik.*'),
    RedirectURI(matching_mode=RedirectURIMatchingMode.REGEX, url=r'http://jellyseerr\.hrmsmrflrii\.xyz/login\?provider=authentik.*'),
]
provider.redirect_uris = new_uris
provider.save()
\""
```

**Key Points**:
- Use `REGEX` matching mode for URLs with query parameters
- Include BOTH `http://` and `https://` versions
- Escape dots in regex: `\.` not `.`

**Verification**: Navigate to https://jellyseerr.hrmsmrflrii.xyz, click SSO button, complete login - should work without errors.

---

## Container & Docker Issues

### Watchtower TLS Handshake Error

**Resolved**: December 2025

**Symptoms**: `tls: first record does not look like a TLS handshake`

**Root Cause**: Using `generic://` instead of `generic+http://` in webhook URL.

**Fix**: Update `WATCHTOWER_NOTIFICATION_URL`:
```yaml
# Wrong
WATCHTOWER_NOTIFICATION_URL: "generic://192.168.40.13:5050/webhook"

# Correct
WATCHTOWER_NOTIFICATION_URL: "generic+http://192.168.40.13:5050/webhook"
```

Then restart: `cd /opt/watchtower && sudo docker compose restart`

---

### Update Manager SSH Key Not Accessible

**Resolved**: December 2025

**Symptoms**: Discord bot returns `❌ Update failed: Could not find compose directory`

**Root Cause**: SSH key not mounted in Update Manager container.

**Fix**:
```bash
scp ~/.ssh/homelab_ed25519 hermes-admin@192.168.40.13:/home/hermes-admin/.ssh/
ssh hermes-admin@192.168.40.13 "chmod 600 /home/hermes-admin/.ssh/homelab_ed25519"
ssh hermes-admin@192.168.40.13 "cd /opt/update-manager && sudo docker compose restart"
```

**Verification**:
```bash
ssh hermes-admin@192.168.40.13 "docker exec update-manager ssh -i /root/.ssh/homelab_ed25519 -o StrictHostKeyChecking=no hermes-admin@192.168.40.11 hostname"
```

---

### Docker Build Cache Issues

**Resolved**: December 2025

**Symptoms**: Bot behavior doesn't change after modifying code.

**Root Cause**: Docker caching old layers.

**Fix**: Force rebuild with no cache:
```bash
sudo docker compose down && sudo docker compose build --no-cache && sudo docker compose up -d
```

---

### Jellyfin Shows Fewer Movies Than Download Monitor

**Resolved**: December 24, 2025

**Symptoms**:
- Download Monitor shows several movies downloaded
- Jellyfin only shows 5 movies
- Movies in `/mnt/media/Completed/` but not in `/mnt/media/Movies/`
- Jellyfin logs: `Library folder /data/movies is inaccessible or empty`

**Root Causes (Multiple)**:
1. **Docker Volume Mount Failure**: Nested bind mounts over NFS didn't initialize
2. **Dual Root Folders in Radarr**: `/data/Movies` and `/movies` causing inconsistent imports
3. **Missing SABnzbd Remote Path Mapping**: Only Deluge had mapping configured
4. **Stuck Radarr Command**: `ProcessMonitoredDownloads` blocked other operations

**Diagnosis**:
```bash
# Check Jellyfin mount
docker exec jellyfin ls -la /data/movies
# Returns "No such file" = mount not active

# Check Radarr root folders
curl -s 'http://localhost:7878/api/v3/rootfolder' -H 'X-Api-Key: KEY' | jq '.[] | .path'
# Shows both paths = dual root folder issue
```

**Fix Steps**:
1. Recreate Jellyfin container:
   ```bash
   cd /opt/arr-stack && sudo docker compose up -d --force-recreate jellyfin
   ```

2. Add SABnzbd remote path mapping via Radarr API

3. Consolidate all movies to `/data/Movies` path using Python script

4. Delete legacy `/movies` root folder

5. Restart Radarr to clear stuck commands

**Prevention**:
- Use unified `/data` mount for all arr-stack services
- Configure remote path mappings for ALL download clients
- Use only ONE root folder per media type
- Avoid nested Docker bind mounts over NFS

> [!warning] Best Practice
> Use single parent mount in docker-compose:
> ```yaml
> volumes:
>   - /mnt/media:/data  # Single mount - good
> # NOT:
>   - /mnt/media/Movies:/data/movies  # Nested - problematic!
> ```

---

## Service-Specific Issues

### Immich VM pings but SSH/Immich refuse after a reboot (2026-10-16)

**Symptom**: immich-vm01 answers ping; ports 22/2283 refused; Proxmox shows `running` with 0% CPU and the guest agent down; the console stops at about 3.5s of boot.
**Cause**: a systemd ordering cycle from a bind mount into an NFS share (`/mnt/synology-photos`) without `_netdev`. Its NFS mounts start before networking; without `nofail` the boot ends in emergency mode, which cannot attach to the console.
**Diagnose without a shell** (read-only, from node03): `losetup -r -P` the raw disk, then `debugfs -R "cat /etc/fstab"` and `debugfs -R "rdump /var/log/journal/<id> /tmp/j"`, then `journalctl -D /tmp/j -b 0 -p warning` and look for `ordering cycle`.
**Fix**: every NFS line gets `_netdev,nofail`; a bind mount into NFS gets `_netdev,nofail,x-systemd.requires-mounts-for=<nfs mountpoint>`. See [21 - Application Configurations](21%20-%20Application%20Configurations.md).

### Sonarr/Radarr "All indexers are unavailable" or Lidarr can't reach Deluge (2026-10-16)

**Symptom**: Sonarr/Radarr health shows `IndexerStatusCheck: All indexers are unavailable due to failures`; Prowlarr logs `Unable to connect to indexer ... DNS/SSL issues`.
**Cause**: the containers on CT205 couldn't resolve names (phase 4 ACLs blocked VLAN 40 → Pi-hole DNS). After repeated failures the apps back off every indexer and stay in that state after DNS returns.
**Fix**: restore DNS, check with `docker exec prowlarr getent hosts apibay.org`, then run *Test All* on indexers in Prowlarr, Sonarr and Radarr (or `POST /api/v3/indexer/testall`), which clears the failure state.
**Lidarr "Failed to authenticate with Deluge"**: its Deluge client had an old password (Deluge logs `Login failed (ClientIP <lidarr container IP>)`). Set the current Deluge WebUI password in Lidarr → Settings → Download Clients.

### GitLab Unsupported Config Value (grafana)

**Resolved**: December 20, 2025

**Symptoms**: GitLab container restart loop with:
```
FATAL: Mixlib::Config::UnknownConfigOptionError: Reading unsupported config value grafana.
```

**Root Cause**: GitLab removed bundled Grafana support. The `grafana['enable'] = false` line is deprecated.

**Fix**: Remove `grafana['enable'] = false` from GITLAB_OMNIBUS_CONFIG in `/opt/gitlab/docker-compose.yml`:
```bash
cd /opt/gitlab && sudo docker compose down && sudo docker compose up -d
```

**Verification**:
```bash
docker ps --filter name=gitlab
docker exec gitlab gitlab-ctl status
```

**Prevention**: Review GitLab release notes for deprecated options before updates.

---

## Network Issues

### Omada devices disconnected and slow DNS after an ACL rollout (2026-10-16)

**Symptom**: Omada switches/APs show disconnected in the controller; services on VLAN 40 are slow to call internet APIs.
**Cause**: Gateway ACL phase 4 denied VLAN 90 devices → controller (UDP 29810, TCP 29814) and controller → VLAN 90, plus VLAN 40/IoT → Pi-hole DNS. The permits meant to allow these (`ALLOW-Omada-Adopt/Manage`, `ALLOW-DNS-PiHole`) did not match. VLAN 40 DHCP hands out 8.8.8.8 as secondary, so lookups waited for a Pi-hole timeout first: slow, not broken.
**Diagnose**: VLAN Flow → filter Denied, look for destination `192.168.0.101` ports 29810-29817 and destination `192.168.90.53:53`. The `DESC=` rule IDs change on every rebuild, so match rules by traffic pattern.
**Fix**: `@torrent acl off` (or disable the `T###-` rules in Omada → Network Security → ACL). Verify the live rule order before re-enabling.

### VLAN-Aware Bridge Missing

**Symptom**: `QEMU exited with code 1`

**Solution**: Configure `/etc/network/interfaces`:
```bash
auto vmbr0
iface vmbr0 inet static
    address 192.168.20.XX/24
    gateway 192.168.20.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

Then: `ifreload -a` or reboot.

---

### NFS Mount Failures

**Diagnosis**:
```bash
showmount -e 192.168.20.31
df -h | grep nfs
```

**Common Fixes**:
- Check NFS service on NAS
- Verify firewall (ports 111, 2049)
- For stale mounts: `umount -l /mnt/stale && mount -a`

---

## Common Issues

### Connection Refused Errors

**Symptom**: `dial tcp 192.168.20.21:8006: connectex: No connection`

**Solution**:
```bash
ssh root@192.168.20.21 "systemctl status pveproxy"
```

---

### Template Not Found (LXC)

**Symptom**: `template 'local:vztmpl/...' does not exist`

**Solution**:
```bash
ssh root@<node> "pveam update && pveam download local ubuntu-22.04-standard_22.04-1_amd64.tar.zst"
```

---

### Terraform State Lock

**Solution**:
1. Ensure no other terraform operations running
2. Force unlock: `terraform force-unlock <lock-id>`

---

## Diagnostic Commands

### Terraform
```bash
terraform state list
terraform state show <resource>
terraform refresh
terraform validate
```

### Proxmox
```bash
pvecm status
pvesh get /cluster/resources --type node
qm config <vmid>
systemctl status pve-cluster corosync pveproxy
journalctl -xeu corosync
```

### Kubernetes
```bash
kubectl get nodes
kubectl get pods -A
kubectl describe node <node>
kubectl logs -n <namespace> <pod>
```

### Ansible
```bash
ansible all -m ping
ansible <host> -m setup
```

### Network
```bash
ip -d link show vmbr0 | grep vlan_filtering
bridge link show
ip route show
```

### Docker
```bash
docker logs <container> --tail 50
docker exec <container> <command>
```

### Authentik
```bash
ssh hermes-admin@192.168.40.21 "sudo docker exec authentik-server ak shell -c \"
from authentik.outposts.models import Outpost
outpost = Outpost.objects.get(name='authentik Embedded Outpost')
print(f'Providers: {outpost.providers.count()}')
\""
```

---

## Related Documentation

- [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) - Cluster configuration
- [01 - Network Architecture](01%20-%20Network%20Architecture.md) - Network configuration
- [05 - Terraform Configuration](05%20-%20Terraform%20Configuration.md) - Deployment configuration
- [19 - Watchtower Updates](19%20-%20Watchtower%20Updates.md) - Container update issues
- [14 - Authentik Google SSO Setup](14%20-%20Authentik%20Google%20SSO%20Setup.md) - SSO configuration

### Homepage SRE SSH Access Fails: Missing Agent Public Key

**Symptoms**:
- `ssh hermes-admin@192.168.40.27` fails with `Permission denied (publickey,password)`.

**Cause**: The public key for `~/.ssh/sre_agent_ed25519` was absent from `/home/hermes-admin/.ssh/authorized_keys` in Homepage CT209. The October 5 provisioning omission is suspected but unproven; ownership and permissions were correct.

**Solution**:
1. Rechecked CT209 status, hostname, IP, agent public key/fingerprint, and SSH configuration.
2. Created and verified both backups with `cp -a` before conditionally appending the verified SRE key. The run record did not include the backup destination paths.
3. Tested a fresh SSH connection with connection sharing disabled and ran `sudo -n true`.
4. Confirmed both administrator keys remained unchanged, the SRE key was present, and `authorized_keys` retained mode `600` and ownership `1000:1000`. Verified the SSH journal fingerprint `<redacted>` and Homepage HTTPS returned HTTP 200.

**Prevention**: Manage the SRE authorized key across every inventory host and verify a fresh login after provisioning or rotation. Distinguish SSH authentication failures from network and application outages in monitoring.

Seen: 2026-10-11 (GitLab #27)
Seen again: 2026-10-11 (GitLab #34; proposal only, remediation and remote verification pending)

### Helios LXC AppArmor Blocks systemd Service Isolation

**Symptoms**:
- `logrotate.service` and `man-db.service` fail on `helios-lxc` (CT210).

**Cause**: CT210 is an unprivileged LXC with nesting disabled. Its generated AppArmor profile rejects systemd's proc mount at `/run/systemd/unit-root/proc`, preventing service namespace creation. The historical configuration change that introduced this incompatibility was not established. [Proxmox development note](https://lists.proxmox.com/pipermail/pve-devel/2025-October/076163.html)

**Solution**:
1. Inspected CT210 configuration, failed units, timers, APIs, and filesystem usage; copied the CT configuration to two backup paths and verified identical contents (the run record did not include their paths).
2. Enabled nesting with `ssh node01 'pct set 210 --features nesting=1'` and gracefully restarted using `ssh node01 'pct shutdown 210 --timeout 60 && pct start 210'` so Proxmox regenerated the AppArmor profile.
3. Validated logrotate with `logrotate --debug /etc/logrotate.conf`, then started `logrotate.service` and `man-db.service` normally and checked their status and journals.
4. Checked timers, host kernel logs, systemd-logind, filesystem usage, and API listeners; queried all three `/health` endpoints from node01 (curl was unavailable in the guest).
5. Both units completed successfully, all three APIs and health endpoints returned healthy results, and no failed units or new matching denials remained. Verification after the next scheduled maintenance runs and log rotation is pending.

**Prevention**: Include `features: nesting=1` in Helios LXC provisioning. After host or LXC updates, check systemd namespace failures, maintenance-run completion, API health, and disk growth. Changing this feature requires a graceful container restart.

Seen: 2026-10-11 (GitLab #31)
Seen again: 2026-10-11 (GitLab #31)

### Defective Container Healthchecks Mark Reachable Apps Unhealthy

**Symptoms**:
- `gaming-pc-stats`, `steam-stats`, `wizarr`, `life-progress`, and `homelab-chronicle` reported unhealthy on `docker-vm-core-utilities01`, although their application endpoints remained reachable.

**Cause**: Four Compose probes called tools absent from the final images: `curl` in three `python:3.11-slim` applications and `wget` in Chronicle's `node:20-slim` image. Wizarr's IPv4-only listener and hosts mapping suggest BusyBox `wget` selected IPv6 localhost (`::1`), though that selection was not directly reproduced. Separately, Chronicle's GitHub requests returned 404; whether the repository is unavailable or private access is missing remains unconfirmed.

**Solution**:
1. Backed up all five Compose files, recorded image IDs and mounts, and created SQLite snapshots plus SQL dumps for Wizarr and Chronicle.
2. Replaced probes with Python standard-library IPv4 `/health` requests for the three Python apps, a Node IPv4 `/api/events` request requiring HTTP 200 for Chronicle, and an IPv4 `wget` GET probe for Wizarr; pinned each service to its recorded running image ID.
3. Ran `docker compose config --quiet` for validation, then `docker compose up -d --no-deps --no-build --pull never` for only the affected services. Checked health probes, published endpoints, images, mounts, databases, saved data, and startup logs.
4. All five stayed healthy with five successful probes each, endpoint checks and database integrity checks passed, and saved data was preserved. Browser rendering and routed access for the Python apps remain unverified; Chronicle's GitHub 404 persists.

**Prevention**: Use probe clients included in each final image, specify IPv4 loopback when the app listens only on IPv4, and exercise the container healthcheck before deployment. Pin third-party images and alert separately on application integrations such as Chronicle's GitHub synchronization.

Seen: 2026-10-11 (GitLab #28)

### node02 NFS Mounts Fail During Boot

**Symptoms**:
- `mnt-nfs-lxcs.mount` and `mnt-nfs-media.mount` failed on node02; `/mnt/nfs/lxcs` and `/mnt/nfs/media` were not mounted.

**Cause**: A network-readiness race is the leading explanation: `network-online.target` may have completed before usable connectivity, no wait-online service was loaded, and carrier arrived after mounting began. This is not proven; connectivity returned 42 seconds before the mount deadline, and a temporary NAS/RPC outage or node02-specific NFSv3 filtering remains possible.

**Solution**:
1. Checked that both mount paths were empty, unmounted, and unused; retried only the two mount units and reviewed their status and journals.
2. Started `remote-fs.target` and confirmed both existing NFSv3 mounts from `192.168.20.32` were accessible.
3. Backed up `/etc/fstab`, added `x-systemd.mount-timeout=180s` to only the LXCs and Media entries, and ran `systemctl daemon-reload`.
4. Verified both generated units use a three-minute timeout, `remote-fs.target` is active, there are no failed units, all eight managed storage entries are active, and the 19 existing non-template guests remain running. No reboot was performed.

**Prevention**: Ensure network readiness includes physical carrier and a usable route before NFS mounts start, or evaluate automounts if access can be deferred. Investigate the 48-second NIC link negotiation with switch logs and driver history. Alert on missing mounts as well as failed units, and require these mounts before workloads write to the paths; the longer deadline is only a mitigation.

Seen: 2026-10-11 (GitLab #29)

### GitLab Patch Completion Delivery Causes a False Failure

**Symptoms**:
- Automated patching reported `Patch drift: gitlab-vm01` after host verification had succeeded.

**Cause**: The executor treated successful host verification and successful GitLab publication as one outcome. `gl_note()`, `gl_labels()`, and `gl_close()` logged API failures without propagating failure to finalization, which cleared `pending_verify`; recovery then treated a leftover `patching` label as proof of an interruption. Container presence was accepted in place of application readiness. GitLab reboot recovery most likely caused the transport failure, but the discarded HTTP 0 error means the specific cause is unconfirmed.

**Solution**:
1. Exported issue #3 and its notes, recorded reconciliation evidence, corrected its labels, and closed it as completed October 5 maintenance. Created issue #59 for current pending packages and left it unapproved.
2. Held `executor.lock`; backed up executor code, state, and history before editing, and preserved the original transcript.
3. Updated the executor to persist verification and completion-publication phases, retain API errors, retry publication idempotently with bounded retries, require application readiness, and clear state only after GitLab confirmation.
4. Passed all 16 regression tests and syntax validation, then restarted only `sre-agent.service` under the held lock. Verified GitLab readiness and authenticated API, issue state, kernel/package state, container health, failed units, and service journals. The executor was active without new journal errors.

**Prevention**: Keep upgrade, reboot, verification, and completion-publication phases separate. Require application readiness, preserve actionable transport errors, retry publication idempotently, and deduplicate notes/history. Confirm GitLab issue updates before clearing durable state. Treat a stale `patching` label as a reconciliation signal, not proof of a crash; verify authenticated API access, kernel, package audit, and container health.

Seen: 2026-10-11 (GitLab #3)


### Excluded PBS Patch Request Creates a Stale Failure Incident

**Symptoms**:
- Automated patching reported `Patch drift: pbs-server`, and the open GitLab issue continued to appear as a Medic failure despite PBS being intentionally unmanaged.

**Cause**: `sre_agent.py` explicitly excludes `pbs-server`, which is also absent from the managed SSH inventory. The executor routes this expected policy rejection through `fail()` and adds `patch-failed`; Medic scans open issues with that label without filtering excluded hosts.

**Solution**:
1. Confirmed the executor exclusion and inventory omission.
2. Exported issue #5 with restrictive umask and verified its IID, title, open state, and backup permissions.
3. Added `manual-maintenance`; removed `patch-failed`, `approved`, `awaiting-approval`, `patching`, and `patch-management`. Re-read the issue and ran Medic's exact open `patch-failed` issue query.
4. Checked CT100 status and PBS exporter responses, and assessed package candidates, task history, verification jobs, and snapshot metadata through node03. Confirmed root access through node01 and an independent CT100 archive on ProxmoxData; no restore test was performed.
5. The issue query no longer returned #5, CT100 remained running, and all checked exporter endpoints returned HTTP 200. The scheduled Medic scan ran but skipped detection while the supervising fix held the executor lock. PBS 3.4.9-2 had 17 cached upgrade candidates; package compliance and restore health remain unverified.

**Prevention**: Apply one eligibility policy to drift discovery, approval, execution, and incident scanning. Route excluded hosts to manual maintenance before approval, classify policy rejections as skipped rather than package failures, and track PBS maintenance separately from automated patching.

Seen: 2026-10-11 (GitLab #5)

### Excluded Controller Patch Requests Recur as Failure Issues

**Symptoms**:
- Automated patching reported `Patch drift: ansible-controller01`, and the open issue remained discoverable as a patch failure even though the executor intentionally excludes the host.

**Cause**: `orchestrator.sh` planned patch-management issues without checking executor exclusions or managed inventory. `sre_agent.py` excludes `ansible-controller01`, which is absent from `sre-hosts.txt`; the execution guard correctly refused the request but labeled it `patch-failed`, and Medic treated that open label as an incident.

**Solution**:
1. Exported issue #1 and confirmed its project, title, IID, and open state; added `manual-maintenance` and removed the automated workflow/failure labels.
2. Created required and timestamped backups of `orchestrator.sh`, then added the approved exclusion and inventory checks before deduplication, claiming, planning, or relabeling.
3. Ran `bash -n`, reviewed the diff without running the pipeline, and queried Medic's live failure discovery with host probes omitted and no state writes.
4. Verified the issue remained open with only `manual-maintenance`, Medic returned no matching failure, the planner guard passed syntax/behavior review, and VM103 remained running. No patching or guest changes occurred; controller package compliance remains unverified (6.8.0-142 running, reboot marker names 6.8.0-146).

**Prevention**: Share eligibility checks across discovery, planning, approval, execution, and incident classification. Route excluded-host drift to manual maintenance and report policy rejection as skipped or unsupported; do not advise reapproval for excluded targets.

Seen: 2026-10-11 (GitLab #1)

### Docker Patch Verification Fails During Container Startup

**Symptoms**:
- Automated patching reported `Patch drift: docker-lxc-media` while Jellyfin or cAdvisor was still `starting` after Docker restarted.
- The services became healthy automatically shortly after startup; the first verifier probe ran before readiness.

**Cause**: The verifier used a Docker polling deadline of 600 seconds only after a host reboot and zero seconds otherwise. A Docker restart without a host reboot therefore made the first `starting` health status fail verification; this was a verifier timing defect, not persistent application damage.

**Solution**:
1. Checked container status and health on `docker-lxc-media` through read-only SSH; confirmed all 16 media containers remained running and Jellyfin, cAdvisor, and MeTube were healthy.
2. Backed up the verifier and test source with `cp -a`, updated Docker verification to use its own 180-second deadline with 10-second polling, and kept `starting` pending until health checks pass.
3. Added offline regression checks, corrected a test-script quoting error after backing up that file, then ran `python3 -m py_compile /home/hermes-admin/sre_agent.py` and `python3 /home/hermes-admin/.olympus/runs/incident40_verifier_test.py`.
4. Restarted with `sudo -n systemctl restart sre-agent.service`; confirmed the service active/running and watcher startup in its journal. Syntax and all eight offline behavior groups passed; live startup polling awaits the next authorized patch run.

**Prevention**: After every Docker restart, poll every 10 seconds for up to 180 seconds; treat `starting` as pending, require checked containers to become healthy and unchecked containers to remain running without restarting, and keep missing baseline containers pending. On timeout, collect Docker health history and the last 200 application log lines. Roll back the verifier change by restoring its incident backup and restarting `sre-agent.service`.

Seen: 2026-10-16 (GitLab #341)
