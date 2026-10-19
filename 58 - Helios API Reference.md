---
tags: [type/note, status/evergreen, domain/technical/homelab, source/claude]
created: 2026-10-03
updated: 2026-10-03
type: note
status: evergreen
---

# Helios API Reference

> Reference for the Helios homelab CLI/API — the "Azure ARM for your homelab" control plane wrapping Proxmox, Synology, Omada, Hyper-V (unconfigured), Ansible, and Terraform behind one REST API + CLI. Generated from its live OpenAPI spec (`http://192.168.40.14:8000/openapi.json`) on 2026-10-03.


---

## Where it runs

| Field | Value |
|-------|-------|
| Host | helios-lxc (CT210), `192.168.40.14:8000` |
| Deploy | systemd + Python venv (`helios.service`), unprivileged LXC — moved here from ansible-controller01 2026-10-03 |
| Auth | `X-API-Key` header — see 11 - Credentials → "Helios" |
| Interactive docs | `http://192.168.40.14:8000/docs` (Swagger UI) — this note is a static mirror of that, for offline reference and linking from other vault notes |
| From Kratos | `helios <command>` (PowerShell function in `$PROFILE`, proxies via SSH to the real CLI on helios-lxc) or direct REST calls |

**Known gap**: the `ansible` and `terraform` endpoints below shell out to local paths (`~/ansible/ansible-playbooks`, `~/homelab-infra-automation-project/terraform`) that only exist on `ansible-controller01`, not on helios-lxc — those two sections currently won't work from the new host. See Task #43.

---

## compute — Proxmox VMs & LXCs

| Method | Path | Summary | Params |
|--------|------|---------|--------|
| GET | `/api/v1/compute/nodes` | List all Proxmox nodes | — |
| GET | `/api/v1/compute/nodes/{node}` | Get node details | `node` (path, required) |
| GET | `/api/v1/compute/nodes/{node}/storage` | List storage on a node | `node` (path, required) |
| GET | `/api/v1/compute/vms` | List all VMs (optionally filter by node) | `node` (query, optional) |
| POST | `/api/v1/compute/vms` | Create a VM by cloning a template | body: `VMCreateRequest` |
| GET | `/api/v1/compute/vms/{node}/{vmid}` | Get VM details | `node`, `vmid` (path, required) |
| DELETE | `/api/v1/compute/vms/{node}/{vmid}` | Delete a VM | `node`, `vmid` (path, required) |
| POST | `/api/v1/compute/vms/{node}/{vmid}/start` | Start a VM | `node`, `vmid` (path, required) |
| POST | `/api/v1/compute/vms/{node}/{vmid}/stop` | Force stop a VM | `node`, `vmid` (path, required) |
| POST | `/api/v1/compute/vms/{node}/{vmid}/shutdown` | Gracefully shut down a VM | `node`, `vmid` (path, required) |
| GET | `/api/v1/compute/templates` | List all VM templates across all nodes | — |
| GET | `/api/v1/compute/tasks/{node}/{task_id}` | Get task status | `node`, `task_id` (path, required) |
| GET | `/api/v1/compute/lxc` | List all LXC containers (optionally filter by node) | `node` (query, optional) |
| POST | `/api/v1/compute/lxc` | Create an LXC container by cloning a template | body: `LXCCreateRequest` |
| GET | `/api/v1/compute/lxc/{node}/{vmid}` | Get LXC container details | `node`, `vmid` (path, required) |
| DELETE | `/api/v1/compute/lxc/{node}/{vmid}` | Delete an LXC container | `node`, `vmid` (path, required) |
| POST | `/api/v1/compute/lxc/{node}/{vmid}/start` | Start an LXC container | `node`, `vmid` (path, required) |
| POST | `/api/v1/compute/lxc/{node}/{vmid}/stop` | Force stop an LXC container | `node`, `vmid` (path, required) |
| POST | `/api/v1/compute/lxc/{node}/{vmid}/shutdown` | Gracefully shut down an LXC container | `node`, `vmid` (path, required) |
| GET | `/api/v1/compute/lxc-templates` | List all LXC templates across all nodes | — |

### `VMCreateRequest` body

| Field | Type | Required |
|-------|------|----------|
| `name` | string | ✅ |
| `node` | string | ✅ |
| `template_id` | integer | ✅ |
| `cores` | integer | |
| `memory_mb` | integer | |
| `disk_gb` | integer \| null | |
| `os_type` | OSType (enum) | |
| `network_bridge` | string | |
| `vlan_tag` | integer \| null | |
| `start_on_create` | boolean | |
| `description` | string | |
| `tags` | array | |
| `pool` | string \| null | |

### `LXCCreateRequest` body

Same shape as `VMCreateRequest` minus `os_type`, plus:

| Field | Type | Required |
|-------|------|----------|
| `unprivileged` | boolean | |

---

## compute-hyperv — Hyper-V (unconfigured, KratosPC retired)

| Method | Path | Summary | Params |
|--------|------|---------|--------|
| GET | `/api/v1/compute/hyperv/host` | Get Hyper-V host (KratosPC) info | — |
| GET | `/api/v1/compute/hyperv/vms` | List all Hyper-V VMs | — |
| POST | `/api/v1/compute/hyperv/vms` | Create a Hyper-V VM | body: `HyperVVMCreateRequest` |
| GET | `/api/v1/compute/hyperv/vms/{name}` | Get Hyper-V VM details | `name` (path, required) |
| DELETE | `/api/v1/compute/hyperv/vms/{name}` | Delete a Hyper-V VM and its VHDs | `name` (path, required) |
| POST | `/api/v1/compute/hyperv/vms/{name}/start` | Start a Hyper-V VM | `name` (path, required) |
| POST | `/api/v1/compute/hyperv/vms/{name}/stop` | Stop a Hyper-V VM | `name` (path, required); `force` (query, optional, bool) |
| POST | `/api/v1/compute/hyperv/vms/{name}/checkpoint` | Create a VM checkpoint/snapshot | `name` (path, required); `checkpoint_name` (query, optional) |
| GET | `/api/v1/compute/hyperv/vms/{name}/checkpoints` | List VM checkpoints | `name` (path, required) |
| GET | `/api/v1/compute/hyperv/switches` | List virtual switches | — |

> Per `config.yaml`, Hyper-V host is unconfigured (`192.168.x.x`) — KratosPC's Hyper-V role was retired. These endpoints will not return useful data until/unless re-wired.

---

## storage — Synology NAS

| Method | Path | Summary | Params |
|--------|------|---------|--------|
| GET | `/api/v1/storage/info` | Get NAS system info | — |
| GET | `/api/v1/storage/shares` | List all NAS shared folders | — |
| GET | `/api/v1/storage/folders` | List contents of a folder | `path` (query, required) |
| POST | `/api/v1/storage/folders` | Create a folder inside a share | body: `FolderCreateRequest` |
| DELETE | `/api/v1/storage/folders` | Delete a folder or file | body: `FolderDeleteRequest` |
| GET | `/api/v1/storage/volumes` | List NAS volumes and disk usage | — |

`FolderCreateRequest` / `FolderDeleteRequest`: `share` (string, required), `path` (string, required).

> Synology auth is currently blocked — `hermes-admin` DSM account has 2FA enabled, which Helios's auth flow doesn't support. See Task #40.

---

## network — Omada Controller

| Method | Path | Summary | Params |
|--------|------|---------|--------|
| GET | `/api/v1/network/sites` | List Omada controller sites | — |
| GET | `/api/v1/network/clients` | List currently connected clients | — |
| GET | `/api/v1/network/devices` | List network devices (APs, switches, gateway) | — |
| GET | `/api/v1/network/vlans` | List all VLANs / networks | — |
| POST | `/api/v1/network/vlans` | Create a new VLAN / network | body: `VLANCreateRequest` |
| DELETE | `/api/v1/network/vlans/{network_id}` | Delete a VLAN / network by ID | `network_id` (path, required) |
| GET | `/api/v1/network/stats` | Get network overview statistics | — |
| GET | `/api/v1/network/wireless` | List wireless networks (SSIDs) | — |

### `VLANCreateRequest` body

| Field | Type | Required |
|-------|------|----------|
| `name` | string | ✅ |
| `vlan_id` | integer | ✅ |
| `purpose` | string | |
| `subnet` | string \| null | |
| `gateway` | string \| null | |
| `dhcp_enabled` | boolean | |
| `description` | string | |

> Uses the `claude-reader` read-only Omada account per `config.yaml` — write endpoints here (`POST`/`DELETE` vlans) will likely fail permission checks with that account. Task #41 notes the data endpoints also hit permission/path errors with this controller's Omada SDN 6.2.10.18 — unresolved.

---

## images — ISO/disk image management

| Method | Path | Summary | Params |
|--------|------|---------|--------|
| GET | `/api/v1/images/isos` | List ISOs on NAS | `os` (query, optional) |
| GET | `/api/v1/images/disk-images` | List disk images (qcow2, vhdx, vmdk) on NAS | `os` (query, optional) |
| GET | `/api/v1/images/search` | Search images by name | `q` (query, required) |
| GET | `/api/v1/images/proxmox/templates` | List Proxmox VM templates | — |
| GET | `/api/v1/images/proxmox/{node}/isos` | List ISOs on a Proxmox node storage | `node` (path, required); `storage` (query, optional) |
| POST | `/api/v1/images/proxmox/{node}/upload` | Upload ISO from NAS to Proxmox node | `node` (path, required); body: `ImageSyncRequest` |

### `ImageSyncRequest` body

| Field | Type | Required |
|-------|------|----------|
| `source_path` | string | ✅ |
| `target_platform` | ImagePlatform (enum) | ✅ |
| `source_share` | string | |
| `target_node` | string \| null | |
| `target_storage` | string \| null | |

---

## ansible — Playbook execution

⚠️ **Won't work from helios-lxc** — see "Known gap" above. Paths are relative to ansible-controller01.

| Method | Path | Summary | Params |
|--------|------|---------|--------|
| GET | `/api/v1/ansible/playbooks` | List available playbooks | — |
| GET | `/api/v1/ansible/playbooks/{playbook_path}` | Get playbook content | `playbook_path` (path, required) |
| GET | `/api/v1/ansible/inventory` | Get inventory structure | — |
| GET | `/api/v1/ansible/inventory/raw` | Get raw inventory file | — |
| POST | `/api/v1/ansible/run` | Run an Ansible playbook | body: `PlaybookRunRequest` |
| GET | `/api/v1/ansible/jobs` | List all Ansible jobs | — |
| GET | `/api/v1/ansible/jobs/{job_id}` | Get Ansible job status and output | `job_id` (path, required) |

### `PlaybookRunRequest` body

| Field | Type | Required |
|-------|------|----------|
| `playbook` | string | ✅ |
| `inventory` | string \| null | |
| `limit` | string \| null | |
| `tags` | string \| null | |
| `skip_tags` | string \| null | |
| `extra_vars` | object \| null | |
| `check_mode` | boolean | |
| `verbose` | integer | |

---

## terraform — Workspace execution

⚠️ **Won't work from helios-lxc** — see "Known gap" above. Paths are relative to ansible-controller01; this homelab doesn't centrally organize Terraform workspaces anyway (Task #43).

| Method | Path | Summary | Params |
|--------|------|---------|--------|
| GET | `/api/v1/terraform/workspaces` | List Terraform workspaces | — |
| GET | `/api/v1/terraform/workspaces/{name}` | Get workspace details | `name` (path, required) |
| GET | `/api/v1/terraform/workspaces/{name}/state` | Get workspace state resources | `name` (path, required) |
| GET | `/api/v1/terraform/workspaces/{name}/outputs` | Get workspace Terraform outputs | `name` (path, required) |
| POST | `/api/v1/terraform/run` | Run a Terraform command (init, plan, apply, destroy) | body: `TerraformRunRequest` |
| GET | `/api/v1/terraform/jobs` | List all Terraform jobs | — |
| GET | `/api/v1/terraform/jobs/{job_id}` | Get Terraform job status and output | `job_id` (path, required) |

### `TerraformRunRequest` body

| Field | Type | Required |
|-------|------|----------|
| `workspace` | string | ✅ |
| `action` | TerraformAction (enum: init/plan/apply/destroy) | ✅ |
| `auto_approve` | boolean | |
| `variables` | object \| null | |
| `var_file` | string \| null | |
| `targets` | array \| null | |

---

## health

| Method | Path | Summary |
|--------|------|---------|
| GET | `/` | Root |
| GET | `/health` | Health check — `{"status": "ok"}`, no auth required |

---

## Usage examples

```powershell
# From Kratos, via the helios function (see $PROFILE)
helios compute nodes
helios compute vms
helios compute lxc

# Direct REST call
$headers = @{ "X-API-Key" = "<see 11 - Credentials.md>" }
Invoke-RestMethod -Uri "http://192.168.40.14:8000/api/v1/compute/nodes" -Headers $headers
```

---

## Related Documentation

- 11 - Credentials — Helios API key, Proxmox token, host details
- [Homelab Changelog](Homelab%20Changelog.md) — 2026-10-03 entry for the LXC relocation
