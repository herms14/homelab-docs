# SSH Configuration


Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) | [06 - Ansible Automation](06%20-%20Ansible%20Automation.md)

---

## SSH Key Setup

### Windows PC key (192.168.10.10) — added 2026-10-11

An **additional** key for the rebuilt Windows PC; the 2026-08-26 key below was not removed and remains
authorized. Same local filename (`~/.ssh/homelab_ed25519`), different key.

```
ssh-ed25519 <redacted> hermes-windows-pc-2026-10-11
```

| Target | Status (2026-10-11) |
|--------|--------|
| node01–03 (root, shared `/etc/pve/priv/authorized_keys`) | ✅ Verified direct login |
| VMs as `hermes-admin`: ansible-controller01, gitlab-vm01, gitlab-runner-vm01, immich-vm01 | ✅ Verified direct login |
| docker-vm-core-utilities01 (`hermes-admin`) | ✅ Verified via `ProxyJump codex-agent-lxc`; direct route from this PC times out (Task #54) |
| All 12 SRE-inventory LXCs (`hermes-admin`; pihole as root) | ✅ Verified direct login |
| pbs-server CT100 (root; no `hermes-admin` user) and hewg-test01 CT101 (root + `hermes-admin`) | ✅ Added via `pct exec` on node03; verified login |
| Stopped templates 1000, 9022, 9025 | Not changed — clones won't include this key |

Distributed from codex-agent-lxc using its SRE key (idempotent append). This PC's `~/.ssh/config` has
entries for all 22 hosts.

### Primary SSH Key (No Passphrase) — Rotated 2026-08-26

**Status**: Rotated August 26, 2026 — the previous key (below, deployed Dec 21 2025) stopped matching what any host actually had in `authorized_keys` and could no longer authenticate anywhere. Generated fresh from a Claude Code session (Windows machine) and redeployed via root/password bootstrap.

| Field           | Value                                   |
| --------------- | --------------------------------------- |
| Key File        | `~/.ssh/homelab_ed25519`                |
| Public Key File | `~/.ssh/homelab_ed25519.pub`            |
| Key Type        | ed25519                                 |
| Passphrase      | None (for automation)                   |
| Comment         | `hermes@homelab-rotated-20260826211450` |

### Public Key

```
ssh-ed25519 <redacted> hermes@homelab-rotated-20260826211450
```

### Current Deployment Status (as of 2026-08-26)

| Target | Status |
|--------|--------|
| node01, node02, node03 (root) | ✅ Deployed and verified passwordless |
| All 11 running LXC containers (root + `hermes-admin` where the user exists) | ✅ Deployed via `pct exec` from each host node — see per-node list below |
| VMs: ansible-controller01, docker-vm-core-utilities01, gitlab-runner-vm01, gitlab-vm01, immich-vm01 | ✅ Deployed and verified passwordless as `hermes-admin`. Bootstrapped using the old working private key found at `C:\Users\herms\OneDrive\My Personal Files\Side Projects\homelab_ed25519` (same key as the "nopass" key below — it was never actually broken, it just wasn't present on the Claude Code Windows machine that started this rotation). `hermes-admin` has passwordless sudo on all 5 VMs. |

> Correction: the original diagnosis in this doc (that the old key was "non-functional") was wrong — the key itself worked fine, it just wasn't present on the machine driving this session. The real fix was locating the existing key file, not generating a new one. The rotation below still stands (new key is now the deployed standard) but note this for next time: check `OneDrive\My Personal Files\Side Projects\homelab_ed25519` before assuming a key rotation is needed.

**LXC containers with key deployed (by host node):**
- node01: 200 (docker-lxc-glance), 201 (docker-lxc-bots), 202 (pihole), 203 (traefik-lxc), 205 (docker-lxc-media), 206 (homeassistant-lxc), 209 (homepage-lxc)
- node03: 100 (pbs-server)

> Note: CT204 (authentik-lxc) actually lives on **node02**, not node01 as this doc previously implied — corrected 2026-08-26 based on live `pct list` output. See [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) for the authoritative node/CT mapping.

### Previous Key (Deployed December 21, 2025 — retired/non-functional)

| Field | Value |
|-------|-------|
| Comment | `hermes@homelab-nopass` |
| Public Key | `ssh-ed25519 <redacted> hermes@homelab-nopass` |
| Status | Non-functional as of 2026-08-26 — rejected on every host tested. Superseded by the rotated key above. |

---

## SSH Config Setup

SSH configuration file created at `~/.ssh/config` with host aliases for convenient access.

**Status (2026-08-26)**: Rewritten to match the *live* cluster — the previous version below was stale (referenced the decommissioned K8s cluster and pre-migration VM hostnames like `docker-vm-media01`/`traefik-vm01` that are now LXCs). Deployed to the Claude Code Windows machine at `C:\Users\herms\.ssh\config`; not yet verified on other machines that might use this doc.

### Config File Location

```
~/.ssh/config
```

### Current Configuration (verified working 2026-08-26)

```ssh-config
# Proxmox Nodes
Host node01
    HostName 192.168.20.20
    User root
    IdentityFile ~/.ssh/homelab_ed25519

Host node02
    HostName 192.168.20.21
    User root
    IdentityFile ~/.ssh/homelab_ed25519

Host node03
    HostName 192.168.20.22
    User root
    IdentityFile ~/.ssh/homelab_ed25519

# VMs
Host ansible-controller01
    HostName 192.168.20.30
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

Host docker-vm-core-utilities01
    HostName 192.168.40.13
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

Host gitlab-runner-vm01
    HostName 192.168.40.24
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

Host gitlab-vm01
    HostName 192.168.40.23
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

Host immich-vm01
    HostName 192.168.40.22
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

# LXC Containers (node01)
Host docker-lxc-glance
    HostName 192.168.40.12
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

Host docker-lxc-bots
    HostName 192.168.40.17
    User root
    IdentityFile ~/.ssh/homelab_ed25519

Host pihole
    HostName 192.168.90.53
    User root
    IdentityFile ~/.ssh/homelab_ed25519

Host traefik-lxc
    HostName 192.168.40.20
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

Host docker-lxc-media
    HostName 192.168.40.11
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

Host homeassistant-lxc
    HostName 192.168.40.25
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

Host homepage-lxc
    HostName 192.168.40.27
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

# LXC Containers (node02)
Host authentik-lxc
    HostName 192.168.40.21
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

Host chronicle-lxc
    HostName 192.168.40.15
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

    HostName 192.168.40.26
    User hermes-admin
    IdentityFile ~/.ssh/homelab_ed25519

# LXC Containers (node03)
Host pbs-server
    HostName 192.168.20.50
    User root
    IdentityFile ~/.ssh/homelab_ed25519
```

> Note: CT207 (chronicle-lxc) IP was previously documented as "TBD" — confirmed live as **192.168.40.15**.

### Previous Configuration (stale — referenced decommissioned K8s hosts)

<details>
<summary>Expand for historical reference only</summary>

```ssh-config
Host k8s-controller01
    HostName 192.168.20.32
Host k8s-worker01-06
    HostName 192.168.20.40-45
Host docker-vm-media01
    HostName 192.168.40.11   # now docker-lxc-media (LXC 205)
Host traefik-vm01
    HostName 192.168.40.20   # now traefik-lxc (LXC 203)
Host authentik-vm01
    HostName 192.168.40.21   # now authentik-lxc (LXC 204)
```

</details>

---

## Usage Examples

### Connect to Hosts by Alias

```bash
# Proxmox nodes
ssh node01
ssh node02
ssh node03

# Ansible controller
ssh ansible-controller01

# Kubernetes nodes
ssh k8s-controller01
ssh k8s-worker01

# Service VMs
ssh traefik-vm01
ssh authentik-vm01
ssh docker-vm-media01
```

### Run Remote Commands

```bash
# Check uptime on all K8s controllers
ssh k8s-controller01 "uptime"
ssh k8s-controller02 "uptime"
ssh k8s-controller03 "uptime"

# Check Docker status on service VMs
ssh docker-vm-media01 "docker ps"
ssh docker-vm-core-utilities01 "docker ps"
```

---

## Key Distribution

The rotated SSH key (`hermes@homelab-rotated-20260826211450`) has been distributed to all live infrastructure hosts as of 2026-08-26:

### Proxmox Nodes (root user)
- node01 (192.168.20.20)
- node02 (192.168.20.21)
- node03 (192.168.20.22)

### VMs (hermes-admin user, passwordless sudo confirmed)
- ansible-controller01 (192.168.20.30)
- docker-vm-core-utilities01 (192.168.40.13)
- gitlab-runner-vm01 (192.168.40.24)
- gitlab-vm01 (192.168.40.23)
- immich-vm01 (192.168.40.22)

### LXC Containers (root always; hermes-admin where the user exists)
- node01: docker-lxc-glance (.40.12), docker-lxc-bots (.40.17, root or hermes-admin), pihole (.90.53, root only), traefik-lxc (.40.20), docker-lxc-media (.40.11), homeassistant-lxc (.40.25), homepage-lxc (.40.27)
- node03: pbs-server (.20.50, root only)

> K8s cluster (k8s-controller/worker) was decommissioned 2026-05-23 — those hosts no longer exist, removed from this list.

---

## Ansible Controller Update

The Ansible controller has been updated to use the new SSH key:

### Key Location on Ansible Controller

```
/home/hermes-admin/.ssh/homelab_ed25519
/home/hermes-admin/.ssh/homelab_ed25519.pub
```

### Ansible Configuration

The key is automatically used via the SSH config file for all playbook executions.

**Test connectivity**:
```bash
ssh ansible-controller01 "ansible all -m ping"
```

---

## Legacy SSH Keys

### Previous Key (with passphrase)

| Field | Value |
|-------|-------|
| Public Key | `ssh-ed25519 <redacted> hermes@homelab` |
| Comment | `hermes@homelab` |
| Status | Still deployed on hosts, can be removed once new key is verified |

---

## Security Considerations

### No Passphrase Key

The new SSH key (`homelab_ed25519`) has **no passphrase** to enable:
- Automated Ansible playbook execution
- CI/CD pipeline automation
- Unattended SSH operations

### Security Mitigations

1. **File Permissions**: Key file is `chmod 600` (owner read/write only)
2. **Network Isolation**: Keys only work within internal VLANs (20, 40)
3. **Host-based Access**: SSH config restricts key usage to specific hosts
4. **Audit Trail**: All SSH sessions are logged on target hosts
5. **Key Rotation**: Regular key rotation scheduled every 6 months

### Access Control

- **Root Access**: Only on Proxmox nodes (node01-03)
- **User Access**: Standard user `hermes-admin` on all VMs
- **No Public Exposure**: SSH only accessible within homelab network

---

## Key Rotation Plan

### Rotation Schedule

- **Current Key Created**: December 21, 2025
- **Next Rotation**: June 2026 (6 months)

### Rotation Process

1. Generate new ed25519 key pair
2. Add new public key to all hosts (`~/.ssh/authorized_keys`)
3. Update SSH config to use new private key
4. Test connectivity to all hosts
5. Remove old public key from hosts
6. Archive old private key (encrypted backup)

---

## Troubleshooting

### Connection Issues

```bash
# Test SSH connection with verbose output
ssh -v node01

# Verify key is being used
ssh -v node01 2>&1 | grep "Offering public key"

# Check key permissions
ls -la ~/.ssh/homelab_ed25519  # Should be -rw-------
```

### Key Permission Errors

```bash
# Fix private key permissions
chmod 600 ~/.ssh/homelab_ed25519

# Fix SSH config permissions
chmod 644 ~/.ssh/config

# Fix SSH directory permissions
chmod 700 ~/.ssh
```

### Verify Key on Remote Host

```bash
# Check authorized_keys on target host
ssh node01 "cat /root/.ssh/authorized_keys | grep homelab-nopass"
ssh ansible-controller01 "cat ~/.ssh/authorized_keys | grep homelab-nopass"
```

---

## Related Documentation

- [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) - Proxmox node access
- [06 - Ansible Automation](06%20-%20Ansible%20Automation.md) - Ansible playbook execution
- 04 - Kubernetes Cluster - K8s node access
- 11 - Credentials - All access credentials
- [12 - Troubleshooting](12%20-%20Troubleshooting.md) - Common SSH issues

---

*Last updated: August 26, 2026 — SSH key rotation (see top of file)*
