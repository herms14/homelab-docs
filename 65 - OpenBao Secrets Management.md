---
created: 2026-10-18
updated: 2026-10-19
type: note
status: growing
tags: [homelab, security, secrets, openbao, roundtable]
related: ["11 - Credentials", "Agents", "[10 - IP Address Map](10%20-%20IP%20Address%20Map.md)"]
---
# OpenBao Secrets Management

> [!NOTE] No secret values in this note
> It records where secrets are kept and who may read them. Values live only in OpenBao and, until the fallback is retired, in the CT211 env files.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | 11 - Credentials | [02 - Proxmox Cluster](02%20-%20Proxmox%20Cluster.md) | Agents

Deployed 2026-10-18 by Claude Code (owner approval: "i want you to deploy it", unseal choice "Auto-unseal on its LXC").

## Design

| Part | What it is | Can it use a secret? |
|---|---|---|
| **OpenBao** on `openbao-lxc` (CT213) | The store. KV v2 mount `homelab`, one secret per **scope** (e.g. `homelab/omada`). File audit log. | Holds them encrypted |
| **Broker** (`olympus_core.load_env` → `bao_broker.py`) | Deterministic code, no model. Each Roundtable entry script reads only its own scopes with its own token, then the env files fill any gap. | The calling script gets its values, as before |
| **Varys** (`keeper.py`, agent id `keeper`) | Keeper persona. Keeps the map of who may read what, sends rotation reminders, reads the audit summary, turns access requests into owner approvals. | **No.** Its token reads metadata, policies and token accessors, never data; its process loads only `.discord_env` to post |

The scope plan is code, not a guess: `keeper.py plan` reads every script's imports (`secret_scopes.json` maps key names to scopes) and gives a script a scope only when its code (or a module it imports) references one of that scope's keys.

## Host

| Item | Value |
|---|---|
| Container | CT213 `openbao-lxc` on **node02**, unprivileged Ubuntu 24.04, 2 vCPU, 1 GiB RAM, 8 GB root (`local-lvm`), `onboot: 1` |
| Address | 192.168.40.39/24 (VLAN 40), DNS .60/.53, IPAM reservation `openbao-vault` active |
| OpenBao | 2.7.1 (`.deb` from the GitHub release, SHA-256 checked against `checksums.txt`), raft storage `/opt/openbao/data`, config `/etc/openbao/openbao.hcl` |
| API / UI | `https://192.168.40.39:8200` (agents) · `https://bao.hrmsmrflrii.xyz` (Traefik, **LAN and Tailscale only**, OpenBao token login) |
| TLS | Own CA "Hermes Homelab OpenBao CA" (10 y) in `/etc/openbao/tls/`; server cert SANs 192.168.40.39, 127.0.0.1, openbao-lxc, openbao-lxc.hrmsmrflrii.xyz, bao.hrmsmrflrii.xyz (825 d, renew before **2029-01**). CA copies: CT211 `~/.olympus/bao/ca.crt`, Traefik `/opt/traefik/certs/openbao-ca.crt`, Prometheus `/opt/monitoring/prometheus/openbao-ca.crt` |
| Unseal | 1 key share. Key on the **no-backup** mount `/etc/openbao-unseal/unseal.key` (mp0, `backup=0`, root 400). `openbao-unseal.service` unseals after every start (`/usr/local/sbin/openbao-unseal`, key via stdin, never on a command line) |
| Root token | `/root/openbao-root.token` (root 400). `/usr/local/sbin/openbao-admin-token` prints a 1-hour admin token for `bao_setup.py` |
| Audit | `/var/log/openbao/audit.log` (declared in the config; OpenBao 2.7 refuses API-enabled audit devices), logrotate daily ×30, values HMACed. `openbao-audit-summary.timer` (hourly) writes a value-free summary to `/var/lib/openbao-report/summary.json`; user `bao-report` can only `cat` it (forced command, Varys's key `~/.ssh/keeper_ed25519` on CT211) |
| Metrics | Prometheus jobs `openbao` (`/v1/sys/metrics`, unauthenticated metrics only) and `node-openbao-lxc` (:9100) |
| Backups | Added to PBS jobs `backup-984a1a4f-60ee` (daily) and `backup-78f24e5d-ed4a` (Fri); the unseal mount is excluded |
| Access | `hermes-admin` (passwordless sudo) with the Kratos `homelab_ed25519` and CT211 `sre_agent_ed25519` keys; listed in CT211 `~/.ssh/sre-hosts.txt` (so it is patched by the SRE pipeline; it auto-unseals after reboot) |

> [!WARNING] Keep an offline copy of the unseal key and root token
> Backups contain the encrypted data and the root token but **not** the unseal key. If CT213's mp0 disk is lost without an offline copy, every secret in OpenBao is unrecoverable (the env files on CT211 are still the fallback today). Copy each to your password manager from Kratos, one at a time, without printing them:
> `ssh -i ~/.ssh/homelab_ed25519 hermes-admin@192.168.40.39 sudo cat /etc/openbao-unseal/unseal.key | clip.exe`
> `ssh -i ~/.ssh/homelab_ed25519 hermes-admin@192.168.40.39 sudo cat /root/openbao-root.token | <redacted>`
> Auto-unseal on the same host protects against stolen backups and disks, not against someone with root on CT213 or node02. That was the accepted trade-off.

## Scopes (key names only)

Source: `agents/olympus/secret_scopes.json`. 16 scopes, 39 keys, imported 2026-10-18 from CT211 `~/.discord_env`, `.codex_env`, `.olympus_env`, `.gitlab_env`.

| Scope | Keys | Rotation reminder |
|---|---|---|
| discord | DISCORD_TOKEN, DISCORD_GUILD_ID | 365 d |
| openai | OPENAI_API_KEY | 180 d |
| gitlab | GITLAB_TOKEN | 180 d |
| observability | GRAFANA_URL, GRAFANA_USER, GRAFANA_PASS, PROM_URL | 365 d |
| omada | OMADA_HOST, OMADA_USER, OMADA_PASS, OMADA_SITE | 365 d |
| github | HARMONIA_GITHUB_TOKEN | 90 d |
| companion | COMPANION_TOKEN | 365 d |
| iam | PROXMOX_IAM_TOKEN_ID, PROXMOX_IAM_TOKEN_SECRET, AUTHENTIK_URL, AUTHENTIK_IAM_TOKEN | 180 d |
| labfeed | LABFEED_TOKEN, SVM_UPDATE_KEY | 365 d |
| dns | TECHNITIUM_MONITOR_TOKEN, TECHNITIUM_API | 365 d |
| uptime-kuma | UPTIME_KUMA_URL, UPTIME_KUMA_USER, UPTIME_KUMA_PASS, UPTIME_KUMA_API_KEY | 365 d |

Rotation age is counted from the import date (the real last-rotation dates were unknown), so the first reminders are honest about that.


## Tokens and policies

- Token role `roundtable-agent`: orphan, renewable, **periodic 90 days** (token auth `max_lease_ttl` tuned to 2160 h). Each script renews its own token at most once a day when it runs, so only a script that does not run for 90 days loses it (Varys warns from 21 days).
- 48 tokens, one per entry script: CT211 `~/.olympus/bao/tokens/<script>.token` (600, directory 700); accessors in `~/.olympus/bao/accessors.json`; the plan the broker uses in `~/.olympus/bao/scopes.json`; per-script broker status (counts, scope names, **names** of drifting keys) in `~/.olympus/bao/status/`.
- Policy `agent-<script>`: `read` on `homelab/data/<scope>` for its scopes only. Verified: netops is refused health, finance, metadata and policies (403).
- Policy `keeper`: list/read `homelab/metadata/*`, read/write `sys/policies/acl/agent-*` **except** `agent-keeper` and `keeper` (explicit deny, verified 403), `auth/token/lookup-accessor`, `auth/token/revoke-accessor`. Plus `agent-keeper` (discord, gitlab) to post.

> [!CAUTION] Limits of the separation
> All Roundtable scripts run as the same Unix user (`hermes-admin` on CT211), so a script *could* read another script's token file, and CT211's `sre_agent_ed25519` key has sudo on CT213. The scopes are least privilege by default and every read is audited, but they are not OS-level isolation. Hard isolation would need one Unix user per agent (not done).

- **Owner login** (2026-10-19): userpass user `hermes`, policy `owner`: create/read/update/delete on `homelab/data/*`, list/read/delete metadata, change its own password; **denied** on policies, auth methods, audit and token creation (verified). Sessions 8 h (max 24 h). Password in the owner's password manager. Set or reset it with `bao_owner_login.sh` on CT213 (password on stdin). UI: https://bao.hrmsmrflrii.xyz → Method **Username**. The root token stays offline for break-glass only.

## How it behaves

- **Order:** OpenBao value first, env file second (`setdefault`). An environment variable already set by systemd still wins over both.
- **OpenBao down or sealed:** 3-second timeout, the script carries on with the env files and records `ok: false` in its status file; Varys reports it.
- **Drift:** if an env file value differs from OpenBao for the same key, OpenBao's value is used and the key **name** is recorded; Varys reports it daily.
- **Varys's schedule:** `keeper.timer` daily 08:15 Asia/Manila: posts to #security-alerts only when something is wrong (sealed or unreachable, denied requests since the last check, root-token use, tokens under 21 days, drift, broker errors); **Mondays** a full report with rotation reminders. Deterministic, no model.

## Runbooks

**Rotate a secret.** Change it at the provider, then in OpenBao (UI at https://bao.hrmsmrflrii.xyz → `homelab/<scope>` → create new version, or `bao kv patch homelab/<scope> KEY=...` with an admin token on CT213) **and** in the CT211 env file while the fallback exists (otherwise Varys reports drift; OpenBao wins). Then `python3 ~/olympus/keeper.py rotated KEY` to restart its reminder.

**Add a new secret or scope.** Add the key to the env file (or OpenBao), add it to a scope in `secret_scopes.json`, deploy, then on CT211: `ssh openbao-lxc 'sudo /usr/local/sbin/openbao-admin-token' | python3 ~/bao_setup.py` (idempotent; writes only changed scopes and creates tokens only for new scripts).

**An agent needs a scope it does not have.** `python3 ~/olympus/keeper.py request --script <script> --scope <scope> --reason "..."` posts an owner approval in #security-alerts. On ✅ Gideon runs `keeper.py grant`, which refuses unless that exact approval is running, then widens only that script's policy. Remove with `keeper.py revoke --script X --scope Y`; `keeper.py revoke --script X` revokes the token.

**Check.** `python3 ~/olympus/keeper.py status` (store, tokens, scopes, rotation, broker, audit) · `keeper.py plan`.

**Unseal by hand.** `ssh openbao-lxc sudo systemctl restart openbao-unseal.service`; check with `curl -s --cacert ~/.olympus/bao/ca.crt https://192.168.40.39:8200/v1/sys/seal-status`.

**Restore from PBS.** Restore CT213, put the unseal key back to `/etc/openbao-unseal/unseal.key` (root 400) from your offline copy, `systemctl restart openbao`.

**Retire the env-file fallback (not done; needs your confirmation).** When Varys has reported no fallback/drift for a while, remove the file reads from `olympus_core.load_env` (or point `_files` at nothing per script) and shred the files. Scripts outside `~/olympus` (`~/doc_agent.py`, `~/orchestrator.sh`, `~/kuma_setup.py`, `sre_agent.py`) still read the env files directly and must move first.

## Files

| Where | What |
|---|---|
| `agents/openbao/` | `bao_install.sh` (CT213 install, idempotent), `bao_setup.py` (import, policies, tokens), `openbao-admin-token`, `bao_owner_login.sh` (owner login, CT213 `/usr/local/sbin/`), `openbao-audit-summary` (+ `.service`/`.timer`), `traefik-openbao.yml` |
| `agents/olympus/` | `bao_broker.py`, `keeper.py`, `secret_scopes.json`, `olympus_core.py` (`load_env`), `agents.json` (Varys), `install.sh` (`keeper` timer row) |
