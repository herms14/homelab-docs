# Ansible Automation

> **Internal Documentation** - Contains playbook structure and inventory details.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [05 - Terraform Configuration](05%20-%20Terraform%20Configuration.md) | [07 - Deployed Services](07%20-%20Deployed%20Services.md)

---

## Ansible Controller

| Setting | Value |
|---------|-------|
| Host | ansible-controller01 |
| IP | 192.168.20.30 |
| User | hermes-admin |
| Playbooks | `~/ansible/` |
| SSH Key | `~/.ssh/homelab_ed25519` (no passphrase) |

See [16 - SSH Configuration](16%20-%20SSH%20Configuration.md) for SSH key setup and host aliases.

---

## Playbook Structure

```
~/ansible/
├── ansible.cfg
├── inventory/
│   └── hosts.yml
├── docker/
│   ├── install-docker.yml
│   └── deploy-arr-stack.yml
├── traefik/
│   └── deploy-traefik.yml
├── authentik/
│   └── deploy-authentik.yml
├── immich/
│   └── deploy-immich.yml
├── gitlab/
│   └── deploy-gitlab.yml
├── n8n/
│   └── deploy-n8n.yml
├── k8s/
│   ├── k8s-deploy-all.yml
│   └── ...
├── opnsense/
│   ├── add-dns-record.yml
│   └── add-all-services-dns.yml
└── callback_plugins/
    └── discord_notify.py
```

---

## Common Playbooks

### Docker Installation

```bash
ansible-playbook docker/install-docker.yml -l docker_hosts
```

### Service Deployments

```bash
# Arr Media Stack
ansible-playbook docker/deploy-arr-stack.yml

# Traefik
ansible-playbook traefik/deploy-traefik.yml

# Authentik
ansible-playbook authentik/deploy-authentik.yml

# Immich
ansible-playbook immich/deploy-immich.yml

# GitLab
ansible-playbook gitlab/deploy-gitlab.yml

# n8n
ansible-playbook n8n/deploy-n8n.yml
```

### Kubernetes Cluster

```bash
# Full cluster deployment
ansible-playbook k8s/k8s-deploy-all.yml
```

## Inventory Groups

```yaml
all:
  children:
    proxmox_nodes:
      hosts:
        node01: {ansible_host: 192.168.20.20}
        node02: {ansible_host: 192.168.20.21}
        node03: {ansible_host: 192.168.20.22}

    docker_hosts:
      hosts:
        docker-vm-core-utilities01: {ansible_host: 192.168.40.13}
        docker-vm-media01: {ansible_host: 192.168.40.11}
        traefik-vm01: {ansible_host: 192.168.40.20}
        authentik-vm01: {ansible_host: 192.168.40.21}
        immich-vm01: {ansible_host: 192.168.40.22}
        gitlab-vm01: {ansible_host: 192.168.40.23}

    k8s_controllers:
      hosts:
        k8s-controller01: {ansible_host: 192.168.20.32}
        k8s-controller02: {ansible_host: 192.168.20.33}
        k8s-controller03: {ansible_host: 192.168.20.34}

    k8s_workers:
      hosts:
        k8s-worker01: {ansible_host: 192.168.20.40}
        k8s-worker02: {ansible_host: 192.168.20.41}
        k8s-worker03: {ansible_host: 192.168.20.42}
        k8s-worker04: {ansible_host: 192.168.20.43}
        k8s-worker05: {ansible_host: 192.168.20.44}
        k8s-worker06: {ansible_host: 192.168.20.45}

```

---

## OPNsense DNS Automation

### Prerequisites

Set environment variables:
```bash
export OPNSENSE_API_KEY="<redacted>"
export OPNSENSE_API_SECRET="<redacted>"
```

### Add DNS Record

```bash
ansible-playbook opnsense/add-dns-record.yml -e "hostname=myservice ip=192.168.40.13"
```

### Add All Homelab Services

```bash
ansible-playbook opnsense/add-all-services-dns.yml
```

---

## Discord Notifications

Automatic notifications at end of every playbook run.

### Setup

1. Create Discord Webhook in server settings
2. Set environment variable:
   ```bash
   export DISCORD_WEBHOOK_URL="<redacted>"
   ```

### Notification Format

```
✅ Playbook: deploy-arr-stack.yml
📊 Status: Success    🖥️ Hosts: 1    ⏱️ Duration: 2m 15s

📈 Task Summary
✓ OK:      45
↻ Changed: 3
⊘ Skipped: 2
✗ Failed:  0

🎯 Hosts: docker-vm-media01
```

---

## Ad-hoc Commands

```bash
# Check connectivity
ansible all -m ping

# Check uptime
ansible all -a uptime

# Gather facts
ansible <host> -m setup

# Run command
ansible docker_hosts -a "docker ps"
```

---

## Related Documentation

- [05 - Terraform Configuration](05%20-%20Terraform%20Configuration.md) - Infrastructure deployment
- [16 - SSH Configuration](16%20-%20SSH%20Configuration.md) - SSH keys and access
- [07 - Deployed Services](07%20-%20Deployed%20Services.md) - Service details
- 04 - Kubernetes Cluster - K8s deployment
- 11 - Credentials - API keys and tokens

