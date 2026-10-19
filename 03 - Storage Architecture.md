# Storage Architecture

> **Internal Documentation** - Contains NAS configuration and mount details.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) | [08 - Arr Media Stack](08%20-%20Arr%20Media%20Stack.md)

---

## Architecture Overview

The cluster uses a production-grade NFS storage architecture with dedicated exports for each content type.

**Design Rule**: One NFS export = One Proxmox storage pool

---

## Synology NAS Configuration

| Interface | IP | Speed | Switch | Purpose |
|-----------|-----|-------|--------|---------|
| eth0 (original) | 192.168.20.31 | 1 GbE | Morpheus Switch (SG2210P) | Legacy — kept for management |
| eth1 (10GbE NIC) | **192.168.20.32** | **2.5 GbE** | New 2.5G switch | **Primary NFS — all mounts** |

> **Standard**: ALL NFS mounts cluster-wide (nodes, VMs, containers) MUST use **192.168.20.32**.
> `192.168.20.31` is management/legacy only — never use it in fstab entries.

| Storage Pool | Export Path | Type | Content | Management |
|--------------|-------------|------|---------|------------|
| **VMDisks** | `<redacted>` | NFS | Disk image | Proxmox-managed |
| **ISOs** | `<redacted>` | NFS | ISO image | Proxmox-managed |
| **LXC Configs** | `/volume2/Proxmox-LXCs` | NFS | App data | Manual mount |
| **Media** | `/volume2/Proxmox-Media` | NFS | Media files | Manual mount |
| **ProxmoxData** | `/volume2/ProxmoxData` | NFS | App data (Immich) | Manual mount |
| **local-lvm** | N/A | Local LVM | Container | Local storage |

---

## Proxmox Storage Pools

### VMDisks

VM disk images with full Proxmox integration.

```
ID: VMDisks
Server: 192.168.20.32
Export: <redacted>
Content: Disk image
Nodes: All nodes
```

- **Used for**: VM virtual disks, cloud-init drives
- **Enables**: Live migration, snapshots, HA

### ISOs

Installation media storage.

```
ID: ISOs
Server: 192.168.20.32
Export: <redacted>
Content: ISO image
Nodes: All nodes
```

---

## Manual NFS Mounts

### /etc/fstab Configuration

Add to `/etc/fstab` on all Proxmox nodes (uses 2.5G NIC at .32):

```bash
192.168.20.32:/volume2/Proxmox-LXCs   /mnt/nfs/lxcs   nfs  rw,soft,timeo=30,retrans=3,noatime,nfsvers=3,_netdev  0  0
192.168.20.32:/volume2/Proxmox-Media  /mnt/nfs/media  nfs  rw,soft,timeo=30,retrans=3,noatime,nfsvers=3,_netdev  0  0
```

### Setup Commands

```bash
# Create mount points
mkdir -p /mnt/nfs/lxcs
mkdir -p /mnt/nfs/media

# Mount all
mount -a

# Verify
df -h | grep /mnt/nfs
```

---

## Docker Host Mounts

### Media Mount (docker-lxc-media / node01 host)

NFS is mounted on the Proxmox host (node01); the media LXC (CT205) accesses it via bind mount.

```bash
# /etc/fstab on node01
192.168.20.32:/volume2/Proxmox-Media /mnt/nfs/media nfs rw,soft,timeo=30,retrans=3,noatime,nfsvers=3,_netdev 0 0
```

**Directory Structure**:
- `/mnt/media/Movies`
- `/mnt/media/Series`
- `/mnt/media/Music`
- `/mnt/media/Downloads`

### Immich Mount (immich-vm01)

```bash
# /etc/fstab — all mounts use 192.168.20.32 (2.5G NIC)
192.168.20.32:/volume2/ProxmoxData       /mnt/appdata          nfs   rw,soft,intr,timeo=300,retrans=3,_netdev,nofail  0  0
192.168.20.32:/volume2/homes             /mnt/synology-root    nfs4  ro,vers=4,_netdev                                0  0
192.168.20.32:/volume2/Immich\ Photos    /mnt/immich-uploads   nfs4  rw,vers=4,_netdev                               0  0
# bind mount (no NFS)
/mnt/synology-root/hermes-admin/Photos   /mnt/synology-photos  none  bind,ro                                         0  0
```

**Directory Structure** (7TB capacity):
- `/mnt/appdata/immich/upload/`
- `/mnt/appdata/immich/library/`
- `/mnt/appdata/immich/profile/`
- `/mnt/immich-uploads/` — direct Immich Photos export
- `/mnt/synology-photos/` — legacy Photos bind mount (read-only)

---

## LXC Bind Mount Strategy

Bind-mount NFS subdirectories into containers for persistent config.

### Example: Traefik Container

Container config (`/etc/pve/lxc/100.conf`):
```
mp0: /mnt/nfs/lxcs/traefik,mp=/app/config
```

**Flow**:
1. Host has `/mnt/nfs/lxcs` mounted via NFS
2. Subdirectory `/mnt/nfs/lxcs/traefik/` bind-mounted into container
3. Container sees `/app/config` as normal directory
4. Data persists on NAS at `<redacted>`

---

## Why This Architecture Works

| Issue | Solution |
|-------|----------|
| Inactive storage warnings | Each storage has dedicated export |
| `?` icons in UI | Homogeneous content types per storage |
| Template clone failures | All storages available on all nodes |
| LXC rootfs errors | App configs are manual mounts |
| Performance degradation | Media not scanned by Proxmox |
| Migration issues | Identical paths across nodes |

---

## Related Documentation

- [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) - Cluster configuration
- [07 - Deployed Services](07%20-%20Deployed%20Services.md) - Service storage paths
- [08 - Arr Media Stack](08%20-%20Arr%20Media%20Stack.md) - Media storage configuration
- [05 - Terraform Configuration](05%20-%20Terraform%20Configuration.md) - IaC storage configuration

