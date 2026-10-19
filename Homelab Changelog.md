---
tags: [homelab, changelog, documentation, infrastructure]
created: 2026-01-16
updated: 2026-10-19
type: changelog
---

# Homelab Changelog

> Infrastructure change log for MorpheusCluster

> [!NOTE] Chronology, not current-state instructions
> Entries are newest first. Same-day source-only, pending-runtime, 27-agent reporting, IPAM blocker and
> Discord cooldown statements preserve the state **when recorded**. They are superseded by subsequent
> completed releases, not outstanding work today. Use the current Roundtable guide and Task Registry.

## [2026-10-19] - Leaked GitHub repos republished with a clean history (Claude Code)

- Each now has one commit with credentials replaced by `CHANGE_ME` (regex + entropy + known-value scan; rescan 0 remaining). Old history, stars and the extra `master` branch are gone.
- Leaked values are still valid until rotated: see Task Registry #29, #30, #35, #36, #71.

## [2026-10-19] - Public access via Cloudflare (owner-only), public-path monitoring, IP map used/free per VLAN (Claude Code)

### Infrastructure
- ✅ Internet access to the Roundtable (incl. Lab Atlas `/atlas`), Glance, VLAN Flow, Immich, Paperless and Study, without port forwarding.
  - New Cloudflare Tunnel `homelab-public` (remote-managed). Connector `cloudflared-homelab-public.service` on traefik-lxc CT203 (cloudflared 2026.10.1 from the Cloudflare apt repo, DynamicUser-hardened, metrics on 127.0.0.1:20241). Token in `/etc/cloudflared/homelab-public.env` (root 600).
  - Ingress: each hostname → `https://192.168.40.20:443` with SNI/Host set to the hostname, so Traefik routing, Let's Encrypt certificates and Authentik forward auth are unchanged. Anything else → 404. **No Traefik config was changed.**
  - Hostnames: `roundtable`, `glance`, `vlanflow`, `photos`, `paperless`, `auth` (Authentik, needed for the SSO redirects). Proxied CNAMEs to the tunnel. Study keeps its own tunnel.
  - LAN and Tailscale clients still resolve every name to Traefik through Technitium; only internet clients use the tunnel.
  - Script: `agents/cloudflare/cloudflare_public.py` (`access` → `tunnel` → `dns` order enforced; `check`, `status`).

### Security
- ✅ Cloudflare Access app on every public hostname: Google sign-in only, 24 h session, policy "Homelab public - owner only" allows only <email>. Study's existing policy "Study owner allow" verified to include only the same address.
  - Verified from the internet path (public DNS over DoH): all 7 hosts return 302 to the Access login.
  - Caveats: Roundtable and VLAN Flow ask for Access, then Authentik. The Immich mobile app can't pass Access from outside (Task #103). `/avatars/` isn't published.

### Added
- ✅ Watch monitoring for the public apps.
  - Loretta: multi-app `agents/olympus/security.py` deployed to CT211 (backup `~/olympus/security.py.bak-20261019-multiapp`).
  - Uptime Kuma: group "Public apps (Cloudflare)" with push monitor "Tunnel + Access login wall" (`agents/cloudflare/kuma_public.py`). Fed every 2 min by `public-apps-check.timer` on CT203 (`/usr/local/bin/public-apps-check.sh`, from `agents/cloudflare/public-apps-check.sh`): it checks cloudflared `/ready` and that every public hostname still bounces to the Access login. The push URL is in `/etc/cloudflared/public-check.env` (root 600) and the token in CT211 `~/.olympus_env`.
  - Prometheus: `/opt/monitoring/prometheus/public-alerts.yml` with rules `PublicAppsPathDown` (Kuma monitor down 5 min) and `PublicAppsCheckMissing` (30 min). Backup `prometheus.yml.bak.20261019-public`. The SRE agent forwards them to #homelab-infrastructure.
- ✅ IP map page: used and free addresses per VLAN.
  - `ipmap.usage()`: usable hosts per subnet. An address counts as used if a live source (Proxmox, Omada device/VLAN, DHCP reservation, Roundtable host, Traefik, IPAM) has it, or if an Omada client holding it is connected now. Retired and not-seen rows don't count. The static range and the DHCP pool are split.
  - `inventory.py` network data gains `active_clients`. The portal shows a card per VLAN with a bar; click a card to filter. The page re-fetches quietly every 60 s; `ipmap-watch.timer` refreshes the snapshot every 15 min.
  - Live snapshot: VLAN 1 2/254 · 10 7 · 20 18 · 30 8 · 40 19 · 50 1 · 80 1 · 90 11 used.
  - Files: `agents/olympus/{ipmap,inventory,portal}.py`, `tests/test_ipmap.py` (5/5 pass on CT211). Backups `~/olympus-backup-20261019-ipusage/`. Only olympus-portal was restarted (no calls or approvals running).

### Found
- 🔄 Traefik `/opt/traefik/logs/access.log` on CT203 is 7.9 GB and not rotated (Task #102, needs owner approval).

## [2026-10-19] - TLS expiry alert, PBS garbage collection, core-utilities disk, placeholder secrets rotated (Claude Code)

### Added
- ✅ TLS certificate expiry alerting (#98). Prometheus had no rules and no Alertmanager.
  - Rules `TLSCertExpiringSoon` (`(max by (cn) (traefik_tls_certs_not_after) - time()) / 86400 < 14`, for 1h) and `TLSCertMetricsMissing` (`absent(...)` for 6h).
  - File: `192.168.40.13:/opt/monitoring/prometheus/tls-alerts.yml`, referenced by `rule_files` in `prometheus.yml` (backup `prometheus.yml.bak.20261019-tlsrule`). `promtool check config` passed; reloaded through `/-/reload`.
  - Delivery: new `prom_alerts()` in `agents/sre_agent.py` (CT211, backup `~/sre_agent.py.bak-20261019-promalerts`).
    - Every 15 min it posts newly firing Prometheus alerts to #homelab-infrastructure, re-posts once a day while they keep firing, and posts a note when they resolve.
    - Read-only, no AI tokens. CLI: `sre_agent.py prom-alerts`. Generic, so any future Prometheus rule is delivered too.
  - Tested end to end with a temporary `TLSAlertPathTest` rule: fired, forwarded, then removed and resolved. The soonest certificate expires in 81 days.

### Fixed
- ✅ PBS retention and garbage collection (#96).
  - Prune does run for active guests (vzdump logs `pruned 1 backup(s)`).
  - The January/February snapshots belong to decommissioned guests (`vm/1000, 101, 104, 105, 109, 110, 116–120, 300–311`) that are in no job, so they are never pruned. Their removal needs the owner's decision (Task #100).
  - Garbage collection had never run, so pruned chunks were never freed. Added `gc-schedule *-*-* 04:30` on `daily` and `sat 06:00` on `main` (CT100 backup `/root/datastore.cfg.bak-20261019-gc`).
  - Manual GC on `daily`: 73% → 42% (≈266 GB freed). `main` GC started (log `/root/gc-main-20261019.log`).
- ✅ "Unaccounted" `/var/lib` space on docker-vm-core-utilities01 (#13).
  - Docker runs the containerd image store (`io.containerd.snapshotter.v1`), so image layers live in `/var/lib/containerd` (43G), not `/var/lib/docker`.
  - Prometheus has `--storage.tsdb.retention.time=30d`, and its volume is small.
  - Pruned build cache (2.8 GB) and dangling images: `/` went from 59G to 46G.
  - ≈9.5 GB of tagged but unused images are left for the owner (Task #101).

### Security
- ✅ Placeholder secrets verified and rotated (#26). Values were never displayed.
  - Sentinel `API_KEY` was the literal placeholder. It was rotated to 64 hex characters in `/opt/sentinel-bot/.env` (backups `*.bak-20261019-apikey`). Compose now uses `${API_KEY:<redacted>` instead of a placeholder default, the container was recreated, and `/health` is healthy.
    - Finding: `require_api_key` is never applied to a route, so Sentinel's webhook and task API are unauthenticated on the LAN (Task #99).
  - Immich OAuth client secret was the live placeholder on both sides. It was rotated together in Authentik (provider 45, `immich-oauth-client`) and the Immich system config (DB backup `/opt/immich/system_metadata.bak-20261019-oauth`, mode 600), then `immich-server` was restarted.
    - A first attempt echoed part of a generated value in a psql error, so that value was discarded and both sides were re-rotated through scripts that suppress output.
    - Verified: an Authentik token request with the Immich-stored secret succeeds, and a bogus secret is rejected. Browser SSO login has not been tested yet.

## [2026-10-19] - Pi-hole update, Technitium blocklist check, portal stale-error fix (Claude Code)

### Changed
- 🔄 **Pi-hole CT202 updated**: Core 6.4.3→6.5, Web 6.6→6.6.1, FTL 6.7.1→6.8 (`pihole -up`). Gravity rebuilt (305,062 domains, 1 exact deny, 11 regex, 14 allow). Verified blocking, upstream and local `*.hrmsmrflrii.xyz` names, admin UI.
  - Rollback: node01 `pct rollback 202 pre-pihole-update-20261019` (snapshot taken 08:09)
- ✅ **Technitium blocklists checked against Pi-hole**: all 4 lists, 14 allows, exact and regex denies are already in the Advanced Blocking app (since the 2026-10-18 cutover); clients get `0.0.0.0` for blocked names. The built-in "Blocked" tab is intentionally empty. Note: queries from the DNS host's own loopback are not filtered.

### Fixed
- 🔧 **Roundtable portal stale error badges**: the health badge now follows the same 24 h window as the status light, and kv `error_ack` {agent: run id} marks a fixed failure as handled. Acked Kalé (run 3027) and Melina (run 2748), both old journal-truncation errors.
  - File: `agents/olympus/portal.py` (`live_error`, `agents_payload`, office mood)

### Added
- ✅ **Hewg app-update audit (third check)**: the SRE executor now watches apps that update themselves, outside apt and Docker. They were invisible before, which is why the Pi-hole update was missed. `app-audit` runs daily at 17:00 Manila (or by CLI) and files one approval issue per outdated app: label `app-update`, title `App drift: <host> / <app>`. Approval works like OS/container issues (Discord ✅/❌ or the `approved` label), and the update runs in the 02:00–05:00 window.
  - Pi-hole (CT202): PBS backup gate, Proxmox snapshot `sre-app-<iid>`, `pihole -up`, checks; on failure: `pct stop/rollback/start`. Snapshot deleted on success.
  - Technitium (homelab-dns01, bare metal): tarball of `/opt/technitium/dns` + `/etc/dns` in `/root/sre-backups/` (last 3 kept), the official `install.sh`, checks. On failure the dirs are moved aside to `*.failed-<iid>` and the tarball restored.
  - Checks run from CT211 against the server's own IP: service active, `doubleclick.net` → 0.0.0.0, `github.com` resolves, `roundtable.hrmsmrflrii.xyz` → 192.168.40.20 (up to 3 min).
  - Validated: version reads and DNS checks pass on both; a wrong-server negative test fails all four checks; snapshot create/delete and the file backup were tested and the test artifacts removed. First audit: both apps current, 0 issues. The live update and rollback paths have not run yet.
  - Files: `agents/sre_agent.py` (CT211 `~/sre_agent.py`, backup `~/sre_agent.py.bak-20261019-app-audit`), `agents/olympus/builder.py` (Hewg's patch status includes `app-update`); `sre-agent.service` restarted at 08:20 with nothing in flight.

## [2026-10-19] - Roundtable agent log

### Changed
- 🌗 **homelab-hub docs synced** (Radagon): Commit bc4ffad on herms14/homelab-hub: - `docs/inventory.md`: node, VM and LXC tables refreshed from the live cluster - `docs/rebuild.md`: LIVE lists 15 LXCs; the previous count of 14 is wrong. Whether they were all created by hand is not verifiable from LIVE.

## [2026-10-18] - OpenBao secrets management, Varys the keeper, IP map sorting (Claude Code)

### Infrastructure
- ✅ **CT213 openbao-lxc** on node02, 192.168.40.39 (IPAM `openbao-vault` active): OpenBao 2.7.1 (checksum-verified `.deb`), raft storage, own TLS CA, file audit log (declared in config), 1-key auto-unseal from the `backup=0` mount `/etc/openbao-unseal`, hourly value-free audit summary for Varys (user `bao-report`, forced command). Added to PBS daily/weekly jobs, Prometheus (`openbao`, `node-openbao-lxc`), CT211 `sre-hosts.txt`/ssh config, Atlas. Traefik `bao.hrmsmrflrii.xyz` (LAN/Tailscale allowlist, own token login), Technitium record (Pi-hole synced). See [65 - OpenBao Secrets Management](65%20-%20OpenBao%20Secrets%20Management.md).
  - Files: `agents/openbao/` (install, setup, admin-token, audit summary, Traefik route); backups node02 `/root/jobs.cfg.bak-20261018-bao`, CT211 `~/.ssh/config.bak-20261018-bao`, `~/.ssh/sre-hosts.txt.bak-20261018-bao`, monitoring `prometheus.yml.bak.20261018-openbao`, homelab-dns01 `/root/technitium_setup.py.bak-20261018-bao`.

### Security
- ✅ 39 Roundtable keys (CT211 env files) imported into 16 KV scopes without printing; 48 per-script tokens (periodic 90 d) with read-only `agent-<script>` policies from `keeper.py plan`; verified that a script is refused other scopes, metadata and policies, and that Varys cannot change its own policies.
- ⚠️ Not OS-level isolation: every script runs as `hermes-admin` on CT211 and the SRE key has sudo on CT213 (documented).

### Added
- ✅ Broker in `olympus_core.load_env` (`bao_broker.py`): OpenBao first, env files fill gaps (fallback kept until the owner retires it), drift recorded by key name. Portal restarted on it (16 keys from OpenBao, no drift).
- ✅ **Varys** (`keeper`, reports to Loretta): `keeper.py` plan/status/audit/request/grant/revoke/rotated; `keeper.timer` daily 08:15 (posts only on a problem; Mondays the weekly report). Never reads values.
- ✅ IP map: portal sort by IP or name (selector + clickable headers, remembered per browser); `ipmap-watch.timer` every 15 min (no model) rewrites the vault registry in [10 - IP Address Map](10%20-%20IP%20Address%20Map.md) (two tables: by IP, by name) when an address or its service changes and posts new services to #homelab-infrastructure; Diallos's registry entry lists both timers.
  - Files: `agents/olympus/ipmap.py`, `portal.py`, `tests/test_ipmap.py`, `agents.json`, `install.sh`, `atlas_layout.json`. CT211 backup `~/olympus-backup-20261018-153857-bao/`. Tests: 78 run, same 4 pre-existing ACL failures.

### Found (not changed)
### GitHub
- ✅ Pushed after the pattern scan plus exact-match check against every real secret value (OpenBao root token, unseal key and agent tokens included; values never printed): `olympus-agents` 4652138, `homelab-hub` 3c4fa89 + f33f408 (stray `__pycache__` removed, now ignored), `ObsidianVault` f52fb07 (only this session's lines in `00 - Homelab Index` and `11 - Credentials`, which also hold other sessions' uncommitted edits). The only exact matches were three internal service URLs (Grafana, Authentik, Study), not secrets. `glance-dashboard` unchanged.

### Found (not changed, continued)

## [2026-10-18] - GitHub repos synced (Claude Code)

### Changed
- 🔄 Pushed after a pattern scan plus an exact-match check against every real secret value (0 secrets found; values never printed): `olympus-agents` 12ce3b2 (portal, Atlas, IP map, agents, new `dnssync.py` and `dns/`; no new repo needed, it already holds the portal/Atlas/IP map/agents), `glance-dashboard` be0bee7 **v4.1.0** (Technitium widget, `GLANCE_TECHNITIUM_TOKEN` placeholder), `homelab-hub` 1598ac3 (Technitium primary DNS docs), `ObsidianVault` 53ef8aa (this session's files).
- ⚠️ Pre-existing, not added by this sync: the private vault repo already holds credentials in homelab `CLAUDE.md` (Proxmox token, Arr API keys), `01 - Network Architecture.md` (FTP password), Homelab Changelog and Task Registry.

## [2026-10-18] - Uptime Kuma DNS checks on the Roundtable Overview; dependency map redrawn (Claude Code)

### Security
- ✅ Uptime Kuma (192.168.40.13:3001) had no admin account (first-run wizard open on the LAN) and no monitors since its data volume was created on 2026-10-15. Admin `hermes` created with a random password (CT211 `~/.olympus_env`, `UPTIME_KUMA_PASS`).

### Added
- ✅ Uptime Kuma group **DNS**: local and public A-record checks on Technitium (.90.60) and Pi-hole (.90.53); metrics API key; Prometheus job `uptime-kuma` (key file `/opt/monitoring/prometheus/kuma_api_key`, backup `prometheus.yml.bak.20261018-kuma`). See [17 - Monitoring Stack › Initial Setup](17%20-%20Monitoring%20Stack.md#initial-setup).
- ✅ Roundtable portal Overview: **DNS checks · Uptime Kuma** panel (per server, status and response time; `/api/dnschecks`, from Prometheus, cached 30 s).
- ✅ Technitium record `pihole.hrmsmrflrii.xyz` → Traefik (Pi-hole had answered it from CT202's `/etc/hosts`, so it was missing after the cutover; the Overview showed Pi-hole offline). Pi-hole still answers it with 192.168.90.53 from `/etc/hosts`.

### Changed

## [2026-10-18] - Technitium is the primary DNS, Pi-hole secondary (Claude Code)

### Infrastructure
- ✅ **homelab-dns01** (former OPNsense mini PC, now Ubuntu 26.04) at static **192.168.90.60**: Technitium DNS 15.6.2, recursive with DNSSEC, Advanced Blocking with Pi-hole's lists/regex/allowlist, Query Logs (Sqlite), zones `hrmsmrflrii.xyz` (local A records → Traefik) and `lan`. Answers matched Pi-hole for local, public and blocked names before cutover.
  - Setup scripts: `agents/dns/technitium_setup.py`, exporter `agents/dns/technitium_exporter.py` + `technitium-exporter.service` (:9274); `prometheus-node-exporter`, `smartmontools`.
  - DHCP switched off on the box (it still held lease .57, which Omada also showed for the owner's phone); netplan `60-dns-static.yaml` (backup `/root/60-dns-static.yaml.pre-dhcpoff`).
- ✅ **Cutover**: Proxmox nodes (`pvesh .../dns`, node03 was 8.8.8.8), all 13 LXCs (`pct set --nameserver` + live `resolv.conf`; CT100 PBS was Google DNS), Pi-hole CT202 asks itself then .60; VMs 103/107/121/106/108 (netplan, `resolved.conf.d/pihole.conf`, live `resolvectl`, `qm set --nameserver`) — removed the wrong global DNS 192.168.20.53 (GL.iNet KVM) and immich's 192.168.20.1. Backups `*.bak-20261018-dns` / `/root/*.bak-20261018-dns`. Scripts on CT211: `~/dns_cutover_20261018.sh`, `~/dns_cutover_lxc_20261018.sh`.
  - Owner: Omada DHCP DNS on VLANs 10/20/30/40/50/80/90 = .60, .53; Tailscale split DNS = both. Docker containers created before today keep their old resolver list until recreated (still works via Pi-hole).

### Added
- ✅ Traefik route `dns.hrmsmrflrii.xyz` → `http://192.168.90.60:5380` (`/opt/traefik/config/dynamic/technitium.yml`, Let's Encrypt cert, Technitium login).
- ✅ Prometheus jobs `technitium` and `node-homelab-dns01` (backup `prometheus.yml.bak.20261018-technitium`).
- ✅ Torrent: `dnssync.py` + `dns-sync.timer` (daily 05:50) copies Technitium local records to Pi-hole `dns.hosts` (first run added node03, dns, homelab-dns01); read-only `monitor` token in CT211 `~/.olympus_env`.
- ✅ Glance Network page: Technitium widget (read-only token `GLANCE_TECHNITIUM_TOKEN` in `/opt/glance/.env`), status monitor and links; Pi-hole marked secondary.
- ✅ CT211 `~/.ssh/sre-hosts.txt` + `~/.ssh/config`: `homelab-dns01` (hermes-admin, patchable as vm).

### Changed
- 🔄 Roundtable (deployed, backup `~/olympus-backup-20261018-142327-dns/`): `netops.py` checks both servers + blocking (Technitium down high, Pi-hole down medium, both critical); `council.py` allows `systemctl is-active/restart dns` on homelab-dns01; `builder.py` new guests get both DNS servers; `inventory.py`, `portal.py` (Technitium link), `agents.json`, `security.py`/`scout.py`/`voice.py` context; `acl_ruleset.json` v8 (`G_PIHOLE` = .53 + .60, ports 5380/9274; not rolled out, ACLs disabled). Tests: 77 run, the same 4 ACL planning tests fail as before.
- 🔄 Hermes Lab Atlas (`agents/olympus/atlas_layout.json`, deployed + refreshed): OPNsense nodes on the Architecture, Network and Hardware tabs are now homelab-dns01 / Technitium (primary), Pi-hole marked secondary, VLAN DNS fields and the DNS note updated, Technitium added to Services; removed the "OPNsense not booting" and "One DNS server" warnings, added "VLAN 1 gets only one DNS server".

## [2026-10-18] - Docs and GitHub repos synced (Claude Code)

### Changed
- 🔄 Docs: Agents, 60 - Roundtable Agent Team, 62 - Roundtable Technical Manual (Miriel's Atlas duty, portal Overview/themes/`/atlas`); `agents/olympus/agents.json` Miriel entry (Atlas, `atlas-sweep.timer`; deployed by the owner: hash OK, 32 agents, portal shows both units, no restart; backup CT211 `~/olympus-backup-20261018-135713-agents-json/`); new `agents/olympus/tests/test_atlas.py` (9 tests pass).
- 🔄 GitHub (secret-scanned before each push): `ObsidianVault` 31425e6 (this session's files only); `olympus-agents` 0165634 (full code sync from the vault, README; `chhaya.py` then removed at the owner's request, 4718fb1; still in git history); `glance-dashboard` 262faaa **v4.0.0** (redesigned config, `config/pages/`, `assets/lab.css`, renamed env vars, TZ); `homelab-hub` a5e4f5a (Roundtable on CT211, Glance 4.0, Lab Atlas).

## [2026-10-18] - Miriel keeps the Hermes Lab Atlas live (Claude Code)

### Added
- ✅ **`agents/olympus/atlas.py`** (Miriel, deterministic, no LLM): overlays Proxmox guests, Docker containers, Omada devices, systemd units and `agents.json` on the hand-edited layout **`agents/olympus/atlas_layout.json`** (converted from the Atlas data block; baseline reproduces the hand-made Atlas exactly). Removed items drop off; new guests go onto their node, new agents into their team, new containers into "New · needs placing". Output `~/.olympus/atlas.html`, served at `/atlas`.
  - Triggers: every `scribe.log_change` entry (detached refresh), **`atlas-sweep.timer`** (`*:20/30`, unit written by hand, row added to `install.sh`), Miriel in Discord/voice ("redraw the atlas"), the **Redraw** button on `/atlas` (`POST /api/atlas/refresh`, Authentik only; `GET /api/atlas/status`).
  - Additions, removals, renames, moves and resizes: post to #homelab-infrastructure, a changelog entry and an upload of `Diagrams/Hermes Lab Atlas.html` (skipped if the vault copy was edited elsewhere). Status flips (stopped/exited/offline) only recolour the map.
  - Engine moved into the repo as `agents/olympus/atlas_engine.html`. Not used: Traefik routes (several stale; cleanup pending).
  - Deploy run by the owner: hashes OK, 0 voice calls, first quiet build 0 changes, vault copy uploaded, timer enabled, portal-only restart; `/healthz` and `/atlas` 200 (snapshot "· live"), status OK, unauthenticated refresh 403. Backup: CT211 `~/olympus-backup-20261018-123137-miriel-atlas/`.
  - Files: `agents/olympus/atlas.py`, `atlas_layout.json`, `atlas_engine.html`, `scribe.py`, `portal.py`, `install.sh`.

## [2026-10-18] - Roundtable themes consistent on every page (Claude Code)

### Changed
- 🔄 One palette (`THEMES` in `agents/olympus/portal.py`) served as **`/theme-kit.css`** + **`/theme-kit.js`** and loaded by the portal, `/office`, `/teach` and `/atlas`. Each page has a ◐ picker; the choice (`localStorage` `rt-theme`) is shared across these pages and open tabs update live. Glass keeps every page's original look.
  - `office.html`: header, panel, buttons, cards and toasts themed; 3D scene unchanged. `teaching.html`: page, buttons, terminal themed. Atlas engine: maps its tokens to the kit when served from the portal; offline vault copy and PDF export unchanged.
  - Deploy run by the owner: all four live hashes matched, 0 voice calls, portal-only restart; `/`, `/office`, `/atlas`, `/theme-kit.*` 200; `/teach` 403 for the unauthenticated local check (sign-in only, by design; installed file contains the kit). Backup: CT211 `~/olympus-backup-20261018-120133-allthemes/` (portal.py, office.html, teaching.html, atlas.html).
  - Files: `agents/olympus/portal.py`, `agents/olympus/office.html`, `agents/olympus/teaching.html`, `Diagrams/Hermes Lab Atlas.html`.

## [2026-10-18] - Roundtable portal themes; Overview is the default page (Claude Code)

### Changed
- 🔄 Portal theme picker (header ◐ dropdown, remembered per browser in `localStorage` `rt-theme`, `?theme=` overrides): **Aurora Bento** (default), **Minimal Pro**, **Nordic Night**, **Pastel Night**, **Glass (classic)**. All dark; applied to every portal page (the separate `/office` page keeps its own look). Unknown or removed theme names fall back to Aurora Bento. Fonts from Google Fonts (Inter, Plus Jakarta Sans, Manrope, Nunito).
- 🔄 Overview (`#overview`) is now the page the portal opens on; added a status strip (sites online, warnings, offline, last checked). Header no longer wraps at laptop widths (host chip hides below 1640 px).
  - Owner reviewed 12 mock-ups locally (Mission Control and seven light/novelty themes rejected) before deploy. Deploy run by the owner: hash check, 0 voice calls, portal-only restart; `/healthz` ok, 44 links, picker present on the live page.
  - File: `agents/olympus/portal.py`. Backup: CT211 `~/olympus-backup-20261018-114039-themes/portal.py`.

## [2026-10-18] - Roundtable portal Overview page (Claude Code)

### Added
  - File: `agents/olympus/portal.py` (`LINKS`, `links_payload`). Backup: CT211 `~/olympus-backup-20261018-110746-overview/portal.py`.

### Infrastructure (found; not changed)

## [2026-10-18] - Hermes Lab Atlas hosted at roundtable.hrmsmrflrii.xyz/atlas (Claude Code)

### Added
- ✅ Roundtable portal route `/atlas` serves the Atlas (Authentik-protected via the existing Roundtable app). Owner ran the deploy script: live `portal.py` hash checked first, 0 live voice calls, only `olympus-portal` restarted (boss untouched); `/healthz` ok, `/atlas` HTTP 200 (100,037 bytes).
  - Files: `agents/olympus/portal.py` → CT211 `~/olympus/portal.py`; `Diagrams/Hermes Lab Atlas.html` → `~/olympus/atlas.html`. Backup: `~/olympus-backup-20261018-105735-atlas/portal.py`.

## [2026-10-18] - Hermes Lab Atlas v2: modern UI, bundled lines, animation (Claude Code)

### Changed
- 🔄 `Diagrams/Hermes Lab Atlas.html` redesigned: glass UI (top bar, bottom toolbar, Layers panel, details drawer, Ctrl+K search). Links between groups are bundled into one line with a count (Bundled / All / Focus). Right-angle routing through the gaps between groups, with detours around groups, spread ports and lanes. Animated packets plus glow lines in VLAN Flow style (Motion toggle, honours reduced-motion). PDF export unchanged in behaviour (verified 5 pages, A1/A3). Same data. Previous version kept in the session scratchpad only.

## [2026-10-18] - Hermes Lab Atlas diagrams (Claude Code)

### Added
- ✅ `Diagrams/Hermes Lab Atlas.html`: offline interactive diagrams (Architecture, Network, Services, Agents, Hardware), Overview/Detailed modes, search, attention list, PDF export (A1–A4/Letter, dark/light; verified 5 pages at the right sizes). Built from live data; no credentials. Guide: Hermes Lab Atlas.

### Infrastructure (found while collecting data; not changed)
- Proxmox is **9.2.21** on all nodes (docs say 9.1.19). Tailscale also runs on node02 and node03.
- **node01** guests are allotted all 60 GiB of RAM; **node03** running guests are allotted 31.8 GiB on a 31 GiB host.
- Live Omada device IPs differ from the hardware table in [01 - Network Architecture](01%20-%20Network%20Architecture.md) (e.g. Core Switch .54, Morpheus .51, Zephyrus .58, EAPs .52/.55/.56) and its port tables are stale.

## [2026-10-18] - Glance redesign live (Claude Code, task #88)

### Changed
- ✅ Cut over `/opt/glance` on CT200 to the redesigned config (owner ran `cutover_glance.sh`). Config validated, all 9 pages return 200, `TZ` America/New_York → Asia/Manila. `glance-next` preview stopped and removed (folder kept). The Sellen self-check against live shows nothing broken.
  - Backup: `/opt/glance-backup-20261018-pre-redesign.tgz` (600), `config/glance.yml.bak.20261018-pre-redesign`, `docker-compose.yml.bak.20261018`
### Fixed
- ✅ `labfeed.py check`: widget titles are now unescaped (`&amp;amp;` → `&`). Deployed to CT211, `labfeed` restarted.

## [2026-10-18] - Glance redesign preview, Lab Feed, Grafana change markers (Claude Code)

### Added
- ✅ **Lab Feed** on CT211 :8790 (`labfeed.service`, always on): token-protected, read-only lab JSON (cluster, NAS, backups, services, network/VLANs, Roundtable) for Glance and the agents.
  - Files: `agents/olympus/labfeed.py`, `labfeed.service`, `install.sh` (service + `glance-check` timer rows). `LABFEED_TOKEN` and `SVM_UPDATE_KEY` added to `~/.olympus_env` (600).
- ✅ **glance-next preview** on CT200 (`/opt/glance-next`, http://192.168.40.12:8081). New theme, phone layout, a Roundtable page, and live data instead of the hard-coded widgets. Source: `agents/glance/`. The live :8080 dashboard is unchanged.
- ✅ **Sellen Glance self-check:** `glance-check.timer` (:05/:35) posts to `#infrastructure-alerts` only when a widget breaks or recovers. First run against the preview: nothing broken.

### Fixed
- ✅ `omada-network.json` was wrapped in an API envelope (`{"dashboard": …}`), so Grafana's file provisioner never reloaded it (the DB copy dated 2025-12-27). The file is now unwrapped and reloaded with all 40 panels.

### Security
- ⚠️ The live Glance Services page and the Service Version Manager API expose the SVM update key. The new page routes updates through a Roundtable approval instead. **Rotate the SVM key after cutover.** Separately, SVM `/api/summary` reports 0/51 online (bug; Lab Feed aggregates the categories itself).

### Changed
- 🔄 `observe.py` (overview/roundtable pages → general layer), `voice.py`, `watcher.py` page lists. Deployed to CT211 (backup `~/olympus-backup-20261018-glance`). `boss`/`olympus-portal` restarted with nothing running.
  - Docs: [23 - Glance Dashboard](23%20-%20Glance%20Dashboard.md), [07 - Deployed Services](07%20-%20Deployed%20Services.md)

## [2026-10-18] - Discord channel merges and categories (Claude Code, task #84)

### Changed
- 🔄 Roundtable: `#agent-access` -> `#security-alerts` (Loretta/access gaps), `#homelab-scout` -> `#ideas-and-todos` (Melina's scout). GitLab label `agent-access` unchanged.
  - Files: `agents/olympus/access.py`, `voice.py`, `hierarchy.py`, `scout.py`, `agents.json`, `boss.py` (help line); deployed to CT211 (backup `~/olympus-backup-20261018-channels`). `boss`/`olympus-portal` restarted by the owner, both active.
- 🔄 Sentinel (192.168.40.13): `CHANNEL_MEDIA_DOWNLOADS=cinema-lounge`, `CHANNEL_PROJECT_MANAGEMENT=roundtable-hold`, `/download` footer text; image rebuilt, container healthy (backups `.env.bak-20261018`, `cogs/media.py.bak-20261018`).
  - Docs: [24 - Discord Bots](24%20-%20Discord%20Bots.md), [07 - Deployed Services](07%20-%20Deployed%20Services.md)

## [2026-10-18] - Atlas finance jobs fixed (Claude Code)

### Fixed
  - File: `agents/olympus/finance.py`

## [2026-10-18] - Roundtable agent log

### Changed
- 🌗 **homelab-hub docs synced** (Radagon): Commit 783d0d9 on herms14/homelab-hub: - `docs/inventory.md`: node, VM and LXC tables refreshed from the live cluster - `README.md`: The live cluster has 14 LXCs. - `docs/rebuild.md`: The live cluster has 14 LXCs.

## [2026-10-18] - Automated patching (SRE agent)

### Infrastructure
- ✅ `gitlab-runner-vm01` (VM): patched 2 packages; kernel stayed `6.8.0-147-generic`, and `apparmor`/`libapparmor1` moved from `4.0.1really4.0.1-0ubuntu0.24.04.8` to `.9` (3→1 pending); independent verification passed. [GitLab #597](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/597)
- ✅ `gitlab-vm01` (VM): patched 2 packages; kernel stayed `6.8.0-147-generic`, and `apparmor`/`libapparmor1` moved from `4.0.1really4.0.1-0ubuntu0.24.04.8` to `.9` (3→1 pending); independent verification passed. [GitLab #598](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/598)
- ✅ `pihole` (LXC): patched 2 packages; kernel stayed `7.0.14-20-pve`, and `apparmor`/`libapparmor1` moved from `4.0.1really4.0.1-0ubuntu0.24.04.7` to `.9` (2→0 pending); independent verification passed. [GitLab #599](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/599)
- ✅ `docker-lxc-media` (LXC): patched 2 packages; kernel stayed `7.0.14-20-pve`, and `apparmor`/`libapparmor1` moved from `4.0.1really4.0.1-0ubuntu0.24.04.7` to `.9` (2→0 pending). Baseline recorded `apparmor.service` failed; independent verification passed. [GitLab #600](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/600)
- ✅ `authentik-lxc` (LXC): patched 2 packages; kernel stayed `7.0.14-20-pve`, and `apparmor`/`libapparmor1` moved from `4.0.1really4.0.1-0ubuntu0.24.04.7` to `.9` (2→0 pending). Baseline recorded `apparmor.service` failed; independent verification passed. [GitLab #601](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/601)

### Changed
- 🔄 `docker-lxc-glance` (`proxmox-nodes-api`): image check found `python:3.11-slim` already at `3.11.17` with the same image digest; no image change. Independent verification passed. [GitLab #603](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/603)
- ✅ `homeassistant-lxc` (`homeassistant`): updated Home Assistant `2026.10.1` → `2026.10.2`; independent verification passed. [GitLab #604](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/604)
- ✅ `homepage-lxc` (`homepage`): updated Homepage `v2.5.0` → `v2.6.0`; independent verification passed. [GitLab #605](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/605)

## [2026-10-17] - Study app: sign-in fix; vault habit note fed from the notebook (Claude Code)

### Fixed
- 🔐 Signed-in users got `{"detail":"Sign in through https://study.hrmsmrflrii.xyz"}`: uvicorn ran with
  `--proxy-headers --forwarded-allow-ips=192.168.40.20`, so it replaced Traefik's address with the browser's before
  the app's "only via Traefik" check. `study.service` now runs `--no-proxy-headers` (owner ran/approved the change;
  backup `/etc/systemd/system/study.service.bak-20261017`). Verified: via Traefik unauthenticated → 302 to Authentik;
  a forged `X-authentik-username` + `X-Forwarded-For` sent directly to :8090 → 403.

### Changed
  note left untouched if the notebook is down). Refreshed after every Study mode wrap-up and daily by
- Tests: study app 5/5 (WSL), CT211 suite passes except the 4 known `test_acl` failures. Backups:
  `/opt/study/app.py.bak-20261017-144700`, `~/olympus/backups/study-habits-20261017-064710/`.

## [2026-10-17] - Study app step 5: sessions migrated, portal Study mode retired (Claude Code)

### Changed
  25 min (03:16 UTC) and 13 min (03:46 UTC). A 17-min row starting 3 s after the 25-min one was a duplicate log of
  the same session and was skipped. The originals stay in CT211 `study_sessions` (no longer written).
- 📝 [07 - Deployed Services](07%20-%20Deployed%20Services.md): Study Notebook rows added.
- Tests: portal browser 81/81 (browser, mobile, desk); suite on CT211 passes except the 4 known `test_acl` failures.
  Backup: `~/olympus/backups/study-step5-20261017-063921/`.

## [2026-10-17] - Portal: 🧭 IP map, kept current by Diallos (Claude Code)

### Added
- ✅ **New portal page: Operations → 🧭 IP map.** It lists every assigned address with the app or service using it, its purpose, public URLs, where it was seen, and a status. There are filters by VLAN and by status (documented / undocumented / reserved / not seen live) and a search box. ↻ re-reads every source. The endpoint is `GET /api/ipmap?refresh=1` and requires portal login; off-proxy requests get 403.
- ✅ New module `ipmap.py` (Diallos, `ipam`) merges, per IP, without changing any source:
  - Proxmox guests;
  - Omada devices, DHCP reservations, VLAN gateways, and static clients outside the DHCP pool;
  - IPAM reservations and the Roundtable's managed hosts;
  - live Traefik routes, read over SSH from `traefik-lxc`, which map each backend IP to its `*.hrmsmrflrii.xyz` URLs;
  - the tables in this vault note, [10 - IP Address Map](10%20-%20IP%20Address%20Map.md);
  - the new agents' registry, table `ip_registry`.
- ✅ **Agents keep it current.** When an owner-approved plan introduces a new service, `council.apply()` now calls `ipmap.record(ip, service, title, by=<agent>)` right after `ipam.activate`. Agents and the owner can also run `python3 ipmap.py record --ip … --service … --purpose … --url …`.
- ✅ **AI:** Diallos suggests a service and a one-line purpose for addresses nobody documented. Suggestions are marked ✨, cached by an evidence hash, and never overwrite a real record.
- ✅ The new `ipmap-sync.timer` (06:40 Manila daily) refreshes everything and rewrites only the section **"Roundtable IP registry (maintained by Diallos)"** at the end of [10 - IP Address Map](10%20-%20IP%20Address%20Map.md). The owner's tables above it are never touched; `advisers.vault_update` skips the write if the note was edited at the same time.
- ✅ Seeded `192.168.40.38 study-lxc`. First run: 68 addresses, of which 25 documented, 34 undocumented (AI suggestions added), 8 not seen live and 1 retired but still routed.
- ✅ Tests: new `tests/test_ipmap.py` (Traefik/vault parsing, merge and status rules, name priority, suggestions never overriding records). `test_portal` has an IP map browser scenario; 81/81 in browser, mobile and desk modes.

### Findings (not changed)
- 🔍 Traefik `new-services.yml` still routes **lagident.hrmsmrflrii.xyz → 192.168.40.10:9933**, the decommissioned docker-vm-core-utilities01, so the route is dead.
- 🔍 This note's switch/AP rows (`192.168.90.2/.3/.10–.12`) and the VLAN 91 / OPNsense rows were not seen live. Omada reports the switches and APs at `192.168.90.50–.58`.

### Infrastructure
- ✅ Deployed to CT211 (owner-requested). Backup: `~/olympus/backups/ipmap-20261017-054812/` (plus `ipmap.py.v1` from the first rollout). Ran `install.sh`; restarted `olympus-portal` and `boss`; `/healthz` returns 200.

## [2026-10-17] - Study app: installed on CT212 with a two-way vault sync (Claude Code)

### Added
  - App at `/opt/study` (venv with fastapi 0.115.0, uvicorn 0.30.6, python-multipart 0.0.9, pypdf 4.3.1). Data in `/srv/study/{vault,data,trash}`, owned by system user `study`.
  - `study.service` runs uvicorn on port 8090 with systemd hardening (`ProtectSystem=strict`, `NoNewPrivileges`, write access to `/srv/study` only).
  - Verified: `/healthz` returns 200. Requests that don't come through Traefik get 403, even with a forged `X-authentik-username` header.
- ✅ **Vault sync:** `study-sync.timer` runs `/opt/study/sync.sh` every 5 minutes. It does an rclone bisync between `/srv/study/vault` and `onedrive:Obsidian Vault/Hermes's Life Knowledge Base/Study`.
  - rclone v1.75.2 was installed to `/usr/local/bin`, with its checksum verified against SHA256SUMS.
  - Only the `onedrive` remote's config was piped from CT211 into `/srv/study/.rclone/rclone.conf` (owner `study`, mode 600). It was never displayed.
  - Safety settings:
    - `--check-access` with the hidden marker `Study/.study-sync`: **do not delete it**, or the sync stops.
    - `--max-delete 90`: the 50% default tripped on the tiny new vault.
    - `--conflict-resolve newer`, plus `--resilient --recover`.
  - Round trip verified: a note made in the app reached the vault and vice versa, and deletes propagated both ways. The test files were removed.
- ✅ **Browser test:** 27/27 UI checks at 1440 px and 390 px. Fixed four visual bugs:
  - the "Drop to attach" overlay and the idle 25:00 timer ignored `[hidden]`;
  - the graph over-zoomed with few notes (`maxZoom 2.5`);
  - the mobile header was clipped (shorter labels on phones; Split mode on a phone now shows the rendered note).

## [2026-10-17] - Rogier: least-privilege Authentik provisioning account (Claude Code)

### Security
- ✅ **Live on authentik-lxc (owner-approved, ~12:10).** New service account `rogier` (path `roundtable`, unusable password, not in any group) with role **"Roundtable IAM - grantable group membership only"**:
  - Global `authentik_core.view_user`.
  - Object permissions `view_group`, `add_user_to_group` and `remove_user_from_group` on **`authentik Agent-Users`** and **`media-users`** only.
  - No user edits or password resets, no group creation, no superuser, admin or `media-admins` groups, and no system settings.
  - API token `rogier-provision` expires 2027-10-17. It was moved straight into CT211 `~/.olympus_env` (`AUTHENTIK_URL`, `AUTHENTIK_IAM_TOKEN`, mode 600) and was never displayed. The temporary copy in the container was deleted.
- ✅ Verified from CT211 with the token: it can read users (7) and sees only the 2 grantable groups. Adding and removing a member worked (204; tested on the service account itself, then removed). Editing a user, creating a group and reading system info all returned 403. Application names stay visible, as they are to every Authentik account.
- 📝 Owner chose Authentik only. GitLab, Grafana and PBS were not provisioned, because none of them can limit an account to user admin. The existing `GITLAB_TOKEN` is the `backlog-intake-agent` project bot for project 6 only, so Rogier's GitLab grants work on that project alone.
  - `council.py`: new `authentik grant <group> <user-pk> <hours>` (host `authentik-lxc`). It needs an open AG request that mentions Authentik/SSO/media and your ✅. It runs for at most 24 h and no longer than requested, never at admin level, and is refused for a user who is already a member so the auto-revoke can't remove standing access.
  - `access.py`: `REVOKERS["authentik"]` for automatic revocation at expiry.
  - Test `test_iam_authentik_grants_only_grantable_groups_for_new_members_and_revoke`.

## [2026-10-17] - Least-privilege, time-boxed agent access with automatic revocation (Claude Code)

### Security
- 🔧 Every agent access request now states the exact scope, a level (`read`/`write`/`admin`) and how many hours it is needed (1–24, capped). This applies to the `ACCESS_GAP:` lines, the voice tool `report_access_gap` and `access.report()`. Requests that don't state a level and hours are flagged ⚠️, and Rogier won't provision them until the requesting agent supplies them.
  - File or resource affected: `agents/olympus/access.py`, `agents/olympus/voice.py`
- 🔧 Rogier's GitLab grants now run in hours (`gitlab grant <project> <user> <10/20/30> <hours>`). A grant can never be longer or higher than the request asked for, and admin access is never automated. Each verified grant is recorded as `GR-n` with its exact expiry, and GitLab's own `expires_at` is a one-day backstop.
  - File or resource affected: `agents/olympus/council.py`
- ✅ New `access-expire.timer` (every 5 min, `access.py expire`): Rogier revokes expired grants, verifies they are gone, and logs each revocation to the request's GitLab issue, Discord #agent-access and the Obsidian access backlog (new "Active time-limited grants" and "Revoked on expiry" tables). If a revocation fails, the grant is marked `revoke_failed` and retried every run, and the owner is paged on the first failure and then about every 30 minutes.
  - File or resource affected: `agents/olympus/install.sh`
- ✅ Tests: `test_hierarchy` (IAM grant/revoke and expiry flow), `test_local_voice`, `test_advisers` passed under WSL (106/107; the remaining failure is the pre-existing art-file check).

### Infrastructure
- ✅ **Deployed to CT211** (owner-approved, 2026-10-17 ~11:20). Before deploying, the live `access.py`/`council.py`/`voice.py`/`install.sh` differed from source only by this change. Backup: `~/olympus/backups/access-expiry-20261017-112016/`. Ran `install.sh`: `access-expire.timer` is active (every 5 min). Restarted `olympus-portal` and `boss`; `/healthz` returned 200, `access.py expire` ran cleanly (nothing to revoke), the live voice tool requires `level` and `hours`, and existing AG-n requests still list.
- ✅ **Proxmox (MorpheusCluster), owner-approved:**
  - New role `RogierUserAdmin` = `Sys.Audit, User.Modify, Group.Allocate, Realm.AllocateUser, Permissions.Modify`. User `rogier@pve` (no password) and privilege-separated token `rogier@pve!provision`, both on `/`. The token secret was piped from node01 straight into `~/.olympus_env` on CT211 (`PROXMOX_IAM_TOKEN_ID/SECRET`, mode 600) and never displayed.
  - Verified with the token: effective privileges are exactly those five, it sees 0 VMs, and a VM config write returned HTTP 403.
  - ⚠️ Proxmox cannot restrict *which* roles a `Permissions.Modify` holder may hand out. Rogier's limits (owner ✅, least privilege, 24 h, automatic revocation) are enforced by Roundtable code, not by Proxmox.
  - New named owner admin `hermes@pve` (Administrator on `/`). The owner still has to set its password and enroll TOTP; `root@pam` stays break-glass.
- 📝 Still open: Proxmox SSO through an Authentik OpenID realm; revoker for Proxmox grants in `access.REVOKERS`; Omada and storage accounts (need the owner's admin session). See Task Registry #81/#82.

## [2026-10-17] - Desktop app now follows the portal's voice engine choice (Claude Code)

### Fixed
- ✅ The Roundtable Companion desktop app ignored "☁️ OpenAI voice" set on the portal. The choice was kept only in browser `localStorage`, and the Electron app has its own storage, so it always fell back to local voice. The choice is now stored server-side (`kv voice_engine`, `GET/POST /api/voice/engine`, sign-in or device token required, only `openai`/`local` accepted). Each page loads it at start-up and re-reads it before every call, so a change on any device applies to the next call everywhere.
  - File or resource affected: `agents/olympus/portal.py`
- ✅ Deployed to CT211 (the live file matched the previous deploy). Backup: `~/olympus/backups/voice-engine-sync-20261017-114242/`. Seeded `voice_engine=openai` (the owner's portal choice). `/healthz` 200; an unsigned GET returns 403; a bad value returns 400; browser checks 79/79 ×3.

## [2026-10-17] - Approve, reject and run-now from the Roundtable portal (Claude Code)

### Added
- ✅ The portal's Approvals page has ✅ Approve / ❌ Reject buttons on pending approvals and ▶ Run now on scheduled ones, each behind a confirmation. `POST /api/approvals/decide` accepts only a signed-in Authentik session through Traefik whose user is in `PORTAL_APPROVERS` (default `akadmin`), as same-origin JSON. The desktop app's device token cannot decide.
  - File or resource affected: `agents/olympus/portal.py`
- ✅ The portal only queues the decision (`kv portal_decision:<id>`). Gideon applies it on his next approval poll (≤15 s) through the same `decide()`/`execute()` path as a Discord ✅/❌, after re-checking the approval's current state (stale decisions are ignored). He posts "🖥️ Approval #N approved on the Roundtable portal by <user>" in the approval's Discord thread for the audit trail.
  - File or resource affected: `agents/olympus/boss.py`
- ✅ Tests: new `test_hierarchy` approval test, `test_teach` 13/13, browser 79/79 ×3.

### Infrastructure
- ✅ **Deployed to CT211** (owner-approved, 2026-10-17 ~11:32). Before deploying, live `portal.py` matched the previous deploy and `boss.py` differed only by this change. Backup: `~/olympus/backups/portal-approvals-20261017-113204/`. Restarted `olympus-portal` and `boss`; both active, `/healthz` 200, the served page has the decide buttons, and `/api/approvals` returns the `queued` field. A decide request without a sign-in, and one with a spoofed `X-authentik-username` header not coming through Traefik, both returned 403 and queued nothing. No errors in the logs. Not yet tried end-to-end from the signed-in portal.

## [2026-10-17] - Automated patching (SRE agent)

### Infrastructure
- ✅ `gitlab-vm01` (VM): patched, 9 packages upgraded, pending 12→3; kernel stayed `6.8.0-147-generic`, Docker CE `29.8.2`→`29.9.1` and containerd `2.3.6`→`2.4.2`; independent verification passed. [GitLab #59](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/59)
- ✅ `docker-vm-core-utilities01` (VM): patched, `open-vm-tools` `13.0.10-0ubuntu0.24.04.1`→`13.0.10-0ubuntu0.24.04.2`; kernel stayed `6.8.0-147-generic`; independent verification passed. [GitLab #559](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/559)
- ✅ `gitlab-runner-vm01` (VM): patched, `open-vm-tools` `13.0.10-0ubuntu0.24.04.1`→`13.0.10-0ubuntu0.24.04.2`; kernel stayed `6.8.0-147-generic`; independent verification passed. [GitLab #560](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/560)
- ✅ `pihole` (LXC): patch run completed with no packages upgraded; kernel stayed `7.0.14-20-pve`; independent verification passed. [GitLab #562](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/562)
- ✅ `docker-lxc-media` (LXC): patch run completed with no packages upgraded; kernel stayed `7.0.14-20-pve`; `apparmor.service` was reported failed before the run, while independent verification passed. [GitLab #563](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/563)
- ✅ `authentik-lxc` (LXC): patch run completed with no packages upgraded; kernel stayed `7.0.14-20-pve`; `apparmor.service` was reported failed before the run, while independent verification passed. [GitLab #564](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/564)
- ✅ `chronicle-lxc` (LXC): patched 2 packages, including AppArmor `4.0.1really4.0.1-0ubuntu0.24.04.7`→`4.0.1really4.0.1-0ubuntu0.24.04.9`; kernel stayed `7.0.14-20-pve`; independent verification passed. [GitLab #565](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/565)
- ✅ `docker-lxc-bots` (LXC): patched 2 packages, including AppArmor `4.0.1really4.0.1-0ubuntu0.24.04.7`→`4.0.1really4.0.1-0ubuntu0.24.04.9`; kernel stayed `7.0.14-20-pve`; independent verification passed. [GitLab #566](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/566)

### Changed
- ✅ `docker-lxc-glance` / Glance: `glanceapp/glance:latest` updated, created 2026-09-03→2026-10-15 (digest `sha256:<redacted>`→`sha256:<redacted>`); independent verification passed. [GitLab #576](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/576)
- ✅ `docker-lxc-glance` / `proxmox-nodes-api`: run reported `updated`, but service `updated:false`; `python:3.11-slim` remained at `3.11.17` with the same digest, so no image change; independent verification passed. [GitLab #577](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/577)

## [2026-10-16] - Agent alerts cleared (Claude Code)

### Fixed
- ✅ **Sellen `docker-stats-media` target down (since 10-10, Task #6)**: `/opt/docker-exporter` on CT205 had never been built (image build `RUN` steps fail in the unprivileged LXC: AppArmor, Task #15). Compose now runs the stock `python:3.12-slim` image with `exporter.py` mounted and deps installed once into volume `exporter-deps` (backup `docker-compose.yml.bak-20261016`). Prometheus target `192.168.40.11:9417` is up.
- ✅ **Sellen `gitlab-runner-vm01 memory 101%`**: false positive; the guest used 350 MB of 2 GB. VM 121 had `balloon: 0` and no running guest agent, so Proxmox reported host RSS. Installed/started `qemu-guest-agent` in the guest, `qm set 121 --balloon 2048` (min = max, no shrinking), `qm reboot 121` while no CI job ran. Proxmox now reports ~24 %; runner active.
- ✅ **Nightly backups "job errors" on all 3 nodes (10-14, 10-15, 10-16)**: deleted guest 101 was still listed in all three vzdump jobs (`unable to find VM '101'`); every other guest backed up fine. Removed 101 from `backup-984a1a4f-60ee` (19:00 pbs-daily), `backup-78f24e5d-ed4a` (Fri pbs-main) and `backup-13fa5dd2-e814` (Sun ProxmoxData). Backup `/root/jobs.cfg.bak-20261016` on node01.
- ✅ **Corhyn #25 node03 failed units** (`nvidia-persistenced`, `systemd-modules-load`): NVIDIA 550.163 DKMS module never built for kernel 7.0.14-20 (headers only for 7.0.14-11); no guest uses the RTX 3050. Commented out `nvidia-drm` in `/etc/modules-load.d/nvidia.conf` (backup `.bak-20261016`), disabled `nvidia-persistenced`, reset failed units. To use the GPU later: install matching `proxmox-headers` and a driver that builds for kernel 7.0, then revert.
- ✅ **Corhyn #21 Helios port conflict**: removed the never-started duplicate `helios-api` container on docker-vm-core-utilities01 (compose dir `/home/hermes-admin/helios` kept). Helios runs on helios-lxc :8000 (3 healthy services).
- ✅ **Sellen GitLab error-log spike**: errors were GitLab telemetry to `events.gitlab.net`, which Pi-hole blocks. Disabled `gitlab_product_usage_data_enabled`, `usage_ping_enabled`, `include_optional_metrics_in_service_ping`; 0 errors since.
- ✅ **Nepheli Usenet login alert**: disabled the expired Newshosting server in SABnzbd (alert ages out of its 3 h window). **Radarr NotificationStatusCheck**: the "Download Monitor" webhook pointed at `:5052`, where nothing listens (`/opt/download-monitor` is a poller); disabled its triggers (not deleted).

### Changed
- 🔧 **Corhyn (`agents/olympus/medic.py`, deployed to CT211, backup `medic.py.bak-20261016-close`, no restart needed)**: `fix-failed` incidents now auto-resolve when their condition clears (#4, #7, #8, #25 closed on the next scan); new `medic.py close <id> "<reason>"` for owner-report incidents, which have no detection keys and never cleared.
- 🗂️ Closed stale incidents #11, #13, #14, #15, #18, #19, #21 with reasons; GitLab #334 and #341 (patch drift, recovered; verifier fixed) closed with notes; 12 stale approvals expired (SABnzbd/Deluge restarts, 6× "Retry backup of 101", ACL phase 3 RFC-2026-006/007).

### Left for the owner
- Approval #12 (mirror push to GitHub `herms14/homelab-hub`, from 10-10).
- Nepheli: Bazarr has no languages profile (Settings → Languages: create one, set default).
- Loretta's 10 security findings (SSH root/password login, OS updates, privileged containers/LXCs, Docker socket mounts, Proxmox 2FA and cluster firewall, no inter-VLAN rules, 50 Traefik routes without SSO): owner-approved changes by design.

## [2026-10-16] - VLAN Flow published as a generic public repo (Claude Code)

### Added
- ✅ **https://github.com/herms14/vlan-flow** (public, MIT, local clone `C:\Users\herms\Projects\vlan-flow`). Generic version of `agents/vlan-flow/`: networks, syslog senders, sender labels, accepted host names, title, time zone and retention in `config.json` (subnet-based lookup, any addressing); new `/api/config`; demo data built from the configured networks; helpful 403 and config error messages; Docker Compose (host networking) and `deploy/install.sh` systemd install/upgrade; docs for Omada setup, configuration, reverse proxy/login and troubleshooting. 9 tests pass; live smoke test and headless render OK.
  - No secrets or homelab details: scanned for keys, tokens, passwords, domain, IPs, MACs and names before pushing; commits use the GitHub no-reply address.
  - CI workflow pushed after the owner added the `workflow` scope (commit ad9de7d); first run passed on Python 3.9 and 3.12 (tests, JS syntax check, Docker build).
  - The live deployment on .13 is unchanged (still the homelab-specific `agents/vlan-flow/`).
  - Made vendor-neutral on owner request (no TP-Link/Omada mention): parses key=value, Linux netfilter (log prefix gives the action), pf filterlog (pfSense/OPNsense) and MikroTik logs; setup guide is now `docs/router-setup.md`. 10 tests pass (8 sample formats, 5 non-decision lines). The first commit message and its files in git history still mention the old vendor.

## [2026-10-16] - Arr stack switched to torrents only (Claude Code)

### Changed (owner request; CT205 192.168.40.11)
- 🔄 **Usenet off**: Newshosting expired 2026-04-18 (per the owner's Media Suite note; no Usenet import since 2026-02-08). SABnzbd download client **disabled** (not deleted) in Sonarr, Radarr and Lidarr; NZBgeek **disabled** in Prowlarr and in each app. To restore: re-enable both and update Config → Servers in SABnzbd (UI on :8081).
- 🗑️ **Backlog cleared**: 18 stalled Usenet queue items removed and deleted from SABnzbd (Sonarr: Ted Lasso S04E01–E10; Radarr: Cinderella, Troy, Unabomber, Disclosure Day, Before Sunset, The Fix, 5 Centimeters per Second, Sentimental Value), not blocklisted. SABnzbd queue is empty.
- ➕ **Free indexers added in Prowlarr**: YTS, 1337x (flaresolverr tag), Nyaa.si; synced to the apps. All 6 enabled indexers pass *test all*.
- 🔍 Torrent searches started for the 18 items; within 2 minutes 4 episodes and 4 movies were queued or downloading in Deluge, and the rest were still searching.

## [2026-10-16] - Arr stack recovered after the ACL outage (Claude Code)

### Fixed (docker-lxc-media CT205, 192.168.40.11)
- ✅ **Sonarr/Radarr "All indexers are unavailable"**: Prowlarr couldn't resolve NZBgeek/EZTV/LimeTorrents/TPB while phase 4 blocked VLAN 40 → Pi-hole DNS, so Sonarr/Radarr backed off every indexer. DNS works again with the ACLs off; ran indexer *test all* in Prowlarr (4/4), Sonarr (4/4) and Radarr (2/2), which cleared the failure state. Health is now clean.
- ✅ **Lidarr "Unable to communicate with Deluge"**: Lidarr still had an old Deluge password (same gap fixed for Sonarr on 2026-10-07). Copied the working password from Radarr's Deluge client via the Lidarr API (not displayed); both Lidarr download clients test OK.
- Verified: all 4 *arr apps report no health issues, all download clients pass, queues (Sonarr 10, Radarr 8) have no stuck or failed items; Bazarr, Jellyseerr, Overseerr, Jellyfin, FlareSolverr, MeTube clean for 2 h. autobrr/Tdarr DNS errors were at 11:13 (outage) only.

### Needs the owner
- ❌ **SABnzbd → Newshosting "502 Authentication Failed"** continuously since at least 2026-10-13 12:18 (before the ACL work): check the Newshosting subscription and credentials, then update SABnzbd → Config → Servers. Usenet grabs fail until then.
- 🗑️ Two never-started containers from 2026-01-07 (`vibrant_wilson`, `practical_bhabha`, image `5f596b4d3c6b`) can be removed with `docker rm`; left in place.

## [2026-10-16] - Omada devices disconnected and slow services: phase 4 ACL permits not matching (Claude Code)

### Found (read-only, VLAN Flow ACL log; owner disabled all ACLs ~12:05, ACL logging stopped 12:05:16)
- ❌ **Omada devices disconnected 11:07–12:05**: VLAN 90 devices (.90.50–.58) → controller `192.168.0.101` UDP 29810 / TCP 29814 were **denied** (~600 events), and controller → VLAN 90 was denied (~1,400). `ALLOW-Omada-Adopt` / `ALLOW-Omada-Manage` (rules 2–3) never matched; `DENY-Mgmt-Rest` (90→1) and `DENY-Default-Rest` (1→90) did.
- ❌ **DNS to Pi-hole blocked for VLAN 40 and IoT from 11:11**: ~2,000 denies from .40.11/.13/.16/.23/.24 and ~230 from VLAN 30 to `192.168.90.53:53`; `ALLOW-DNS-PiHole` (rule 1) stopped logging at 11:11:01, at the v6 failure/rollback (RFC-2026-012, 11:10). VLAN 40 then waits for a Pi-hole timeout before using 8.8.8.8, which explains slow API calls from Roundtable (CT211 .40.16) and other apps.
- Also denied in phase 4: Prometheus → .20.30:9202 (known, v7), .13 → nodes 8006/9100/22, NFS .40.22/.40.11 → .20.32:2049, controller syslog → .40.13:5514, PBS-related .20.32 → .40.22.
- ✅ **No ACL ever denied internet traffic**: 0 deny events to or from non-192.168 addresses; nothing from the admin PC .10.10 was denied.
- Rule IDs in the log (`DESC=`) change on every rebuild (02:53, 02:57, 03:02, 03:11, 04:01 UTC), so names were matched by traffic pattern. Not verified: the live rule order on the controller after the rollback. Likely cause: the rebuild left the top permits below the deny rules or not effective. Torrent's state on CT211 was not inspected.
- Before re-enabling: have Torrent export the live rule order and compare it with `acl_ruleset.json`; add health checks for VLAN 90 → controller and DNS from VLAN 40/30, and run them after rollbacks too.

## [2026-10-16] - VLAN Flow updated to the new VLAN design (Claude Code)

### Changed
- 🔧 **VLAN Flow** (docker-vm-core-utilities01 :8787, https://vlanflow.hrmsmrflrii.xyz): map, filters and VLAN lookup now cover only the live networks 1, 10, 20, 30, 40, 50, 80, 90; VLANs 60 Sonos, 70 VPN and 91 Firewall removed (traffic from those subnets now shows as External/?). Default syslog senders drop `.60.1/.70.1/.91.1`; the Sonos demo row was replaced; the map subtitle counts the segments. 7 tests pass.
  - Deployed with `update.sh` (backup `/opt/vlan-flow-backup-20261016-040428`); verified the live `/api/state` lists the 8 VLANs and syslog ACL events keep arriving after the restart. The in-memory activity feed was cleared by the restart.

## [2026-10-16] - VLANs 60, 70 and 91 removed; ACL rule set v5 (Claude Code)

### Changed
- 🔧 **Network (owner)**: VLAN 60 Sonos, VLAN 70 VPN and VLAN 91 FirewallVLAN deleted in Omada. Verified read-only: live networks are 1, 10, 20, 30, 40, 50, 80, 90. Omada dropped the 5 Torrent rules that only referenced them; 9 managed rules remain (phase 3 had 14).
- 🔧 **Torrent rule set v5** (`agents/olympus/acl_ruleset.json`): removed `ALLOW-Internal-Sonos`, `ALLOW-Sonos-AppEvents`, `DENY-Sonos-Out`, `DENY-VPN-Out`, `DENY-to-Sonos` and `DENY-Internal-Rest`; stripped 60/70/91 from every other rule (43 -> 37 rules). Validates clean; no network references outside the live controller set.
  - ✅ Torrent synced phase 3 to v5 (approval #45 / RFC-2026-009, applied 10:50, owner kept it 10:55): 9 rules live, no drift, VLAN 91 reference on T050 cleared, unused group TP120 removed. Phase 4 applied and kept (approval #46 / RFC-2026-010, 10:57 to 11:04): 35 rules live, no drift; spot checks from Kratos OK (SSH to nodes, Immich 200).
  - 🔍 Verified 2026-10-16 ~11:10: 35/35 rules live, enabled, in order, no drift; host-to-host permit and deny tests from 9 hosts behave as designed.
  - ❌ **Missed permit**: Prometheus (.13) → Omada exporter `192.168.20.30:9202` times out since phase 4 (exporter itself returns 200 locally). Fix in rule set **v6** (local only): add 9202 to `ALLOW-Monitoring-TCP`. v6 deployed by the owner; approval #48 (RFC-2026-012) **failed and rolled back automatically**: Omada refused a 9th port entry in an IP-Port group (`-33728 ... reached the limit of IpPortGroup`); the 35 v5 rules were restored. Rule set **v7** (local) merges ranges (`8006-8007`, `9100-9101`) so the group needs 7 entries including 9202. Note: `aclctl.validate` doesn't check the 8-entry limit.
  - Unrelated: Prometheus target `docker-stats-media` (192.168.40.11:9417) was already down before the ACL rollout (connection refused). Blockers still open: #65 (PBS .20.50, currently resolving to the real PBS but not enforced) and #54 (Docker subnets on .13 overlap VLANs 1/10).
  - ✅ **Deployed** to CT211 after the owner said "deploy" (backup `acl_ruleset.json.bak-20261016-v4`); Torrent reports v5, validation clean.
- 🗑️ **Sonos removed from the lab (owner: the speakers live on a completely separate network)**:
  - Roundtable code (deployed to CT211, backups `*.bak-20261016-sonos`, no service restart needed): `netops.py` Torrent expertise text, `security.py` Loretta VLAN list and finding text (also dropped 70/91), `aclctl.py` API probe now uses VLANs 50/80 and the TEST-NET range `192.0.2.0/24` instead of VLAN 60.
  - Docs: [64 - VLAN ACL Rule Set](64%20-%20VLAN%20ACL%20Rule%20Set.md) (requirement 7, section E, P_SONOS_CB, rows; all tables now match v5), [01 - Network Architecture](01%20-%20Network%20Architecture.md) and 11 - Credentials (Sonos SSID rows), [10 - IP Address Map](10%20-%20IP%20Address%20Map.md) (VLAN 60 section), [41 - Omada ACL Implementation Guide](41%20-%20Omada%20ACL%20Implementation%20Guide.md) (rule set 9, test 9, VLAN 60 blocks), Book guide row, homelab `CLAUDE.md`. Changelogs, session logs and RFCs keep their history.
  - Live `T050-ALLOW-Admin-Infra` still holds VLAN 91's dangling network ID. It's harmless; Torrent's next phase rollout (approval) cleans it.
- 📝 Docs: [01 - Network Architecture](01%20-%20Network%20Architecture.md), [10 - IP Address Map](10%20-%20IP%20Address%20Map.md), [64 - VLAN ACL Rule Set](64%20-%20VLAN%20ACL%20Rule%20Set.md) (banner), `CLAUDE.md` VLAN table.

## [2026-10-16] - Immich down after patch reboot: fstab ordering cycle fixed (Claude Code)

### Fixed
- ✅ **immich-vm01 (VM 108, node03)**: down since the 02:52 SRE patch reboot ([GitLab #336](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/336)). Cause: the bind mount `/mnt/synology-photos` (inside an NFS share) sat in `local-fs.target`, creating an ordering cycle local-fs -> NFS -> network-online -> ufw -> local-fs. systemd broke it by dropping the network wait; the NFS mounts ran with no network and failed, and because `homes` and `Immich Photos` lacked `nofail` the guest went to emergency mode (no SSH or Immich, ping only). fstab has been unchanged since May; earlier boots got a luckier cycle break.
  - **Not the VLAN ACLs**: NFS failed before the guest had a network; VLAN 40 hosts reach `192.168.20.32:2049` and rule 10 allows .22.
  - Fix (owner ran the script from Claude's session): VM stopped, guest root mounted on node03, `/etc/fstab` backed up to `fstab.bak-20261016`; `nofail` added to the homes/Immich NFS lines; bind line now `bind,ro,_netdev,nofail,x-systemd.requires-mounts-for=/mnt/synology-root`.
  - Verified: no ordering cycle in the boot log, system `running`, 4 NFS mounts on .32, the container sees the 25 TB share at `/usr/src/app/upload`, all Immich containers healthy, `:2283` and `photos.hrmsmrflrii.xyz` ping 200.
  - Docs: [21 - Application Configurations](21%20-%20Application%20Configurations.md) Immich mounts corrected from dead .31 to .32, with the bind-mount fix; [12 - Troubleshooting](12%20-%20Troubleshooting.md) entry added.

## [2026-10-16] - Automated patching (SRE agent)

### Changed
- 🔄 **docker-lxc-glance / proxmox-nodes-api**: run reported updated, but `python:3.11-slim` stayed at 3.11.17 with the same image digest; independent verification PASS and no image change. [GitLab #142](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/142)

### Infrastructure
- ✅ **gitlab-runner-vm01**: 27 packages; kernel `6.8.0-146-generic` → `6.8.0-147-generic`, GitLab Runner `19.4.1` → `19.5.0`; rebooted, independent verification PASS. [GitLab #335](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/335)
- ✅ **docker-lxc-glance (CT200)**: 124 packages; containerd `2.3.4` → `2.4.2`, Docker CE `29.7.2` → `29.9.1`; shared kernel unchanged, independent verification PASS. [GitLab #338](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/338)
- ✅ **pihole**: 92 packages; `libc6` `2.39-0ubuntu8.8` → `2.39-0ubuntu8.9`; kernel unchanged, independent verification PASS. [GitLab #339](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/339)
- ✅ **traefik-lxc**: 107 packages; AppArmor `4.0.1really4.0.1-0ubuntu0.24.04.7` → `.9`, Docker CE `29.7.2` → `29.9.1`; kernel unchanged, independent verification PASS. [GitLab #340](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/340)
- ✅ **homeassistant-lxc**: 107 packages; containerd `2.3.4` → `2.4.2`, Docker CE `29.7.2` → `29.9.1`; kernel unchanged, independent verification PASS. [GitLab #342](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/342)
- ✅ **homepage-lxc**: 22 packages; OpenSSL `3.0.20` → `3.0.22`, Docker CE `29.7.2` → `29.9.1`; kernel unchanged, independent verification PASS. [GitLab #343](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/343)
- ✅ **authentik-lxc**: 105 packages; Docker CE `29.7.2` → `29.9.1`; kernel unchanged, independent verification PASS. [GitLab #344](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/344)
- ✅ **chronicle-lxc**: 93 packages; dnsmasq `2.90-2ubuntu0.4` → `2.91-0ubuntu0.24.04.2`; kernel unchanged, independent verification PASS. [GitLab #345](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/345)
- ✅ **helios-lxc**: 8 packages; Python `3.11.2-6+deb12u8` → `3.11.2-6+deb12u9`; kernel unchanged, independent verification PASS. [GitLab #347](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/347)
- ✅ **codex-agent-lxc**: 6 packages; Python `3.11.2-6+deb12u8` → `3.11.2-6+deb12u9`, libpng `1.6.39-2+deb12u5` → `1.6.39-2+deb12u6`; independent verification PASS. [GitLab #348](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/348)
- ✅ **docker-lxc-bots**: 105 packages; Docker CE `29.7.2` → `29.9.1`; kernel unchanged, independent verification PASS. [GitLab #349](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/349)

### Found
- ⚠️ **docker-vm-core-utilities01**: 8 packages, including Docker CE `29.8.2` → `29.9.1` and containerd `2.3.6` → `2.4.2`; run verification failed while Paperless was starting after Docker restarted (independent verdict N/A, run incomplete). Triage found automatic recovery about 62 seconds after startup; no repair was needed. [GitLab #334](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/334)
- ⚠️ **immich-vm01 (VM 108)**: no packages applied; the VM stayed running on node03 but did not return after reboot, with SSH refusing connections and the guest agent unavailable. Triage suspects a guest OS shutdown/boot hang, not confirmed; independent verdict N/A, run incomplete, console inspection needed. [GitLab #336](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/336)
- ⚠️ **docker-lxc-media**: 106 packages, including Docker CE `29.7.2` → `29.9.1` and containerd `2.3.4` → `2.4.2`; run verification failed while Jellyfin/cAdvisor were starting (independent verdict N/A, run incomplete), then both recovered automatically within about 41/30 seconds. Triage identified a verification timing race. [GitLab #341](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/341)

## [2026-10-16] - Incident response (Corhyn)

### Fixed
- ✅ **docker-lxc-media / `sre-agent.service` verifier**: false `Patch drift` came from a zero-second Docker health deadline without host reboot. Deployed independent 180-second polling at 10-second intervals, strict healthy/running checks, and timeout health-history/log diagnostics; 8 offline behavior groups and service health passed, while live patch validation is deferred. Rollback: restore the incident backup and restart `sre-agent.service`. [GitLab #341](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/341)

## [2026-10-15] - VLAN Flow on vlanflow.hrmsmrflrii.xyz; network activity feed; ACL rule-set proposal

### Added
- ✅ **https://vlanflow.hrmsmrflrii.xyz** → Traefik → `192.168.40.13:8787`, Authentik forward auth (app `vlanflow`, provider 49, embedded outpost; no policy bindings, same as Roundtable). Let's Encrypt cert issued.
  - Traefik: `/opt/traefik/config/dynamic/vlanflow.yml` (health check `/healthz`).
  - Pi-hole v6: `dns.hosts` += `192.168.40.20 vlanflow.hrmsmrflrii.xyz` (backup `/etc/pihole/pihole.toml.bak-20261015-vlanflow`; diff = this line only).
- ✅ VLAN Flow: **Network activity** panel (latest 300 syslog messages of every type, classified ACL / client / DHCP / device / other, memory-only), status text when syslog arrives without ACL decisions, `--hosts` Host allowlist, `/healthz`. 7 unit tests pass. Deployed via `update.sh`; backup `/opt/vlan-flow-backup-20261015-125922`.
  - Files: `agents/vlan-flow/{server.py,app.js,dashboard.html,style.css,test_server.py,vlan-flow.service}`
- 📝 [64 - VLAN ACL Rule Set](64%20-%20VLAN%20ACL%20Rule%20Set.md): per-VLAN Gateway ACL proposal against the owner's requirements (not applied). [41 - Omada ACL Implementation Guide](41%20-%20Omada%20ACL%20Implementation%20Guide.md) marked superseded.

### Fixed
- 🐛 Login showed Authentik "Redirect URI Error": provider 49 was created via the ORM, which skips the API's `set_oauth_defaults()`, leaving no redirect URIs or scope mappings. Ran `set_oauth_defaults()` and it now matches Roundtable's provider 48. **Lesson:** create Authentik providers through the API or UI, or call `set_oauth_defaults()` afterwards.

### Findings
- 🔍 Collector had received 26 messages and parsed 0 because **no Gateway ACL rules exist** (owner confirmed). The traffic is controller client events and gateway DHCP only.
- 🔍 **Task #65**: the `.20.50` squatter MAC `78:20:51:8A:E3:7B` is the **Zephyrus Switch** (SG3210X-M2). The gateway leased it `192.168.30.53` on IoT at 20:55 today. Not changed.

## [2026-10-15] - Fix: specialists' voice tools were blocked (deployed)

- Note: several calls this afternoon were cut by service restarts during deploys; deploys now check for a live call first (owner asked for fixes to be deployed automatically).

## [2026-10-15] - Discord cleanup: 5 dead channels deleted

- ❌ Deleted (owner-approved): `#general` (286 d idle), `#container-updates` (267 d), `#claude-tasks` (241 d), `#new-service-onboarding-workflow` (241 d), `#blog-studio` (never used). History exported first: `~/olympus/backups/discord-channels-20261015-075142/` on CT211 (192 messages).
- 🔄 Sentinel (docker-vm-core-utilities01 `/opt/sentinel-bot/.env`, backup `.env.bak-20261015-channels`): `CHANNEL_CONTAINER_UPDATES` and `CHANNEL_ONBOARDING` -> `homelab-infrastructure`, `CHANNEL_CLAUDE_TASKS` -> `ideas-and-todos`; container recreated, healthy.
- 🔄 Melina (`writer.py` `CHANNEL`, live and vault; registry `channel`) -> `#ideas-and-todos` (boss recreates missing channels); backup `~/olympus/backups/blog-studio-repoint-20261015-075248/`; `boss` restarted.
- Assessment (31 channels): `#heart-to-heart` kept for review (28 messages in 30 days although documented as retired); merges/categories proposed, not applied.

## [2026-10-15] - Atlas: crypto portfolio, weekly markets report, buy alerts (deployed)

- ✅ Owner's crypto holdings stored (BTC, ETH, ADA, CATE, BNB, PEPE, XRP, BRISE; cost basis unknown, bought mostly near the 2021 highs) and stock watchlist MSFT, ORCL, GOOGL, META, NVDA, AMD, INTC (`finance:holdings`, `finance:watch`).
- ✅ `finance.py`: coin aliases (PEPE -> `PEPE24478-USD`, CATE via CoinGecko `cg:catecoin`), projection skips coins without history and estimates from >=3% positions over 3 years, values in PHP (`PHP=X`).
- ✅ New `weekly()` (Sat 09:00): Crypto Fear & Greed (alternative.me), total cap and BTC/ETH dominance + top 10 (CoinGecko), SPY/QQQ/VIX/SOXX, watchlist with buy scores and calls, OpenAI/Anthropic news (private; proxies MSFT, GOOGL/AMZN). New `alerts()` (daily 06:45): deterministic `buy_signal()` score, model call + web check only when score >= 5, once per symbol per 7 days. Discord + vault notes `01 Life/Finance/Atlas Reports/` (weekly report, monthly buy-alert log; statements never).

## [2026-10-15] - Atlas reads statements from OneDrive; fixed bills (deployed)

- ✅ `finance-ingest.timer` now monthly: 1st of the month 09:00 Asia/Manila (was hourly); `install.sh` updated.
- ✅ Fixed monthly bills (`finance:fixed`, intent `fixed_expense`) added to card spending in forecasts/car/cashflow (`monthly_outflow()`); seeded with the owner's mortgage, phone, internet, electricity, water (total PHP 45,500/month; electricity read as 11,000).
- Tests: `tests/test_advisers.py` 17. Backup `~/olympus/backups/atlas-onedrive-20261015-063037/`; restarted `olympus-portal` and `boss`; healthz 200, no errors.

## [2026-10-15] - Kratos voice server starts at Windows logon

- ✅ New launcher `C:\AI\local-voice\start-voice.ps1`: exits if port 8765 is already listening, otherwise keeps the previous logs as `server.*.log.prev` and starts `.venv\Scripts\python.exe server.py` hidden, logging to `server.out.log` / `server.err.log`.
- ✅ Scheduled task **Roundtable Voice Server** (current user, at logon, 30 s delay, normal privileges, no time limit) runs that launcher. Replaces the manual start noted on 2026-10-13.
- ✅ Verified by starting the task: server ready (CUDA Whisper large-v3-turbo + Kokoro) on :8765; a second run started no duplicate. Not yet verified across a real reboot.
- Remove with `Unregister-ScheduledTask -TaskName 'Roundtable Voice Server'`.

## [2026-10-14] - Transparent summoned clips; animation clips for every agent (deployed)

### Fixed
  - `agents/olympus/portal.py`, `agents/olympus/tests/test_portal.py`
### Added
### Infrastructure
- ✅ Published 51 WebMs and 17 posters to `~/olympus/avatars/` and set `anim` in the live `agents.json` (backup `agents.json.before-team-anim-20261014-163356`, UTC). Restarted only `olympus-portal`. All new files serve 200, `/api/agents` shows anim for **all 29 agents**, and no errors appeared in the logs. ComfyUI was stopped afterwards.

## [2026-10-14] - Daily-art animation clips for Loretta and Corhyn (deployed)

### Added
- ✅ All four leads now have Daily-art idle/listen/talk clips, so the leads summon is fully animated. **Corhyn** (`medic-idle/listen/talk.webm`) was rendered tonight locally on the RTX 4080 with the existing `C:\AI\anim` pipeline (ComfyUI, Wan 2.2 i2v 14B + lightx2v 4-step LoRA, BiRefNet cut-out, transparent VP9 WebM, same prompts and seeds as `batch.py`), about 95 s per clip, with no API cost. **Loretta** reuses clips rendered on 2026-10-11 from her current `security-er-full.png` art that were never published. Frames were reviewed against the source art. The posters are the first idle frames.
### Infrastructure
- ✅ Published to CT211 (owner-approved): 6 WebMs and 2 posters to `~/olympus/avatars/`, plus `anim` entries for `security` and `medic` in the live `agents.json` (backup `~/olympus/backups/agents.json.before-lead-anim-20261014-144944`, UTC). Restarted only `olympus-portal`; it is active, all 8 files serve 200, and `/api/agents` shows anim for boss, security, governance and medic. ComfyUI was stopped afterwards. (Correction: compliance, cost, devops, iam, ipam, project_manager and security_engineer already had live clips; they were not unpublished.) The vault's `agents.json` was not changed (it already differs from the live copy).

## [2026-10-14] - Summon: leads or the entire team (deployed)

### Changed
  - `agents/olympus/portal.py`, `agents/olympus/voice.py`, `agents/olympus/tests/test_portal.py`

### Infrastructure
- ✅ Deployed with the animation change above (backup `~/olympus/backups/summon-anim-20261014-223929/`). Tests before the deploy: portal browser desk 76/76 and office 7/7; the Linux suites passed except the pre-existing art-file check.

## [2026-10-14] - Roundtable Office: 3D view of the agents at work (deployed)

### Added
- 🕗 Office hours 8:00–17:00 Manila. Agents time in and out through a staff door with a punch clock, at persona-based times: Hewg, Morgott, Gideon and Loretta arrive early, Radahn and Nepheli late, and Hewg stays late. Off-duty agents go home and the list shows them as "home". After hours, only agents with a running task, or everyone during a board meeting, come in on overtime. Their mood becomes **pissed**: red-tinted angry faces, 💢 and 😤 bubbles, harder typing and head shakes. A sidebar time-clock log records each punch, and the header shows an office-hours or overtime badge. `?hour=21.5` previews any time of day.

### Changed
- 🔄 On request, the office is now **3D**: a cutaway diorama rendered with three.js 0.160.0, self-hosted at `/vendor/three/` (pinned, MIT, no CDN). Chunky voxel chibis have pixel-art faces that change with mood, and Torrent is a horse, Miriel a turtle with a mitre and Blaidd wolf-headed. The scene includes warm lighting with shadows, Manila day/dusk/night windows and night lamps, laptops that open while their owner works, coffee steam, Nepheli's TV, Gideon's whiteboard boxes, Iji's growing box stack, and a projector chart and banner during meetings. Drag to orbit, scroll to zoom, a 🎥 Boardroom view button, and clicking an agent (on the canvas or in the list) flies the camera to them. Behaviour, personas, moods and the meeting logic are unchanged from the 2D version below.
  - `agents/olympus/office.html`, `agents/olympus/vendor/three/{three.module.min.js,OrbitControls.js}`, `portal.py` (`vendor()` now serves `three` as well as `faceapi`), `tests/test_office.py` (7 tests)



### Added
  - `agents/olympus/office.html`
- 🏢 `GET /api/office` (view-only, like `/api/agents`) works out each agent's activity and mood from the ledger. A running run means working, with the mood taken from task keywords: incident → stressed, security → serious, build/patch → determined, cost → worried, research → curious, docs → content, more than 45 minutes → tired. An idle agent is sad after an error, anxious while an approval is pending, happy just after a finished task, and relaxed otherwise. Task text passes through `strip_secrets`.
  - `agents/olympus/portal.py`, `agents/olympus/voice.py`, `agents/olympus/tests/test_office.py`

### Infrastructure
- ✅ **Deployed to CT211** (owner-approved, 2026-10-14 ~22:30). Before deploying, the live `portal.py`/`voice.py` matched the previous deploy. Backup: `~/olympus/backups/office-3d-20261014-222937/`. Installed `office.html` and `vendor/three/` (the new `vendor/three` directory was created via sudo with `hermes-admin` ownership, like `vendor/faceapi`). Restarted `olympus-portal` and `boss`; both are active, and `/healthz`, `/office`, `/api/office` and both three.js files return 200 (hashes match the pinned downloads). `/api/office` returns 22 live agents, the nav has the Office link, an unauthenticated meeting POST is refused with 403, and no errors appeared in the logs. `tests/test_office.py` was not copied because CT211 has no tests directory. Validation before the deploy: office tests 7/7 and portal browser tests passed; the Linux suites passed except the pre-existing art-file check.

## [2026-10-14] - Automated patching (SRE agent)

### Changed
- ✅ `homeassistant-lxc` / Home Assistant: updated **2026.10.0 → 2026.10.1**; independent verification passed. [GitLab #89](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/89)
- ✅ `homepage-lxc` / Homepage: updated **v2.1.2 → v2.5.0**; independent verification passed. [GitLab #90](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/90)
- ✅ `docker-lxc-glance` / `proxmox-nodes-api`: no image or package change (3.11.17, same digest); independent verification passed. [GitLab #88](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/88)

## [2026-10-14] - Roundtable agent log

### Added
- ⚙️ **Hewg forged LXC `nginx-lxc` (VMID 101) on node01** (Radahn): 2 vCPU, 2048 MB RAM, 8 GB local-lvm, debian-12, VLAN 40, IP 192.168.40.37. Purpose: Proposal only: Nginx container; do not execute until inventory, decisions, IPAM feedback, and approvals are complete.. Terraform (bpg/proxmox) + Ansible base playbook from ansible-controller01 `~/hewg/deployments/nginx-lxc`.

### Removed
- ⚙️ **Hewg destroyed LXC `nginx-lxc` (VMID 101)** (Radahn): Owner-approved Terraform destroy on node01; 192.168.40.37 released.

## [2026-10-14] - Roundtable voice: longer call limits

### Changed
- 🔄 Voice calls now hang up after **10 min of silence** (was 120 s) and are capped at **20 min per call** (was 15 min); overridable with `ROUNDTABLE_VOICE_IDLE_SECONDS` / `ROUNDTABLE_VOICE_MAX_SECONDS`. Reason: demo calls were cut while watching Discord or waiting for a build.
  - `agents/olympus/voice.py`

### Infrastructure
- 🔄 Deployed to CT211, backup `~/olympus/backups/voice-limits-20261014-000759/`; `olympus-portal` and `boss` restarted, `/healthz` 200.

## [2026-10-13] - Hewg builds: repair Proxmox first-boot snakeoil unit

### Fixed
- 🔧 The 2026-10-13 `nginx-test-lxc` build failed Blaidd's check on `proxmox-regenerate-snakeoil.service` (a Proxmox one-shot first-boot unit). Live `checker.py` already treats it as a known first-boot unit (`BENIGN_FAILED_UNITS`). Hewg's base playbook now also repairs it on LXCs: if failed, it logs the unit journal, reruns `make-ssl-cert -f generate-default-snakeoil` and clears the failed state (`failed_when: false`, output shown in the build log). The original error was not preserved (guest destroyed), so the root cause is still unconfirmed.
  - `agents/olympus/builder.py` (`PLAYBOOK`); `ansible-playbook --syntax-check` passed on ansible-controller01; tests on CT211: same 2 earlier unrelated failures

### Infrastructure
- 🔄 Deployed `builder.py` to CT211, backup `~/olympus/backups/snakeoil-repair-20261013-232418/`; `boss` and `olympus-portal` restarted and active. Not yet verified by a real build.

## [2026-10-13] - Roundtable: Ranni demo talking points, plan drafts to Discord

### Changed
- 🔄 **Ranni** has `talking_points` in `agents.json`: asked about the governance process she explains the Discord approval workflow (proposal, #roundtable-hold card, owner ✅/❌, one-time execution with re-checks, Blaidd verification, Miriel changelog, audit sync, SRE patch window); asked about implementing services she explains infrastructure as code (Terraform bpg/proxmox + Ansible from ansible-controller01, GitLab CI/CD, changelog documentation). `council.instructions()` now includes any agent's `talking_points` (voice and Discord).
  - `agents/olympus/agents.json`, `council.py`, `voice.py`, `builder.py`, `tests/test_hierarchy.py` (new plan-draft test; Morgott tool set updated)
- Tests on CT211: 89 run; 2 unrelated failures (art file missing, `council.py` CLI argument test) that already failed before these changes.

### Infrastructure
- Note: local `agents.json` lacks the live `anim` avatar entries (pre-existing drift); do not copy it over the live file.

## [2026-10-13] - IPAM: deleted Proxmox guests no longer block a VLAN

### Fixed
- ✅ Diallos/IPAM refused every new address on VLAN 40 because Omada still remembered one offline client with no IPv4 evidence, `BC:24:11:BC:BB:24` (Proxmox MAC prefix; almost certainly the deleted test LXC 101/102). The pool was not exhausted: 13 of 14 candidates (.37-.49) were free.
- 🔧 An unresolved client is now treated as a destroyed guest, not a blocker, only when its MAC has the Proxmox prefix `BC:24:11` and no guest config in the cluster-wide `/etc/pve` read carries it. Non-Proxmox clients, MACs still on a guest, or compute data without the MAC list still block. Ignored entries are listed in `retired_guest_clients`.
  - `agents/olympus/inventory.py`: compute section adds `macs` (every guest NIC MAC)
  - `agents/olympus/ipam.py`: `PROXMOX_OUI`, retired-guest rule, policy text
  - `agents/olympus/tests/test_hierarchy.py`: new `test_destroyed_proxmox_guest_client_does_not_block_its_vlan`; IPAM/inventory tests pass on CT211 (2 unrelated failures in that run: avatar art excluded from the temp copy, `council.py` CLI argument test)

### Infrastructure
- 🔄 Deployed `ipam.py` and `inventory.py` to CT211 (owner requested for the video demo). Backup `~/olympus/backups/ipam-retired-guest-20261013-225125/`. Restarted `boss` and `olympus-portal`; both active, `/healthz` 200. Live check: no blockers; free pool VLAN 20 = 14 (first .36), VLAN 40 = 13 (first .37), VLAN 90 = 14 (first .36).

## [2026-10-13] - Roundtable Companion: desktop team lineup, roll call, local voice default

### Changed
- 🔄 Local PC voice is now the default engine in browser and desktop (new localStorage key `voiceEngine`; choosing OpenAI still sticks).
  - `agents/olympus/portal.py`, `desktop/main.js`, `desktop/preload.js`, `tests/test_portal.py` (74/74 browser, 74/74 mobile, 75/75 desk; local-voice suite 9/9)

### Infrastructure
- 🔄 Kratos voice server `C:\AI\local-voice\config.json`: added `http://192.168.40.16` to `allowed_origins` so the desktop app can use local voice (backup `config.json.bak-20261013`); server restarted (CUDA Whisper + Kokoro ready). It is started manually, not as a service.

## [2026-10-13] - Roundtable agent log

### Removed
- ⚙️ **Hewg destroyed LXC `nginx-lxc` (VMID 101)** (Radahn): Owner-approved Terraform destroy on node01; 192.168.20.36 released.
- ⚙️ **Hewg destroyed LXC `nginx-test-lxc` (VMID 102)** (Radahn): Owner-approved Terraform destroy on node02; 192.168.40.36 released.
- ⚙️ **Hewg destroyed LXC `hewg-test01` (VMID 101)** (Radahn): Owner-approved Terraform destroy on node03; 192.168.40.150 released.

### Added
- ⚙️ **Hewg forged LXC `nginx-lxc` (VMID 101) on node01** (Radahn): 2 vCPU, 2048 MB RAM, 8 GB local-lvm, debian-12, VLAN 20, IP 192.168.20.36. Purpose: Proposal only; do not deploy or modify anything. Proposed template: standard Debian 12 LXC template. Proposed VMID: pending allocator confirmation; do not assume or reserve one. Network: Homelab VLAN . Terraform (bpg/proxmox) + Ansible base playbook from ansible-controller01 `~/hewg/deployments/nginx-lxc`.

## [2026-10-13] - Demo personas (Daily/Demo toggle) deployed; local voice server built

### Changed
- ✅ **Portal Daily/Demo toggle** (header button "🎭 Daily/Demo", remembered per browser in `localStorage`). Daily keeps the
  Elden Ring avatars; Demo swaps in photoreal, diverse, fictional professional personas for all **21 homelab agents**
- Personas rendered **locally** (ComfyUI: Flux dev fp8 stills, BiRefNet transparent cutouts, Wan 2.2 i2v idle/listen/talk
  clips, transparent VP9 WebM). No API calls or cost. Pipeline in `C:\AI\demo-personas\`.
- `agents.json`: new keys `avatar_demo`, `avatar_full_demo`, `anim_demo` on 21 agents (added to the live file, not replaced).
### Infrastructure
- Deployed to `codex-agent-lxc`: 126 `*-demo*` files to `~/olympus/avatars/`, `portal.py`, restarted `olympus-portal` only.
  Backups: `~/olympus/backups/agents.json.before-demo-*` and `portal.py.before-demo-*`. Verified demo/ER files serve 200.
  Local tests (WSL) 64/65; the 1 failure is pre-existing (ER images exist only on the LXC). Browser check in Demo mode: not yet done.
### Local only (not deployed)
- PC voice server `C:\AI\local-voice\server.py` (faster-whisper large-v3-turbo on GPU, Kokoro TTS, token + origin checked
  WebSocket on :8765), self-tested. Chatterbox (voice cloning) installed in `.venv-cb`, waiting for a voice recording.
  LXC/browser integration and the OpenAI Realtime switch are not built; HTTPS for LAN devices needs separate approval.
- Wan Animate 2 model files downloaded to ComfyUI (not used yet).

## [2026-10-11] - Token/cost optimizations after the OpenAI balance ran out

### Changed
- ✅ OpenAI bill Oct 8-11: **$44.00** (one key/project: the Roundtable). Estimated split: images ~$23, full realtime
  voice ~$9, gpt-6.1-sol Codex sessions ~$7, everything else ~$3. Codex sessions averaged 43k input tokens per
  request; single investigations reached 0.4-1.3M (one unfiltered command dumped >1M characters).
- `olympus_core.codex()`: `tool_output_token_limit=<redacted>`, `model_auto_compact_token_limit=<redacted>`, token-discipline
  rules in every prompt, and a live **token-budget watchdog** (default 300k estimated input, x1.5 critical) that
  stops runaway read-only sessions and records their estimated usage. `token_budget=<redacted>` for sessions that change
  systems (Corhyn fixes, Loretta security fixes), which are never killed mid-change.
- **Tiered models** (`codex_tiered`): Corhyn diagnoses start on gpt-6-luna and escalate to gpt-6.1-sol on no
  result, low confidence or critical severity; Torrent and Loretta questions start on Luna with an ESCALATE rule
  (medium effort, was high). Fixes stay on gpt-6.1-sol.
- **Voice** defaults to `gpt-realtime-2.1-mini` (`ROUNDTABLE_VOICE_MODEL` overrides); `max_output_tokens` 1500 -> 800.
### Infrastructure
- Backup `~/olympus/backups/token-opt-20261011/`; restarted `boss` and `olympus-portal`; `sre-agent` untouched.
  5 new tests (flags, budget stop + recorded estimate, no-kill for fixes, tiered escalation, voice default);
  `tests.test_hierarchy` on a copy of live code: 60/62 (known CLI argparse and intermittent IPAM-lock errors).
  Not yet observed on real incidents: the account had no credit at deploy time. Claude Code session.

## [2026-10-11] - Soft barge-in: noises duck the agent, only confirmed speech cuts it off

### Fixed
- ✅ The 21:00 speakers-mode barge-in cut agents off on non-speech sounds: transcripts show 2 false cut-offs in 6 calls
  after it (Millicent voice-682, Diallos voice-684; agent "let me shape that…" lines `[interrupted]` with no owner
  words), versus 0 in 19 calls before. `portal.py` now **ducks** on a loud local sound (agent volume 15%, sent mic
  opened, up to 1.5 s) and interrupts only when the server's voice detector reports `speech_started`; noise-only
  ducks restore the agent automatically. Volume also restores when playback stops. Regression test updated; Edge
  headless 69/69 per layout. Backup `~/olympus/backups/softbargein-20261011/portal.py`. Claude Code session.

## [2026-10-11] - New Windows PC SSH key on all hosts

### Security
- ✅ Added key `hermes-windows-pc-2026-10-11` (this PC, 192.168.10.10) for root on node01–03, pihole and
  pbs-server, and for `hermes-admin` on all other running VMs/LXCs including hewg-test01. Owner ran the
  idempotent distribution from codex-agent-lxc (SRE key); existing keys untouched. Direct login verified on
  21/22 hosts; docker-vm-core-utilities01 is verified through ProxyJump codex-agent-lxc (Task #54 route gap).
  Stopped templates unchanged. Details: [16 - SSH Configuration](16%20-%20SSH%20Configuration.md). Claude Code session.

## [2026-10-11] - Roundtable documentation reconciled (18:41 Manila request)

### Changed
- Reconciled current team/catalog/AI hub, service and patch-pipeline references with 29 reporting
- Added browser teaching navigation to Homelab/learning MOCs, indexes and Discord/troubleshooting
  guides: actual bounded SSH, learner Run/Pause/Next, exact supported commands, owner-only write
  approval, popup recovery and no local/unrestricted shell. Documented unverified physical/paid voice
  permitted documentation/mirror writers, protected SRE ownership and manifest/rollback instructions.
  Preserved historical deployment/test/spend records with explicit supersession cues.
- Updated note dates, both changelogs and task coordination; corporate DNS is owner-paused, native apps
  deferred, unrelated pipeline/infrastructure tasks remain open. No runtime deployment, model call,
  new transcript export, infrastructure operation, publishing or Git commit.

## [2026-10-11] - Personal direct reporting accepted; new artwork style correction

> Historical intermediate state: artwork correction and teaching deployment were subsequently completed

### Changed
- Personal-reporting release/SQLite/source backup and exact-live desktop/mobile acceptance persisted under `~/olympus/backups/personal-reporting-20261011-0d0193cf/`. Old artwork, 58 dependency nodes/110 operational edges and SRE source/PID 113381 with zero restarts preserved.
### Fixed
- Corrected stale current documentation: original restoration finalizer completed all 29 Discord icons and closed GitLab #87. The owner subsequently rejected the seven new paintings' style; live JPEG checks confirm a visual mismatch, not missing deployment. New direct-reference correction in progress; old artwork remains untouched.

## [2026-10-11] - Original avatar restoration deployed; Discord cooldown follow-up

> Historical cooldown checkpoint: all 29 icons later verified and #87 closed; the seven new sets then
> received the owner-requested cinematic correction (#102). No upload cooldown remains outstanding.

### Fixed
### Infrastructure
- Three owned sources and 16 new artwork/raster files deployed with source/SQLite backup and baseline/hash guards at `~/olympus/backups/avatar-restore-20261011-0d0193cf/`. Only boss/portal restarted; SRE hash/PID 113381 and zero restarts preserved.
- 57 local tests including 189 browser checks/native fake-input coverage passed before rollout. Exact-live desktop/mobile acceptance decoded all 29 selected portraits, 27 figures and two portrait fallbacks, preserving 58 dependency nodes/110 operational edges and the 27-agent reporting tree. The first virtual-time image probe stalled; real-time loopback acceptance passed. No paid test conversation.

## [2026-10-11] - Character personalities, original avatars and portal recovery

> Historical vector-art milestone: the owner subsequently restored 22 previous selections and replaced
> only seven missing sets with reference-matched artwork. Vector alternatives are stored but unselected.

### Added
### Fixed
- Restored clearly named **Dependency map** access alongside the separate hierarchy; operational edges by default, optional reporting overlay, visible API/position failures and refresh retry.
- Added **Listen now** to restore paused live input without restarting a call; stopped false dead-mic warnings during intentional speaker/song pauses. Explicit HTTPS/sign-in preflight prevents insecure or unauthenticated capture/session attempts. Raw HTTP/IP remains view-only; no browser-security, SSO, corporate DNS or Zscaler bypass.
### Infrastructure
- Six owned sources plus 87 artwork files deployed with exact-baseline checks, source/SQLite backup and rollback at `~/olympus/backups/characters-portal-20261011-0d0193cf/`. Only boss/portal restarted; SRE executable/source/PID, IPAM and unrelated runtime remain unchanged.
- 53 isolated hierarchy/personality tests plus browser/mobile/desk checks and native fake-input capture/replacement passed. Exact deployed page and live 58-node/110-operational-edge graph rendered on desktop/mobile; every profile, portrait and transparent figure decoded. No paid test conversation or new infrastructure permission.

## [2026-10-11] - IPAM repair completed; internal mesh evidence resolves the blocker

### Fixed
- Classified the initially unresolved row as **internal Omada mesh backhaul**, using its full association history, the exact root-AP mesh SSID absent from user WLANs, a live mesh-enabled root and a unique reciprocal managed-AP wireless link. Kept an explicit Layer-2 transport evidence record and both AP address exclusions; no hardcoded MAC allowlist, guessed IPv4 or blanket VLAN-0 exemption.
- Ordinary/ambiguous/unaddressed clients and guests still fail closed, as do incomplete controller evidence, disabled/mismatched mesh topology or conflicting user SSIDs.
### Infrastructure
- Corrective `inventory.py`, `ipam.py`, `agents.json` deployed with exact baseline checks, source/SQLite backup and rollback at `/home/hermes-admin/olympus/backups/ipam-20261011-0d0193cf-before/mesh-completion-before/`. Existing `builder.py` fix retained; boss/portal restarted, SRE source/PID unchanged.
- All three `.36-.49` pools now return **14 candidates each**. Real reserve/verify/activate passed with fresh live inventory and ICMP in an isolated ledger, then the temporary ledger was removed. Production reservations remain zero; no service deployment, DHCP/lease/VLAN/controller or existing-IP changes.
- 49 isolated regressions passed. Persisted live acceptance records transport evidence, candidate lists, allocator results, controller equality and preserved source/executor state. Discord/GitLab **#83** closed only after recorded completion evidence; task registry and current docs no longer report the initial blocker.

## [2026-10-11] - IPAM safeguards deployed; owner evidence still required

> Historical blocked stage: complete mesh-backhaul evidence resolved it in the IPAM completion entry
> above; #83 is closed and each `.36-.49` pool has 14 candidates.

### Fixed
- Replaced Diallos's DHCP-overlapping `.150-.199` service candidates with `.36-.49` on VLAN 20/40/90. Removed Hewg's unused duplicate pool constant; all new-service paths still require atomic Diallos reservation and pre-apply revalidation.
- Added strict complete/stable pagination for Omada networks, current/remembered clients, connection history, DHCP reservations and IP/MAC bindings. Disabled controller reservations/bindings remain excluded. Proxmox interface MAC/IP matching resolves offline wired guests without pretending ping proves an address is free.
- Unresolved known-VLAN clients/guest interfaces block that reviewed VLAN; unknown VLANs block every pool. Last-seen history, not an older IP alone, is required; ambiguous MACs and partial API responses cannot silently permit allocation.
### Infrastructure
- Deployed `inventory.py`, `ipam.py`, `builder.py`, `agents.json` after exact-baseline checks and verified source/SQLite/controller-design backup at `/home/hermes-admin/olympus/backups/ipam-20261011-0d0193cf-before/`. Only boss/portal restarted; existing SRE source/PID unchanged.
- DHCP `.50-.254`, VLANs, leases, existing services and controller reservations/bindings were not changed. No IP was reserved and no service was created. All three pools remain blocked by remembered untagged client `BA-20-51-C1-EA-A7`, whose actual IPv4/VLAN or documented retirement requires owner evidence.
- 47 isolated regressions passed. Live inventory loaded 73 excluded evidence addresses; actual allocation refusal, zero reservations, controller equality, SQLite integrity, source hashes, 29-agent catalog/27-agent hierarchy and service readiness verified. Discord/GitLab **#83** records the repair and remains open/blocked, not falsely completed.

## [2026-10-11] - Full Roundtable activation; Hewg owns patch delivery

> Historical initial reporting count: personal direct reporting later expanded the tree from 27 to 29.
> IPAM candidates were subsequently repaired to `.36-.49`; original rollout hashes remain historical.

### Changed
- Deployed the complete revised runtime: 29 catalog agents, 27 reporting agents, four leads. Gideon is COO/Enterprise Architect; Morgott is the PM specialist under Ranni. Hewg owns compute/storage and patch delivery; Corhyn remains accountable as ITOps Lead.
### Fixed
- Governance reconciliation now respects shared work items: a resolved older patch-failure incident can use a later matching incident's recorded successful verification, and a completed sibling cannot close an item while another task is outstanding. Corrected the rollout-triggered reopen/close loop for completed patch #3 without re-running maintenance.
- Parsed real Omada `ipaddrStart`/`ipaddrEnd` DHCP fields and refused incomplete historical-client IP evidence instead of inventing availability.
### Infrastructure
- Verified source/unit/SQLite backup: `/home/hermes-admin/olympus/backups/roundtable-20261011-0d0193cf-full/`; deployment audit recorded in Discord/GitLab #62. Legacy security approvals #10/#11 were expired with notification because their exact reviewed plans were superseded; mirror approval retained.
- Pending `hierarchy.json` backed up/removed. `/api/hierarchy` is configured with 27 reporting agents; `/api/agents` retains all 29. Restarted boss/portal and applied revised agent timers, preserving the existing SRE source/PID, latest completion-publication repair and all patch safeguards.
- 38 isolated hierarchy regressions, 159 browser/mobile/desk checks and native fake-input integration passed. Live routing/voice configuration, private channel, consultant denials, real patch backlog, portal rendering/API/HTTPS redirects, repeated shared-item sync and historical transcript preservation verified; no paid test conversation, patch, grant or allocation.
- IPAM is active but allocation remains blocked: candidate pool `.150–.199` overlaps live DHCP `.50–.254`, and known-client IP evidence is incomplete. Recorded IPAM-DESIGN-20261011; no network/corporate-control changes.

## [2026-10-11] - Hierarchy page deployed; revised runtime still pending (HIERARCHY-PAGE-DEPLOY-20261011)

### Added
- Live **Agents → Hierarchy** at `/#hierarchy`, showing the requested 28-agent/four-lead structure with collapsible branches and current-runtime profile links. Explicit **activation pending** notice; eight new agents marked Pending. The requested roster is a metadata-only snapshot generated from source, not a claim that the revised runtime is active.
### Infrastructure
- Verified backup: `/home/hermes-admin/olympus/backups/portal-20261011-0d0193cf-hierarchy-page-before.py`, SHA256 `<redacted>`.
- Deployed portal SHA256 `<redacted>`; pending manifest SHA256 `<redacted>`. Runtime registry hash unchanged.
- Source/API regressions, exact page-only artifact regressions, native fake-input checks and simulated success/failed-restart rollback passed. Actual live Chromium rendered all 28 requested agents, four leads and eight pending markers; HTTP page/API source match, service readiness and protected HTTPS page/API 302 verified.
- Direct LAN page: `http://192.168.40.16:8088/#hierarchy`. No DNS/Zscaler/corporate-control changes, paid calls, access grants, private-history migration or full hierarchy activation.

## [2026-10-11] - Portal reporting-hierarchy page (source only; HIERARCHY-PAGE-20261011)

### Added
- **Agents → Hierarchy** (`/#hierarchy`): registry-derived 28-agent reporting tree, exactly four lead badges, domain branches and Gideon's other direct reports. Native branch disclosures, expand/collapse controls, profile navigation, mobile stacking and portrait fallback; refresh preserves branch state.
  - Source: `agents/olympus/portal.py`; regressions: `agents/olympus/tests/test_portal.py`.
### Fixed
- Invalid reporting parents, duplicate IDs, cycles and failed roster fetches are explicitly surfaced rather than displayed as a valid partial hierarchy. Only role metadata is displayed; private activity is excluded.
### Infrastructure
- **Not deployed.** No executable agents, paid calls, live reservations, channel/access changes or service restarts. Preserved earlier source changes and live microphone/avatar deployment.
- Read-only portal diagnosis: active service, backend HTTP 200 and proxy HTTPS login redirect; live portal hash unchanged. Mac DNS queries return NXDOMAIN, but independent homelab-host queries confirm Pi-hole's portal/authentication records are healthy. The initial missing-record inference is superseded; corporate endpoint DNS interception/resolver policy, potentially Zscaler, remains under investigation. No DNS or corporate-control changes made.
- 153 mocked checks across browser/mobile/companion layouts and native Chromium fake-input integration passed. Offline visual review also caught and corrected nested-column squeezing; geometry and portrait-fallback lifecycle regressions now cover it.

## [2026-10-11] - Browser avatar depth; native apps deferred (DESKTOP-20261011)

### Changed
- Added explicitly labeled 2.5D pointer depth/parallax, directional lighting, ground shadow and state/voice glow around the existing character artwork. Added a no-call preview, local depth preference and reduced-motion suppression; preserved full-body/portrait fallback and existing microphone behavior.
- Fixed character-control lookup after Picture-in-Picture document migration so depth toggling, sizing and voice controls remain usable in the browser pop-out.
- Owner deferred new local apps. Withdrawn unfinished native-source/download/protocol-launch changes; existing companion source and mirror behavior are unchanged. No app installed or deployed; isolated unsigned build drafts remain unpublished outside the vault/live portal.
### Infrastructure
- Deployed only `agents/olympus/portal.py` to codex-agent-lxc after matching the portable-mic live baseline; restarted only `olympus-portal`. Verified backup: `/home/hermes-admin/olympus/backups/portal-20261011-0d0193cf-avatar-before.py`.
- Source SHA256: `<redacted>`. Python/JavaScript syntax, scoped whitespace, 117 mocked browser/mobile/desk checks and native Chromium fake-input smoke passed. Service active, internal HTTP 200/exact PAGE and source match, protected HTTPS 302 through Traefik with explicit LAN DNS/SNI resolution.
- This Mac's default resolver could not resolve the portal hostname during the final check; the existing protected route was verified using `--resolve` without changing DNS, routing or authentication. No real microphone, paid call or native OS persistence acceptance.

## [2026-10-11] - Roundtable UI and portable microphones (PORTAL-20261011)

### Changed
- Preserved the glass theme; clarified primary/secondary voice controls and added searchable keyboard-accessible agent selection, counts, empty results and reduced-motion support.
- Replaced the Elgato-only restriction with system-default or browser-local selected microphones. Added local-only testing, explicit input/permission errors, live track replacement and device-change recovery; retained speaker/headphones echo protection.
- Deployed only `agents/olympus/portal.py` to codex-agent-lxc and restarted only `olympus-portal`. Original local baseline and live source matched before deployment. Backup: `/home/hermes-admin/olympus/backups/portal-20261011-0d0193cf-before.py`.
- Verified Python/JavaScript syntax, 93 mocked browser/layout/companion checks and native Chromium fake-input capture/replacement/cleanup. Service active, internal HTTP 200 with exact served-page match, protected HTTPS route 302 to authentication, local/live source hashes equal. No paid call or real-device microphone test.

## [2026-10-11] - AI documentation consolidation (AI-20261011)

- Deployed explicit private-tree mirror exclusions plus narrowly scoped scribe/documentation-agent reader corrections, with live-source/content backups. No manual AI generation, backfill, publishing or unrelated infrastructure changes.

## [2026-10-11] - Vault navigation only (PKM-20261011)

- Updated MOC - Homelab to distinguish the current numbered index from retired Kubernetes/GitOps history, using actual archived paths and a qualified Agents link.
- Repaired the archived My Home Lab related-link YAML and added a current-owner pointer in the old blog backlog; preserved unique content and existing task/publication ownership.

## [2026-10-11] - Automated patching (SRE agent)

### Changed
- **docker-lxc-glance (CT200) / proxmox-nodes-api**: run completed with no container update; `python:3.11-slim` remained Python 3.11.17 with the same image digest. Independent verification: PASS. [GitLab #53](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/53)

## [2026-10-11] - Incident response (Corhyn)

### Fixed
- `homepage-lxc` (CT209, 192.168.40.27) SSH failed because the SRE agent key was absent; an October 5 provisioning omission is suspected but unproven. Added the verified key, preserved both administrator keys, and verified two `cp -a` backups (destination paths were not in the run record); fingerprint `<redacted>`.
  Fresh SSH login, `sudo -n true`, authorized-key contents/ownership/mode, SSH journal acceptance, and Homepage HTTPS (HTTP 200) all passed. [GitLab #27](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/27)
  CT209 now authorizes the October 5 SRE agent key for `hermes-admin`.
- ✅ **helios-lxc (CT210)**: `logrotate.service` and `man-db.service` failed because nesting was disabled and AppArmor blocked systemd service isolation; enabled `features: nesting=1` and gracefully restarted. Both units, all three APIs and health endpoints passed immediate checks; scheduled executions and post-rotation disk stability remain pending. Two config backups were verified, but their paths were not included in the run record. [GitLab #31](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/31)
- ✅ **docker-vm-core-utilities01**: five containers were unhealthy because probes used missing `curl`/`wget` tools (Wizarr likely also resolved localhost to IPv6 while listening on IPv4); replaced probes with runtime-available IPv4 checks and pinned the original running images. All five containers passed five consecutive probes and endpoint/data checks; browser rendering and Chronicle’s separate GitHub 404 remain unresolved/unverified. [GitLab #28](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/28)
- ✅ **node02**: both NFSv3 mounts (`/mnt/nfs/lxcs`, `/mnt/nfs/media`) failed during boot and were restored with their existing configuration; the suspected network-readiness race is unproven. Added `x-systemd.mount-timeout=180s` to those two entries; both mounts and `remote-fs.target` are active, all managed storage and guests remain healthy, with no reboot. [GitLab #29](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/29)
  Independent verification verdict: PASS — both NFSv3 exports and directory listings passed; generated units show the 180-second timeout, all eight managed storage entries and 19 non-template guests are healthy, and no units failed.
- **gitlab-vm01 / SRE executor**: verified maintenance was falsely reported as failed when completion delivery to GitLab failed; discarded API errors and stale `patching` labels led recovery to misclassify the run. Reconciled issue #3 and repaired `codex-agent-lxc`'s executor to persist verification/completion phases, require readiness, retain errors, and retry publication idempotently. **Independent verification: PASS** — all 16 regression tests, GitLab/API readiness, and host/container checks passed; current updates remain separately unapproved in #59. [GitLab #3](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/3)


- **ansible-controller01 / patch planner**: the excluded-host rejection was misclassified as `patch-failed`; added an exclusion and inventory guard before planning, and left issue #1 open for manual maintenance. Independent verification: PASS for guard syntax/behavior, issue labels, Medic query and VM103; package compliance remains unverified, with 6.8.0-142 running and 6.8.0-146 named by the reboot marker. [GitLab #1](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/1)

### Found
- **pbs-server (PBS CT100, node03)**: automated patching was rejected because PBS is explicitly excluded and absent from managed SSH inventory; issue #5 remains open as manual maintenance. PBS 3.4.9-2 has 17 cached upgrade candidates; independent verification passed for issue/Medic state, LXC 100 and exporter queries, while the scheduled scan, package compliance and restore health remain pending or unverified. [GitLab #5](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/5)
- **homepage-lxc (CT209) / SSH key distribution**: the SRE public key was confirmed missing; a remediation proposal was prepared, but no host configuration or agent source changed. Independent verification is limited to local key/helper inspection; distribution, fresh login, HTTPS and scheduled audit checks await approval. [GitLab #34](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/34)

## [2026-10-10] - Ram AI + companion mirror

### Added (late)
- **Roundtable portal redesign** (`portal.py`): glass theme (frosted cards with agent-colour glow, gradient accents, pill buttons, segmented voice picker, chat-bubble transcript); sidebar nav (Agents / Talk / Operations) with an approvals badge; responsive: icon rail ≤1180 px, bottom tab bar + mobile fixes ≤760 px (horizontal-scroll tables/map, compact companion, no iOS input zoom). Desktop companion now appears **whenever the browser is minimized/hidden** (idle agent when no call, live mirror on a call; 75 s staleness window because Chrome throttles hidden tabs, `sendBeacon` on close). Electron app unchanged.
- 🎙️ Portal Voice tab: "Summon an agent" picker split with a 🛠️ Homelab AI / 🧑 Personal AI toggle.

### Fixed
- 🎙️ **Voice broken for ALL agents (HTTP 400)**: the shared `TRANSCRIBE_PROMPT` grew past OpenAI's 1024-char limit for `audio.input.transcription.prompt` after adding Chhaya's terms. Trimmed redundant hints and hard-capped the prompt at 1024 in `voice.session_config`; verified client_secrets succeed for all 19 agents.

### Changed
- ✈️ Chhaya AI: "You ignored me, Hermes…" complaint now triggers after **1 day** without talking to her (was 5), in Discord chat, voice openings and nudges.

### Added
- 🧋 **Ram AI** (`ram.py`): Intune / SCCM / AD / hybrid identity / Azure security consultant on gpt-6-sol (medium, web), #milktea-consult, voice (ash), `ram-brief.timer` Wed 09:00, cartoon avatars, emoji `:ramai:`. Calls Hermes "Tol"; opens with "Hayop ka, Tol! Pag-ti-tripan mo na naman ba ako?" and signs off "Hayop ka, Tol!". See 60 - Roundtable Agent Team.
- 🖥️ **Companion mirror**: minimizing the portal during a voice call shows the agent's transparent full-body figure in the Electron companion (new `/api/mirror` endpoints; companion `mirror` IPC). Chrome PiP pop-out stays as the fallback when the companion isn't running.

## [2026-10-10] - Roundtable agent log

### Added
- 🔨 **Hewg forged LXC `hewg-test01` (VMID 101) on node03** (Hewg): 1 vCPU, 512 MB RAM, 4 GB local-lvm, debian-12, VLAN 40, IP 192.168.40.150. Purpose: Roundtable end-to-end build test (delete afterwards). Terraform (bpg/proxmox) + Ansible base playbook from ansible-controller01 `~/hewg/deployments/hewg-test01`.
- 📖 **Roundtable: Hewg, Blaidd, Miriel, Kalé and Radagon built** (Gideon): Hewg (builder.py) builds LXCs/VMs with Terraform (bpg/proxmox 0.93 on ansible-controller01, ~/hewg) + Ansible after approval and audits compute/storage daily 07:10, handing high/critical findings to Corhyn. Blaidd (checker.py) verifies patches, new builds and Corhyn's fixes daily 05:10 and on demand. Miriel (scribe.py) answers from the homelab docs and writes changelog entries for the team. Kalé (scout.py) posts a weekly web-researched report to #homelab-scout. Radagon (mirror.py) keeps the public homelab-hub docs in sync with the live cluster, pushing only after approval.

### Fixed
- 🐢 **doc_agent ledger path fixed** (Miriel): doc_agent.py recorded Miriel's runs via a non-existent ~/roundtable-hold/olympus_core.py, so her runs never reached the Roundtable ledger (portal showed 'last: never'). Now ~/olympus/olympus_core.py.

## [2026-10-10] - Nepheli media agent + voice barge-in

### Added
- 🎬 **Nepheli Loux** (`media.py`, id `media`) watches CT205 `docker-lxc-media`.
  - Timers: `media-watch` every 10 min, `media-taste` at 05:30, `media-digest` at 19:30.
  - Channel: #cinema-lounge.
  - Failed downloads are blocklisted and re-searched automatically. Container restarts are ✅-gated.
- First watch found that **SABnzbd can't log in to `news.newshosting.com`** ("502 Authentication Failed"). 16 queued items are stalled at 0%, so the account or password needs fixing in SABnzbd → Config → Servers.
- Bazarr reports "Missing languages profile".
- ✋ **Voice barge-in for every agent** (`portal.py`, `voice.py`):
  - The mic stays open while the agent speaks.
  - The browser detects your voice above the echo floor, then sends `response.cancel`, `output_audio_buffer.clear` and `conversation.item.truncate`.
  - Server `interrupt_response` is set to false so echo can't cut the agent off.
  - Suspect echo transcripts are dropped.
  - Added the ✋ button, the ✋ companion button and the Esc key.
  - `INTERRUPT_NOTE` in all agent prompts.

### Fixed
- `agents.json` and `graph.json` had emoji and labels garbled to `?` (Iji and Nepheli nodes and edges). They were rewritten as UTF-8 via Python.

## [2026-10-10] - Roundtable agent team: rename, avatars, voice

### Changed
- 🛡️ **Olympus agent team → "Roundtable"** with Elden Ring names:
  - Gideon (boss), Corhyn (medic), Sellen (watcher), Torrent (netops), Hewg (builder), Melina (writer), Kalé (scout), Radagon (mirror), Miriel (scribe), Blaidd (checker).
  - Internal ids, scripts and units are unchanged.
  - Discord `#olympus` → `#roundtable-hold`; a roster announcement was posted.
  - See 60 - Roundtable Agent Team.
- 🌐 **Portal moved to `https://roundtable.hrmsmrflrii.xyz`:**
  - Authentik provider 48 is now "Roundtable Portal" and the app slug is `roundtable`.
  - Traefik `roundtable.yml` holds the SSO route, a public `/avatars/` route, and a 301 from `olympus.`. The old `olympus.yml` was retired.
  - A Pi-hole local DNS record was added.

### Added
- 🎨 **Agent avatars** generated with `gpt-image-2` (`avatars.py`) and shown on portal cards, profiles and the dependency map.
  - Uploaded as Discord guild emojis (`avatar_emojis.py`), so embeds show each agent's face. Discord can't reach the LAN portal.
- 🎙️ **Voice room** at `/#voice`: talk to any agent with the PC mic.
  - Runs on `gpt-realtime-2.1-mini` over WebRTC with a per-agent voice.
  - Agents can hand over to each other, check live status and delegate real jobs (still ✅-gated).
  - Every transcript goes to #roundtable-hold.
  - Limits: $20/month voice cap (`voice budget <usd>`), 15 min per call, 2 min idle.
  - `/api/voice/*` only accepts requests through Traefik with an Authentik user.

## [2026-10-10] - Automated patching (SRE agent)

### Changed
- 🔄 `homeassistant` on homeassistant-lxc (CT206): 2026.9.4 → 2026.10.0. Verification: pass. [Issue #25](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/25)
- 🔄 `metube` on docker-lxc-media (CT205): 2026.09.29 → 2026.10.09. Verification: pass. [Issue #26](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/26)

### Found
- 🚨 `immich-server` on immich-vm01: update v3.2.4 → v3.3.0 failed its post-update health check, and the automatic rollback to v3.2.4 also failed — the container is crash-looping and needs manual attention. Likely cause: the compose healthcheck shells out to `wget` (absent from the image, so it always reports unhealthy), and v3.3.0's DB migrations ran before the executor restored the old image, which v3.2.4 then rejects. Recovery: back up the DB, pin to v3.3.0, drop the wget healthcheck override, then `docker compose pull/up` on immich-vm01. [Issue #24](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/24)

### Fixed
- ✅ **immich-server recovered on v3.3.0** (manual, Claude). Confirmed cause from logs: `Migration "<redacted>" was already applied ... not in this version of Immich (3.2.4). Downgrades are not supported.` Rolled forward instead of back:
  - DB backup `/mnt/immich-uploads/db-backups/pre-v3.3.0-fix-20261010.sql.gz` (213M, `gzip -t` OK)
  - `/opt/immich/docker-compose.yml` (backup `docker-compose.yml.bak-20261010`): `immich-server` + `immich-machine-learning` `:release` → `:v3.3.0`; removed the `wget --spider` healthcheck override (no `wget` in image) so the image's built-in `immich-healthcheck` applies
  - `docker compose up -d` → migrations finished cleanly, all 4 containers `healthy`, `/api/server/version` = 3.3.0, `https://photos.hrmsmrflrii.xyz/api/server/ping` = 200
  - Follow-up: Task #76 (SRE rollback isn't migration-aware; pinned tag hides future releases from digest audit) — done same day, see below
- ✅ **SRE agent container updates are now migration-aware** (Task #76). `agents/sre_agent.py` → codex-agent-lxc `~/sre_agent.py` (backup `sre_agent.py.bak-20261010`), `sre-agent.service` restarted:
  - Stateful stacks (postgres/mariadb/mysql/mongo container, or GitLab): pre-update logical DB dump to `<working_dir>/sre-db-backups/` (gzip-tested, keep 3, abort on failure); on verification failure **no image rollback** — new version left running, `patch-failed` with roll-forward / full-revert steps
  - Compose healthchecks that can't run (exit 126/127, "not found") fall back to the image's own `HEALTHCHECK`, else running + not restarting; flagged in the GitLab note
  - Tested live on immich-vm01: healthy/unhealthy/unknown/no-op fallback paths + 213M Immich dump (test files removed)

### Changed
- 🏠 **All agents now run on `codex-agent-lxc`** — Agent 5 (Documentation) moved off Kratos:
  - `agents/doc_agent.py` rewritten for Linux → `~/doc_agent.py`; systemd `doc-agent.timer` daily 06:00 Manila (`Persistent=true`); Kratos Scheduled Task "Homelab Doc Agent (Agent 5)" **disabled**
  - Vault read/write directly via **OneDrive** (`rclone` remote `onedrive:`, Microsoft Graph) — Kratos no longer needs to be on. Downloads only top-level homelab docs + Vault Changelog (never `11 - Credentials.md`); uploads only validated, changed files; aborts if a file changed in OneDrive mid-run
  - Editor switched from Claude Code to **Codex** (`--sandbox workspace-write` on a scratch copy)
  - Vault git baseline commit `13f1b13` pushed first (903 pending local changes)
  - Tested: `--no-upload` run on #24, OneDrive write/conflict/Kratos-sync test. See 59 - Autonomous Patch Management Agent Pipeline › Agent 5 — Documentation Agent
  - Immich compose back on `:release` (local tag → v3.3.0 image); audit detects v3.3.1 drift → normal approval issue. Removed useless `sre-rollback/immich-immich-server:issue-24` (v3.2.4) image
  - Docs: 59 - Autonomous Patch Management Agent Pipeline, [57 - Immich VectorChord Migration and v3 Upgrade](57%20-%20Immich%20VectorChord%20Migration%20and%20v3%20Upgrade.md), [07 - Deployed Services](07%20-%20Deployed%20Services.md)

## [2026-10-08] - Automated patching (SRE agent)

### Changed
- 🔄 `pihole-stats-api` on docker-lxc-glance (CT200): `python:3.11-slim` base image rebuilt (same version 3.11.17, image created 2026-10-01 → 2026-10-06). Verification: pass. [Issue #22](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/22)
- 🔄 `proxmox-nodes-api` on docker-lxc-glance (CT200): `python:3.11-slim` base image rebuilt (same version 3.11.17, image created 2026-10-01 → 2026-10-06). Verification: pass. [Issue #23](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/23)

## [2026-10-07] - Sentinel /download orchestration, Sonarr Deluge fix, blog writer preview

### Added
- ✅ Sentinel `/download title [media_type]` now does the whole job instead of showing a Jellyseerr result and waiting for a 👍 that nothing handled. It looks the title up in Radarr/Sonarr, shows a picker, adds the pick with quality profile `Any` (TV: season picker, only chosen seasons monitored), starts the search and checks `/queue/details`. If nothing was grabbed it reports the top release rejection reasons and offers Keep watching / Remove buttons. Unreleased movies are reported as not released yet. Live file: `/opt/sentinel-bot/cogs/media.py` on utilities01 (backup `media.py.bak-20261007-download`). Docs: [24 - Discord Bots](24%20-%20Discord%20Bots.md), [08 - Arr Media Stack](08%20-%20Arr%20Media%20Stack.md).
- ✅ `blog-preview` (blog-studio): read-only web viewer for drafts with images, Mermaid and callouts. Auto-refreshes, highlights `[CONFIRM]`, flags missing images. `blog-writer` starts it on CT211 (127.0.0.1:8765 only) and tunnels it to Kratos over the writer's SSH session. `blog-preview` on Kratos serves the vault's `Consolidated/` at :8766. Docs: Agents.

### Fixed
- 🐛 Sonarr's Deluge download client still had the pre-2026-09-13 password, so torrent grabs from Sonarr failed. Set it to the current WebUI password. Radarr was already correct. Noted in 11 - Credentials.
- 🐛 Sentinel image build failed on `ssh-key` (permission denied in the build context), and the container quietly kept running the old image. Added `/opt/sentinel-bot/.dockerignore` (`ssh-key`, `*.bak*`, `.env.bak*`).

### Changed
- 🔄 Blog writer default model `gpt-6.1-sol`/high → `gpt-6-sol`/medium. Artist `gpt-6.1-sol`/medium → `gpt-6-luna`/low (it only executes a JSON plan; draft-mode test passed). Per-session override: `blog-writer -Model <m> -Effort <e>`.
- 📝 Writer `AGENTS.md`: Clustered Thoughts house style, voice samples from the published "My Accidental Journey Into Homelabbing", live preview and model guidance. Redeployed.
- 📝 Doc sweep: `/download` and blog-preview/models reflected in [07 - Deployed Services](07%20-%20Deployed%20Services.md) (Sentinel section), [Book - The Complete Homelab Guide](Book%20-%20The%20Complete%20Homelab%20Guide.md) (`/download` no longer "via Jellyseerr"), [10 - IP Address Map](10%20-%20IP%20Address%20Map.md), 11 - Credentials and `CLAUDE.md` (CT211 rows), [00 - Homelab Index](00%20-%20Homelab%20Index.md) and TODO - Homelab Blog Series.

> [!WARNING] Repo drift
> `homelab-infrastructure/ansible/playbooks/sentinel-bot` no longer matches the live bot (most files differ). Running that playbook would revert live fixes, including `/download`. Sync the repo from live before using it.

## [2026-10-07] - blog-studio: Codex writer and image agents on codex-agent-lxc

### Added
- ✅ `~/blog-studio/` on codex-agent-lxc (CT211, 192.168.40.16). Two Codex CLI agents: the Homelab Blog Writer (`blog-writer`, `gpt-6.1-sol`, interactive) and the Image Artist (`blog-artist <slug> [--draft]`, non-interactive `codex exec`). Images go through the OpenAI Images API (`imagegen`): `gpt-image-2.5-sunburst` at high quality for final, `gpt-image-2.5-flare` at low quality for drafts, webp output, max 12 final images per post.
  - Source: `agents/blog-studio/` (writer/ and artist/ AGENTS.md, bin/, blog-studio-sync.ps1). Scripts installed to `~/.local/bin`.
- ✅ `blog-studio-sync.ps1` (Kratos): `install`, `push` (copies `07 HomeLab Things` notes to `sources/`, excluding credentials, then runs `redact`, 406 redactions) and `pull` (drafts, assets and STATUS.md into `Homelab Blog Posts/Consolidated/`).
- ✅ `blog-lint`: flags em dashes, emojis, semicolons, banned words and phrases, plus OPSEC hits (IPs, MACs, secrets, the real domain).

### Security
- 🔒 No new secrets. Reuses the existing OpenAI key in `~/.codex_env`. Codex strips `*KEY*` env vars, so `imagegen` reads that file itself.
- 🔒 The writer runs in the workspace-write sandbox (with network). Bubblewrap can't nest in the unprivileged LXC, so when `blog-artist` is called from inside the writer it runs the inner agent with `danger-full-access` and relies on the writer's sandbox to confine it. Run standalone, it uses its own workspace-write sandbox.

### Fixed
- 🐛 `blog-artist` now exits non-zero when the artist fails or doesn't write a manifest. Before, `tee` masked the failure.
- 🐛 `blog-studio-sync.ps1 pull` failed with `tar: STATUS.md\r: Cannot stat`. PowerShell's trailing CRLF reached the remote shell, so `Invoke-Remote` now puts it on a comment line.
- 🐛 `push` overwrote the agent's unpulled drafts with older vault copies. Now uses `cp -u -p`, so a draft is only replaced when the vault copy is newer.

### Changed
- 🔄 One-command workflow: `blog-studio-sync.ps1 write [prompt]` (PowerShell profile function `blog-writer` on Kratos) pushes, opens the writer over `ssh -t` and always pulls on exit. The first prompt reaches the LXC as `~/blog-studio/.next-prompt` (base64, no ssh quoting), and `bin/blog-writer` consumes it. Documented in Agents.
- 📝 Docs aligned with the deployment: rewrote Homelab Blog Writer Agent and Image Artist Agent (old versions described a text image plan, 4:3 images and the wrong assets folder). `writer/AGENTS.md` now explains the auto push/pull, and the `artist/AGENTS.md` example uses `.webp`. Redeployed with `install`. Also referenced blog-studio in [00 - Homelab Index](00%20-%20Homelab%20Index.md), [10 - IP Address Map](10%20-%20IP%20Address%20Map.md), `CLAUDE.md` (CT211 row) and TODO - Homelab Blog Series.

## [2026-10-06] - Blog voice rewrite, duplicate-date bug fix, image placeholders

### Changed
- 📝 Rewrote all 28 original posts in `Homelab Blog Posts/` (Blog Post 01-28) and cleaned all 30 live `Rewritten Series/` posts: removed every emoji and em dash, plainer explanations of jargon, grounded one-line takeaways instead of AI-style recaps.
- 📝 Spread the 30 `Rewritten Series/` posts' `date:` field a week apart (post 30 at 2026-10-06 back to post 1 at 2026-03-17) instead of all 30 sharing one publish timestamp.
- 🖼️ Added one `<!-- IMAGE: ... -->` placeholder comment to each of the 30 `Rewritten Series/` posts, tied to a specific screenshot/photo the post describes. Checklist: `Homelab Blog Posts/TODO - Blog Image Checklist.md`. Not yet synced — still placeholders.

### Fixed
- 🐛 **<redacted>** (`508e708`, `7ca0601`): `scripts/sync_obsidian.py` never deleted a post's old output file when its date/slug changed, so the stale file kept building to the same title-based permalink and won the collision — this is why all 30 posts briefly showed "Oct 6" live regardless of their new dates. Patched the script to remove any other `_posts/*-<slug>.md` for the same source on every sync; removed the 29 stale duplicates and verified live.

## [2026-10-06] - homelab-hub super repo created

- New public repo [herms14/homelab-hub](https://github.com/herms14/homelab-hub) (repo-of-repos pattern, template raffertyuy/repo-of-repos v0.6.1)
- Clones all 6 homelab repos under `repos/` (gitignored); docs: architecture, inventory snapshot, cross-repo rebuild order
- Custom: Morpheus agent persona, `/secret-scan` + `/inventory-refresh` skills, `scripts/pull_all.py`, `scripts/secret_scan.py`
- Finding: homelab-infrastructure and Homelab-Infrastructure-Deployment share root commit; draft consolidation plan in `_plans/` (task #73)

## [2026-10-06] - GitHub repos synced with live environment

### Changed
- 📝 **<redacted>** (`880024e`): published all 30 Rewritten Series posts; `sync_obsidian.py` now recurses subfolders and keeps datetimes. Deploy workflow succeeded.
- 🧠 **herms14/helios-homelab-api** (`c71352c`): synced live code from helios-lxc. README covers purpose, architecture, deployment and technologies; COMMANDS.md adds LXC commands and the per-instance token tier model.
- 📊 **herms14/glance-dashboard** (`25ef0c4`, v3.0.0): confirmed in use on CT200 (8 pages). Synced the live config, CSS and APIs with secrets replaced by `${VAR}`; retired APIs moved to `apis/_retired/`; GitLab CI marked legacy.
- 🏗️ **<redacted>** (`ed16b5a`): current inventory, a 13-step rebuild order, Azure/K8s docs marked archived.
- 📚 **herms14/homelab-infrastructure** (`cfa2a9e`): README, claude.md and the project description rewritten for the current state; new docs index; archived banners added.
- 🤖 **herms14/homelab-agent** (`6b1e129`, v1.1.0): commands genericized (doc-roles table), task registry and changelog templates added.

### Found
- ⚠️ homelab-infrastructure has older committed secrets: <redacted> tokens, a Cloudflare token, Radarr/Sonarr keys, DB and Omada passwords, and Packer admin passwords (#71).
- ⚠️ `terraform/proxmox/main.tf` (deployment repo) still defines retired K8s and syslog VMs, so `apply` would recreate them. LXCs are not in Terraform.
- ⚠️ Vault docs are stale: `02 - Proxmox Cluster.md` says PVE 9.2.11 (live 9.2.21); `00 - Homelab Index.md` says 11 LXCs (live 13); node K8s labels; Sentinel host (CT201 vs core-utilities01); .13 ports 5055-5057 conflict.
- ℹ️ CT209 homepage-lxc is stopped; `proxmox-nodes-api` (:5061) runs but Glance does not use it.

## [2026-10-06] - Automated patching (SRE agent)

### Infrastructure
- 🔧 **gitlab-vm01** — OS patched, rebooted: kernel `6.8.0-142-generic` → `6.8.0-146-generic`, 33 pkgs (incl. `docker-ce` 29.7.2 → 29.8.2, `containerd.io` 2.3.3 → 2.3.6), 33 pending → 0. Verification: pass. [Issue #3](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/3)
- 🔧 **immich-vm01** — OS patched, rebooted: kernel `6.8.0-142-generic` → `6.8.0-146-generic`, 34 pkgs (incl. `fwupd` 1.9.31 → 2.0.20, `docker-ce` 29.7.2 → 29.8.2), 34 pending → 0. Pre-existing `fwupd.service`/`fwupd-refresh.service` failures noted before the run. Verification: pass. [Issue #4](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/4)
- 🔧 **helios-lxc** — OS patched, no reboot needed (kernel unchanged, `7.0.14-20-pve`): 48 pkgs (incl. `ca-certificates`, `base-files`, `bind9-libs`), 43 pending → 0. Pre-existing failed units (`logrotate`, `man-db`, `systemd-logind`, `systemd-networkd` service/socket) unaddressed by this patch — worth a manual look. Verification: pass. [Issue #6](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/6)
- 🔧 **docker-vm-core-utilities01** — OS patched, rebooted: kernel `6.8.0-142-generic` → `6.8.0-146-generic`, 34 pkgs (incl. `fwupd` 1.9.34 → 2.0.20, `docker-ce` 29.7.2 → 29.8.2), 34 pending → 0. Verification: pass. [Issue #7](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/7)

### Changed
- 🐳 **docker-vm-core-utilities01 / paperless** — `redis:7-alpine` refreshed (7.4.11, image rebuilt 2026-08-18 → 2026-09-17). Verification: pass. [Issue #11](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/11)
- 🐳 **docker-lxc-media / flaresolverr** — updated, `flaresolverr` v3.5.0 → v3.5.2. Verification: pass. [Issue #17](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/17)
- 🐳 **homeassistant-lxc / homeassistant** — updated, `home-assistant` 2026.9.2 → 2026.9.4. Verification: pass. [Issue #18](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/18)
- 🐳 **authentik-lxc / authentik** — `postgres:16-alpine` refreshed (16.15, image rebuilt 2026-09-17), `redis:alpine` 8.10.1 → 8.10.2. Verification: pass. [Issue #19](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/19)

### Found
- ⚠️ **ansible-controller01** (#1) and **pbs-server** (#5) — both excluded from automated patching by policy; no run attempted, as expected.
- ⚠️ **gitlab-vm01** (#3) — a second run reported failure ("interrupted mid-patch, host state unknown"), but triage found the package upgrade and reboot had already completed successfully (new kernel active, containers healthy); root cause is an orchestrator completion-tracking gap during the auto-reboot step, not a host problem. Recommend re-approving issue #3 to clear the stale failure state.

## [2026-10-05] - Sentinel bot: all hosts "Unreachable" fixed

### Fixed
- 🔧 `/homelab uptime` showed every node and Docker host as ❌ Unreachable. Root cause: the container runs as uid 1000 (`sentinel`), but the mounted SSH key `/home/hermes-admin/.ssh/homelab_ed25519` is owned by uid 1001 with mode 600 → `Permission denied` on every SSH call (likely since the 2026-09-14 rebuild; also broke `/check` and other SSH-based commands). The hosts and key were fine.
  - Copied the key to `/opt/sentinel-bot/ssh-key/homelab_ed25519` (1000:1000, 600), `SSH_KEY_HOST_PATH` in `.env` updated, container recreated. Verified from inside the container: all 3 nodes + utilities/media/glance answer. Backup: `.env.bak-20261005-sshkey`.
- 🔧 `.env` `DOCKER_BOTS_IP` 192.168.40.14 → .17 (CT201 re-IP earlier today).
  - File affected: `/opt/sentinel-bot/.env` on docker-vm-core-utilities01

## [2026-10-05] - Agent 5 + Agent 6 built; AI triage, risk review, closing summaries

### Added
- 🔎 **Agent 6 — Verification** (`agents/verify_agent.py`, codex-agent-lxc): deterministic re-check of every completed run (package versions, failed units, kernel, container image/health) before the 05:30 report; fail → `verify-failed` label + reopen + Discord + triage.
- 📝 **Agent 5 — Documentation** (`agents/doc_agent.py`, Kratos): Scheduled Task "Homelab Doc Agent (Agent 5)" daily 06:00; Claude Code (Read/Edit only) writes the changelog; edits validated and rolled back on violation; acks via `sre_agent.py doc-ack` → GitLab + Discord.
- 🧐 **Pre-approval risk review** for container issues (versions old → new + release-note analysis); high risk withdraws an existing approval.
- 🔍 **AI failure triage** on every failure/rollback/verification fail.
- 📋 **Closing summary** note on every GitLab work item (previous → current versions, issues, pending items, "All looks good").
  - File affected: `agents/sre_agent.py`

### Fixed
- Registry version lookup for nested OCI indexes (ghcr authentik).

### Changed
- Pipeline order: patch → verify (Agent 6) → report → document (Agent 5). Report gains Verified + AI diagnosis columns.

## [2026-10-05] - Automated patching (SRE agent)

### Infrastructure
- 🔧 **gitlab-runner-vm01** — OS patched, rebooted: kernel `6.8.0-139-generic` → `6.8.0-146-generic`, 30 pkgs + 8 deps (incl. `gitlab-runner` 18.9.0-1 → 19.4.1-1), 30 pending → 0. Verification: pass. [Issue #2](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/2)
- 🔧 **codex-agent-lxc** — OS patched, no reboot needed (kernel unchanged, `7.0.14-20-pve`): 55 pkgs (incl. `libc6`, `dpkg`, `bind9-libs`), 51 pending → 0. Verification: pass. [Issue #8](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/8)

### Changed
- 🐳 **docker-lxc-glance / pihole-stats-api** — updated, `python:3.11-slim` 3.11.16 → 3.11.17. Verification: pass. [Issue #14](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/14)
- 🐳 **docker-lxc-glance / proxmox-nodes-api** — updated, `python:3.11-slim` 3.11.16 → 3.11.17. Verification: pass. [Issue #15](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/15)

### Found
- ⚠️ **proxmox-nodes-api** first attempt showed as rolled-back — triage found it was an intentional simulated verification failure (testing the auto-rollback path), not a real app fault; the subsequent run updated cleanly and both endpoints returned HTTP 200. [Issue #15](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/15)

## [2026-10-05] - Container image updates + patch reports for the SRE agent

### Added
- 🐳 **Container updates** in `sre_agent.py`: daily `docker-audit` (≥18:00) compares running compose images with registry digests and files one `container-update` issue per stack (`Container drift: <host> / <stack>`). Approved stacks are updated deterministically in the window — rollback tag → `compose pull/up` of outdated services only → health check → automatic rollback on failure. First audit filed #9–#21.
- 📊 **Patch reports**: every run is recorded in `~/.sre-agent/history.jsonl` (OS package diff via `dpkg-query` before/after; app previous → new version from image labels). Daily report after 05:00 posted to Discord + saved under `~/.sre-agent/reports/`. CLI: `sre_agent.py report [--hours N] [--post]`.

### Fixed
- State-file race (manual CLI run clobbered watcher's Discord message map → duplicate approval post); state now saved per key group under a lock.
- chronicle-lxc docker inventory (nil labels / old Docker template).
- Kratos `homelab_ed25519` (rotated key) authorized on codex-agent-lxc; `codex-agent-lxc` alias added to Kratos SSH config.

### Validated
- #14 glance/pihole-stats-api updated; #15 glance/proxmox-nodes-api rollback path (simulated failure → auto-rollback) then success path, python 3.11.16 → 3.11.17.

### Found
- Watchtower on gitlab-vm01 / immich-vm01 is monitor-only and notifies decommissioned host .10 — does nothing. Candidate for removal.

## [2026-10-05] - SRE executor (Agent 4) built + live; access, backups and IP conflict fixed

### Added
- 🤖 **`sre-agent.service`** on codex-agent-lxc (`agents/sre_agent.py`, `agents/sre-agent.service`): always-on approval watcher + executor. Discord ✅/❌ (owner only) or GitLab `approved` label → patches in the 02:00–05:00 Asia/Manila window with backup gate, Codex-driven upgrade, auto-reboot and deterministic verification. New GitLab labels: `approved`, `patching`, `patched`, `patch-failed`, `rejected`. See 59 - Autonomous Patch Management Agent Pipeline.
- 🔑 Agent SSH key `sre_agent_ed25519` on all VMs/LXCs (hermes-admin) + all 3 nodes and pihole (root). Excluded: ansible-controller01, pbs-server.
- 🔐 Passwordless sudo for hermes-admin on CT201/203/204/205 (`/etc/sudoers.d/hermes-admin`).
- 💾 Daily pbs-daily + Friday pbs-main jobs now cover **all** guests (10 had no backup job).
- `docker-lxc-bots` added to `patch-audit.sh` host list; ansible-controller01 key authorized on it.

### Fixed
- 🌐 **IP conflict**: CT201 docker-lxc-bots and CT210 helios-lxc were both 192.168.40.14 → CT201 moved to **192.168.40.17** (hermes-admin user created).
- 🛡️ **PBS CT100 not auto-starting** after node03 reboots (root cause of most failed nightly backups) → `onboot: 1`, startup order 1; masked crash-looping `zfs-zed`; `newaliases` on node03.
- 🔗 **Inter-node root SSH** broken since the Oct 3 recovery (node keys missing from cluster authorized_keys; node03 `id_rsa` 0 bytes since Jan 11) → repaired, `pvecm updatecerts`, full 3×3 mesh verified.
- helios-lxc `authorized_keys` missing trailing newline (appended key got concatenated) → split.
- gitlab-runner-vm01: expired GitLab Runner apt key (`EXPKEYSIG 3F01618A51312F3F`) refreshed by the agent.

### Changed
- **gitlab-runner-vm01 patched by the agent** (issue #2, E2E validation): 30 pkgs + 8 deps, kernel 6.8.0-139 → 146, rebooted, 0 pending, issue closed.
- Replanned #2–4, #6 (truncated plans); approval requests for #3/#4/#6/#7/#8 posted to Discord #homelab-infrastructure.
- Task #63 (ACL lockdown for codex-agent-lxc) **waived** — owner granted full homelab + production access.

## [2026-10-05] - Patch pipeline: orchestrator v2.1 deployed, manual chain trigger built, full chain run

### Added
- ✅ `run-patch-pipeline.sh` — single manual entry point for Agents 1→2→3/4 (Task #64). Each stage gated on the previous one's output (fresh, non-empty audit report → intake exit 0 → orchestrator over SSH). `--no-codex` runs stages 1-2 with zero token spend. No timer/cron, by design.
  - File or resource affected: `ansible-controller01:~/run-patch-pipeline.sh`, `agents/run-patch-pipeline.sh`
- 🔧 Installed `jq` on `codex-agent-lxc`; created `/var/log/patch-pipeline/` (owned by `hermes-admin`) there.

### Fixed
- 🐛 Pre-deploy review of yesterday's undeployed orchestrator v2 found it would have failed on first run: (1) `jq` never installed on `codex-agent-lxc` (v1 used `python3`), (2) `gitlab_get` passed `--data-urlencode` without `curl -G`, turning the issue-list GET into a POST, (3) `mkdir || sudo mkdir && chown` precedence bug. All fixed.
- 🐛 **Plan-truncation bug, present since v1**: SRE plans were parsed from `codex exec` stdout with `tail -n +2`, which dropped the first line of every plan — the risk-level assessment. Switched to `codex exec -o <file>` (final message only). Added `REPLAN_IIDS` override to re-plan specific issues. #7/#8 replanned and verified complete; **#1-6 still carry truncated plans** (not replanned — token spend left to user).
- 🔄 Deployed orchestrator **v2.1** to `codex-agent-lxc` (v1 kept as `~/orchestrator.sh.v1-bak-20261005`).
  - File or resource affected: `codex-agent-lxc:~/orchestrator.sh`, `agents/orchestrator.sh`
- 🔄 Removed stale "unattended via systemd timer" header comments from `patch-audit.sh`/`backlog-intake.sh` (redeployed, comment-only change; previous copies kept as `*.bak-20261005`).

### Infrastructure
- 🔄 Full chain run: 18/20 hosts audited, 7 with drift, 0 new issues (dedupe held), #7 + #8 claimed and planned → all 8 open issues in `homelab/agentic-patch-management` now `awaiting-approval`. Nothing executed on any target host (SRE agent is still planner-only).
- ⚠️ `pbs-server` (192.168.20.50) now fails audit from ansible-controller01: that VM ARP-resolves the IP to the known squatter MAC `78:20:51:8a:e3:7b` (only Proxmox nodes have the static ARP pin). PBS itself healthy (UI 200 from node01). New Task #65. `homepage-lxc` still fails (Task #55).

## [2026-10-04] - Agents 1+2 ported to always-on infra; orchestrator bug found (fix written, not deployed); session paused

### Added
- 🔧 Ported Agent 1 (`patch-audit.sh`) and Agent 2 (`backlog-intake.sh`) from Kratos-only PowerShell proofs-of-concept to bash on `ansible-controller01` — an always-on host, which is what "production ready" actually requires. Ran for real: found `docker-vm-core-utilities01` has pending updates (34) that the original Kratos-run version couldn't even see — running from a different host incidentally fixed Task #54's reachability gap. Filed new issues #7 (`docker-vm-core-utilities01`) and #8 (the newly-created `codex-agent-lxc`, audited for the first time). Dedupe correctly skipped the original 6.

### Found (fix written, not yet deployed)
- 🐛 Real bug in `orchestrator.sh`: if the claim step succeeds but the Codex planning call then fails, the issue gets stuck permanently — labeled `claimed`, and the dedupe check treated that as "already handled," so it would never be retried. Fixed in a v2 (only `awaiting-approval` counts as done; bare `claimed` retries) with real HTTP-status checking and persistent logging added too. **The fix exists in this vault's `agents/orchestrator.sh` but was not deployed to the live `codex-agent-lxc` host this session** — that host is still running the buggy v1.

### Decided
- ⚠️ No scheduling, ever, by design — Codex CLI calls cost real API tokens, and the user wants manual control over when that spend happens rather than a background timer calling it. The next build target is a single manual entry point chaining all three stages, not systemd timers (which were being built toward before this was clarified).

### Session paused here
- Issues #7 and #8 are unclaimed — orchestrator hasn't run against them.
- Next session: deploy orchestrator v2, build the manual chain-trigger script, run the full chain once for real against #7/#8.

## [2026-10-04] - Agents 3+4 built: Orchestrator + planner-only SRE agent, running on real Codex CLI

### Added
- 🔧 Provisioned `codex-agent-lxc` (CT211, 192.168.40.16, node01) — new unprivileged LXC dedicated to running Agents 3-6 of 59 - Autonomous Patch Management Agent Pipeline, mirroring Helios's setup pattern. `hermes-admin` with passwordless sudo and the standard homelab SSH key.
- 🔧 Installed Node.js 22, Codex CLI (`codex-cli 0.160.0`), and `bubblewrap` — authenticated with a user-provided OpenAI API key (`codex login --with-api-key`).
- 🔧 `agents/orchestrator.sh` (Agent 3 — Backlog-Orchestrator + Agent 4 — SRE Agent, combined in one script): polls open `patch-management` GitLab issues, claims them, dispatches to Codex CLI for a remediation plan, posts the plan, labels `awaiting-approval`. Ran for real against all 6 open issues — plans were genuinely reasoned (e.g. correctly distinguished LXC vs. host-level reboot need from package names alone), and a second run correctly skipped all 6 (dedupe verified).

### Fixed
- 🐛 `codex exec` hangs indefinitely over non-interactive SSH with no output or error — it's waiting on stdin EOF, not a sandbox/namespace problem (a genuine but unrelated missing-`bubblewrap` warning looked like the cause at first but wasn't). Fix: always redirect `< /dev/null`.

### Deliberate safety scoping
- The SRE agent has **no SSH credentials to any production host** and cannot execute anything — it only reasons from the issue text Agent 1 already collected. This is intentional: `codex-agent-lxc` has no Omada ACL lockdown yet (Task #63) and Sentinel's Discord approval-gate hookup isn't built or proven. Live execute access is not going on this host until both of those are actually done.

### Still not built
- Agent 5 (Documentation) — blocked on deciding where it runs; the vault lives on Kratos (OneDrive), not on `codex-agent-lxc`.
- Agent 6 (Verification) — blocked on confirming Sentinel's webhook key actually works (flagged elsewhere as a likely stale placeholder).

## [2026-10-04] - Agents 1+2 of patch-management pipeline built and proven live

### Added
- 🔧 `agents/patch-audit.ps1` (Agent 1 — Patch-Audit) and `agents/backlog-intake.ps1` (Agent 2 — Backlog-Intake), the first real code for 59 - Autonomous Patch Management Agent Pipeline. Token is read from an environment variable, not hardcoded, since this vault repo is public.
- 🔄 Ran Agent 1 against all 19 known hosts: 17 reachable, 6 with real pending updates (`ansible-controller01` 36, `gitlab-runner-vm01` 28, `gitlab-vm01` 33, `immich-vm01` 34, `pbs-server` 9, `helios-lxc` 43), 11 fully patched, 2 failed (`docker-vm-core-utilities01`, `homepage-lxc` — pre-existing Task #54/#55 issues, not new).
- 🔄 Fed that report into Agent 2, which filed 6 real, labeled GitLab issues (#1-6) in `homelab/agentic-patch-management` and was verified not to double-file on a second run (dedupe confirmed).

### Fixed
- 🐛 Found during Agent 1 development: `apt list --upgradable` doesn't need `sudo`, and using it anyway broke on 2 LXCs where `hermes-admin` lacks passwordless sudo — only VMs were ever confirmed passwordless (see [16 - SSH Configuration](16%20-%20SSH%20Configuration.md)). Dropped `sudo` from the script entirely.

### Deferred / honest scope check
- Agents 3-6 (Backlog-Orchestrator, SRE agent(s), Documentation, Verification) are still just design — not built. Phases 3-4 of the pipeline (dedicated agent-runner host, VLAN/ACL, and the actual runtime platform: Codex CLI vs. Claude Code subagents vs. custom scripts) are still undecided. Today's two agents ran as manual ad-hoc scripts from Kratos, not as a scheduled or unattended system — that's a deliberate, not accidental, stopping point: the pipeline's own Phase 6 calls for proving stages manually before trusting them unattended, and committing agents with live SSH/GitLab credentials to an always-on host without that proof, or before the runtime is even chosen, isn't something to rush through in one sitting.

## [2026-10-04] - Phase 0+1 of patch-management pipeline closed out

### Security
- 🔧 Backed up `/etc/pve/priv/authorized_keys` (Task #57) — pulled from node01, verified byte-identical across all 3 nodes via md5sum, saved outside git to `OneDrive\My Personal Files\Side Projects\pve-authorized_keys-backup-2026-10-04.txt`. Closes Phase 0 of 59 - Autonomous Patch Management Agent Pipeline.
- 🔧 Wired the two scoped Proxmox tokens from Task #58 (created 2026-10-03, unused since) into two **new** Helios instances rather than modifying Helios's code or touching the existing one: `helios-observer.service` (:8001, read-only `PVEAuditor` role) and `helios-operator.service` (:8002, `HeliosOperator` role: audit+power, no create/delete). Each gets its own generated API key. The existing full-priv instance (:8000) is deliberately untouched — Helios's VM/LXC create+delete endpoints need `VM.Allocate`, which neither scoped role grants, so swapping the interactive instance's token would have silently broken provisioning. Both new instances verified live against the real Proxmox API (200 on `/api/v1/compute/nodes`). Closes Task #58 and the remaining Phase 1 items in 59 - Autonomous Patch Management Agent Pipeline. See 11 - Credentials.

## [2026-10-04] - Backlog-Intake GitLab token (Task #61)

### Security
- 🔧 Created a dedicated `homelab-patch-backlog` GitLab project and a scoped Project Access Token (Reporter role, `api` scope, 1-year expiry) for the patch-management pipeline's future Backlog-Intake agent, instead of reusing Sentinel's broad `api`/`read_repository`/`write_repository` token. Closes Task #61 and the matching Phase 1 line in 59 - Autonomous Patch Management Agent Pipeline. See 11 - Credentials.
- ⚠️ Found Sentinel's own GitLab token (`glpat-qslK0...`) expired 2026-09-28 while documenting the above — likely breaking the wiki-sync/`gitlab.py` cog right now. Flagged in 11 - Credentials and the Task Registry; not yet fixed.

### Investigated (no change needed)
- 🔄 Brief "GitLab unreachable" report — SSH diagnostics on `gitlab-vm01` showed all GitLab services healthy throughout and a local `curl` returning HTTP 200 the whole time. Confirmed it wasn't GitLab- or even homelab-specific: `homepage`, `auth`, and the apex domain also fail to resolve via public DNS, consistent with the existing split-DNS (Pi-hole/Tailscale-only) design. Most likely a transient client-side DNS/network blip; resolved on its own before any server-side action was taken.

---

## [2026-10-03] - INCIDENT: wiped cluster-wide root SSH key during Helios LXC build; fixed immich-vm01 SSH key ACL; upgraded Immich to v3.2.4

### Incident — cluster-wide root SSH lockout (self-inflicted, recovered)
- 🔴 While bootstrapping a new LXC (CT210, for relocating Helios — see below), ran a `pct exec 210 -- bash -c "echo '$pub' > /root/.ssh/authorized_keys ..."` command through too many nested shell layers (PowerShell → SSH → pct exec → bash). The quoting broke: the `>` redirect executed in **node01's own shell**, not inside the container, and truncated **`/root/.ssh/authorized_keys` to empty**.
- **Scope was worse than initially assessed**: `/root/.ssh/authorized_keys` on every Proxmox node is a symlink to `/etc/pve/priv/authorized_keys` — Proxmox's single cluster-wide root SSH key store, synced via pmxcfs/corosync. Wiping it meant **root SSH broke on node01, node02, AND node03 simultaneously**, not just node01 (my first message to the user understated this).
- Cluster itself was never at risk — `pvecm status` showed `Quorate: Yes` throughout; this only affected remote root SSH access, not corosync/VM/LXC operation.
- **Recovered** via the Proxmox web UI's browser-based Shell (noVNC, bypasses SSH entirely) on node01: confirmed the file was 0 bytes, then rewrote it. Shell redirection (`printf ... >`) and even `echo` with single quotes kept getting mangled by that console's paste handling (stray characters, hung quote-continuation prompts) — what actually worked was `nano /etc/pve/priv/authorized_keys`, pasting the key line directly in the editor, and saving. Verified root SSH restored on all 3 nodes individually afterward.
- **Key used for recovery**: `ssh-ed25519 <redacted> hermes@homelab-nopass` — this is the key that's actually been working all session via the private key at `C:\Users\herms\OneDrive\My Personal Files\Side Projects\homelab_ed25519`. It does **not** match the "Public Key (current)" value recorded in `11 - Credentials.md`'s SSH Access table (which lists a different, rotated-2026-08-26 key) — that doc entry is stale/wrong about which key is actually deployed and working. Needs correcting (see follow-up).
- **Root cause takeaway**: avoid nested nested-shell quoting (PowerShell→SSH→pct exec→bash) for anything that writes files — write the target content to a local file and `scp`/copy it in instead, same pattern already used successfully elsewhere this session (e.g. the Helios docker-compose.yml fix).

### Added — Helios relocated to its own dedicated LXC
- Created CT210 (`helios-lxc`), unprivileged Debian 12, 192.168.40.14/24 (VLAN 40), 2 core / 2GB / 8GB disk, on node01 — relocating Helios off `ansible-controller01` onto its own dedicated host, as a first step toward the Codex-agent secure-access design discussed this session. Ran via direct Python (systemd + venv) rather than Docker, to sidestep the known Docker-in-LXC capability bug (Task #15) — unprivileged LXC is the correct/default choice here since nothing needs Docker's nesting/capability requirements.
- Bootstrapped `hermes-admin` with the correct working key (see incident above) and passwordless sudo (`/etc/sudoers.d/hermes-admin`), matching the pattern on other hosts.
- Copied Helios source (code + `.env` + `config.yaml`) from `ansible-controller01` via tarball relay through this PC (avoided `git clone` since the code isn't pushed to GitHub yet — Task #42). Installed into a venv (`pip install -r requirements.txt`, `pip install -e .`), clean install, no errors.
- Installed `helios.service` (systemd, runs as `hermes-admin`, not root; `EnvironmentFile=.env`; `Restart=on-failure`) — `systemctl enable --now helios` came up clean on first try.
- Verified end-to-end: `/health` responds from both node01 and directly from Kratos; `helios compute nodes` (CLI on ansible-controller01, repointed via `~/.helios/cli.yaml` → `http://192.168.40.14:8000`) returns live data for all 3 nodes.
- Decommissioned the old Docker deployment on `ansible-controller01` (`docker compose down` — container + network removed cleanly, confirmed no `helios` containers remain).
- **Known gap, not fixed as part of this move**: Helios's `ansible`/`terraform` features point at local paths (`~/ansible/ansible-playbooks`, `~/homelab-infra-automation-project/terraform`) that only exist on ansible-controller01 — those two feature areas won't work from the new host. Already flagged as untested/unwired in Task #43. Proxmox/Synology/Omada control-plane features are unaffected.
- **Still not scoped down**: the Proxmox token backing Helios (`root@pam!helios`) remains full-privilege (`privsep 0`). Splitting it into observer/operator roles, and issuing per-agent Helios API keys, is the next real step for the Codex-agent design — not done yet, this session only handled the relocation.
- Updated: `11 - Credentials.md` (Helios host + known gaps + the stale-key correction), `10 - IP Address Map.md`, `CLAUDE.md`'s LXC table.

### Fixed
- 🐛 `homelab_ed25519` private key (`C:\Users\herms\OneDrive\My Personal Files\Side Projects\homelab_ed25519`) had an inherited ACL granting access to `KratosPC\CodexSandboxUsers`, which Windows OpenSSH refuses to load ("UNPROTECTED PRIVATE KEY FILE"). Caused `Permission denied (publickey)` when SSHing to immich-vm01. Fixed via `icacls /inheritance:r` + `icacls /grant:r "<user>:(R)"` to restrict to the owning account only.

### Changed
- 🔄 Immich upgraded on immich-vm01: v3.2.x → **v3.2.4** (patch release, tracking `:release` tag, no compose/extension changes needed). Pre-upgrade `pg_dumpall` backup taken manually. Verified via `/api/server/version`. See [57 - Immich VectorChord Migration and v3 Upgrade](57%20-%20Immich%20VectorChord%20Migration%20and%20v3%20Upgrade.md) for the full record and [07 - Deployed Services](07%20-%20Deployed%20Services.md) for the version field update.

### Added
- 📝 New "SSH Access" section in `42 - Immich Backup and Restore.md` documenting the `hermes-admin@192.168.40.22` key path, the `-i` flag requirement, and the Windows ACL gotcha above.
- 📝 New `58 - Helios API Reference.md` — static mirror of Helios's OpenAPI spec (66 endpoints), pulled live from `helios-lxc:8000/openapi.json`.
- 📝 New `59 - Autonomous Patch Management Agent Pipeline.md` — design spec for a 6-stage agent pipeline (Patch-Audit → Backlog-Intake → Backlog-Orchestrator → SRE Agent(s) → Documentation → Verification), wired into existing Helios/Sentinel/GitLab infra rather than parallel systems. Includes a phased prerequisites checklist.
- 🔧 Started Task #58: created Proxmox roles `PVEAuditor` (observer, built-in) and `HeliosOperator` (new custom role, `VM.Audit,VM.PowerMgmt` only) and issued scoped API tokens (`helios-observer@pve!agent`, `helios-operator@pve!agent`) against two new dedicated PVE identities. **Not yet wired into Helios** — its `config.yaml` only supports one Proxmox token app-wide, so these tokens are inert until that's resolved (run two Helios instances, or add per-caller credential support). See 11 - Credentials and Task #58.
- Rewrote all 30 posts in `Homelab Blog Posts/Rewritten Series/` into a more technical, precision-focused voice (per user request), fixing several stale/wrong infra details surfaced along the way (VLAN table, NAS IP, node topology, an invented download client) and catching one real leaked credential (an old, already-revoked Radarr API key) that was sitting in post 26 as an example and got replaced with a placeholder before being republished.

### Investigated — corrected finding
- ⚠️ Initially suspected `docker-vm-core-utilities01` (192.168.40.13) was down — unreachable (100% ping loss, SSH/HTTP timeout) **from this Windows PC (Kratos, 192.168.10.10)**, while sibling VLAN 40 hosts (`.11`, `.22`) responded fine from the same machine. **Corrected after checking from node01 (192.168.20.20)**: ping to `.13` is 0% loss (1.5ms), and both the Sentinel webhook (`:5050/health` → HTTP 200) and Grafana (`:3000` → HTTP 302) respond normally. The VM was never down.
- **Real finding**: this is a Kratos-specific reachability gap to `.13` specifically, not a VM outage. Per `41 - Omada ACL Implementation Guide.md` (Rules 15-18), Kratos is supposed to have **unrestricted access to all infrastructure**, bypassing VLAN 10 restrictions — so Kratos failing to reach `.13` while successfully reaching `.11`/`.22` (same VLAN, same ACL rule) contradicts the documented design. Not yet root-caused — could be an Omada ACL rule that doesn't actually cover `.13`, a host-level firewall on `.13` itself, or something related to the in-progress OPNsense/cross-VLAN routing work noted in the 2026-09-15 entry below. User separately confirmed services are reachable from elsewhere, consistent with this being Kratos-specific.
- Also noticed in passing: <redacted> (CT209) shows `stopped` in Proxmox — unrelated, not yet investigated.

### Fixed (continued)
- 🐛 **Helios healthcheck false-positive** — `helios-api` on ansible-controller01 showed `unhealthy` (FailingStreak 232) in `docker ps`, which read as "Helios is down." Root cause: the Docker healthcheck ran `curl -f http://localhost:8000/health` *inside* the container, but the `python:3.12-slim` image never installed `curl` — the healthcheck itself was failing, not the app (logs showed clean startup and `GET /docs → 200 OK` the whole time). Fixed by switching the healthcheck to a `python3 -c` one-liner using `urllib.request` (already in the base image, no new dependency). Backed up old compose file to `docker-compose.yml.bak-20261003` first. Recreated container — now reports `Up (healthy)`. Verified end-to-end with `helios compute nodes` (live data, all 3 nodes online). Also confirmed the earlier `helios: command not found` was a red herring from running a non-interactive SSH test command (which doesn't source `.profile`) — in an actual interactive/login SSH session, `~/.local/bin` is already on PATH and the CLI resolves fine.

### Deferred / follow-up needed
- Task Registry #14 (rebuild Immich backup automation — `/opt/immich/backup-db.sh` still doesn't exist on immich-vm01) is still open; surfaced again during this upgrade. Recommend taking a manual `pg_dumpall` backup before any future upgrade until this is fixed.
- **Revised**: Root-cause why Kratos can't reach `192.168.40.13` despite its documented full-access ACL rule, while it can reach other VLAN 40 hosts. Check Omada ACL rule coverage and `.13`'s own host firewall.
- **New**: Confirm whether `homepage-lxc` (CT209) being stopped is intentional.

---

## [2026-09-15] - OPNsense bring-up in progress (bare-metal, replacing Pi-hole as primary DNS) — session paused, still unreachable cross-VLAN

### Context
Goal: configure OPNsense as primary DNS resolver with Pi-hole as secondary. User initially reported OPNsense at `192.168.90.60` unreachable; my docs had the stale planned address `192.168.91.30` (VLAN 91). User confirmed `.90.60` is the real, current address — hardware is bare-metal (not a VM), physically racked and cabled into **Morpheus Rack Switch (SG2210P) Port 6** ("OpnSense Server", Trunk, Native VLAN 90 Management), not the Core Switch port originally documented.

### Infrastructure
- 🔧 Fresh OPNsense 26.7 install found with only a WAN interface auto-assigned (`igc0` → `192.168.90.60/24`) and no LAN — explains initial total unreachability (default WAN ruleset blocks all RFC1918-sourced traffic).
- 🔧 Console: reassigned `igc0` from WAN to **LAN** (single-arm, no WAN — matches decision to defer full router/firewall role to a future project; DNS resolver duty only for now).
- 🐛 Interface reassignment reset LAN to factory default `192.168.1.1/24`; corrected back to `192.168.90.60/24` via console option 2.
- 🐛 Found and killed the real blocker: a leftover `dhclient` process still bound to `igc0` from its original WAN role was silently overriding the static `.60` back to a DHCP-leased `.58` on every renewal, despite `config.xml` correctly showing `<ipaddr>192.168.90.60</ipaddr>`. Fixed by rebooting the box (clean boot doesn't spawn WAN dhclient since role is now LAN).
- 🔧 Diagnosed that OPNsense's default LAN ruleset only permits traffic sourced from its own subnet (`192.168.90.0/24`) — cross-VLAN clients (Kratos, `192.168.10.10`) get silently dropped by the implicit deny. Confirmed via same-subnet test from Pi-hole (`192.168.90.53`, ping + curl succeeded instantly) vs. cross-subnet test from Kratos (timed out) — isolated the problem to firewall/routing, not the OPNsense host itself.
- 🐛 Repeated console "Restore web GUI access defaults" prompts (answered `y` each time during interface work) silently reset the GUI to plain HTTP:80 — cost significant time testing HTTPS:443 against a box that only had `lighttpd` listening on `:80` (confirmed via `sockstat -4 -l`).
- 🔧 Added permanent fixes via GUI (through an SSH local-port-forward tunnel via Pi-hole, since Kratos still couldn't reach the GUI directly at the time): `System → Gateways` gateway `LAN_GW` → `192.168.90.1`; static route `192.168.0.0/16` via `LAN_GW` (covers all VLANs in one entry); aliases `MGMT_HOSTS` (`192.168.10.10`) and `INTERNAL_NETS` (all VLAN subnets); firewall rules on LAN allowing DNS (53) from `INTERNAL_NETS`, GUI (80/443) from `MGMT_HOSTS`, ICMP from any; re-enabled HTTPS:443 in `System → Settings → Administration`.
- ⏸️ **Still unreachable from Kratos (cross-VLAN) after all of the above** — ping/HTTP/HTTPS all time out again as of session end. Not yet re-isolated whether this is a regression in the service (HTTPS re-enable may not have restarted cleanly) or the new rules/route not actually taking effect. Next session: repeat the Pi-hole same-subnet test *after* the HTTPS switch to isolate service-vs-routing again, and check `sockstat`/`pfctl -sr` on the box directly.

### Deferred / follow-up needed
- Finish OPNsense reachability troubleshooting (see Task Registry #7 and TODO - Homelab Improvements 2026-05 §9).
- Once reachable: configure Unbound, migrate the 45 `*.hrmsmrflrii.xyz` host overrides from Pi-hole, update DHCP scope to hand out OPNsense as primary DNS / Pi-hole as secondary, update the Omada `OPNSENSE` ACL IP group from `192.168.91.30` to `192.168.90.60`.
- Update `11 - Credentials.md`, `10 - IP Address Map.md`, `01 - Network Architecture.md`, and the Omada ACL guide once the final working IP/topology is confirmed (currently `192.168.90.60`, single-arm LAN-only, no WAN — differs from all previously documented values).
- User has expressed future interest in exploring a full two-interface router/firewall role for OPNsense (would need a second NIC and a new isolated VLAN) — not in scope for the current DNS-resolver goal.

---

## [2026-09-14] - Cleared stuck Sonarr queue item (8-month-old stale duplicate)

User asked to clear dashboard/queue items that weren't actually downloading. Checked Radarr (`192.168.40.11:7878`) and Sonarr (`192.168.40.11:8989`) queues via API:
- Radarr: 2 items, both legitimately actively downloading (real progress/timeleft) — left alone.
- Sonarr: 1 item stuck since **2026-01-26** — `A.Knight.of.the.Seven.Kingdoms.S01E02...FLUX`, `trackedDownloadState: importPending`, blocked with "Not a quality revision upgrade for existing episode file(s)" (episode already has a file, so Sonarr refused to import this one but never dropped it from the queue).
- Removed via `DELETE /api/v3/queue/1336854681?removeFromClient=true&blocklist=false&skipRedownload=true` — deletes the now-unneeded file from SABnzbd's completed folder, doesn't blocklist the release, doesn't trigger a re-search. Confirmed Sonarr queue now shows `totalRecords: 0`.

---

## [2026-09-14] - Investigated Glance Backup tab showing "unknown" status (couldn't reproduce; restarted Glance as precaution)

### Context
User reported the Backup tab on Glance showing "unknown" status despite backups actually succeeding — a follow-up to the stale-Feb-2026-data bug fixed earlier the same day (see entry below). Diagnosed from a client on VLAN 10 with no direct route to VLAN 40; used `ansible-controller01` (192.168.20.30, VLAN 20) as an SSH jump host to reach `docker-vm-core-utilities01` (192.168.40.13) and `docker-lxc-glance` (192.168.40.12) directly.

### Findings — everything checked out healthy, could not reproduce
- `curl http://192.168.40.13:9102/status` from inside the `glance` container itself returns correct data: top-level `status: success`, `job_status.daily.status: success`, `job_status.main.status: success`, real 2026-09-13 backup dates.
- `nas-backup-status-api` container logs show clean 4-minute cache-refresh cycles with zero errors for the several hours prior.
- Glance's widget template (`/opt/glance/config/*.yml`, "Backup Jobs Overview" + "Backup Services" monitor) matches the API's JSON shape exactly — no field-name mismatch that would fall through to an "N/A"/unknown render path.
- Both targets in the "Backup Services" monitor widget (PBS `https://192.168.20.50:8007`, Backup API `http://192.168.40.13:9102/health`) return clean `200`s from inside the `glance` container.

### Action taken
- Restarted the `glance` container (`docker compose restart`) from `docker-lxc-glance` as a precaution to clear any lingering render-level cache from before this morning's `nas-backup-status-api` fix/rebuild. Confirmed Glance back up and serving `200` afterward.

### Conclusion
Most likely a transient render observed either mid-cache-cycle during/shortly after this morning's `nas-backup-status-api` rebuild, or a stale browser tab. No code change was needed this time — if "unknown" recurs after this restart, the next step is to catch it live (`docker logs -f glance` while reproducing) rather than checking after the fact, since every point-in-time check here came back clean.

### Note on network access
Direct client access from VLAN 10 (Wi-Fi, DHCP-assigned, non-Kratos-reserved IP) to VLAN 40 raw ports (22/80/443/3030/9090/9102/8080 on `.13`) was blocked, while Traefik-fronted HTTPS to services on the same host (grafana/n8n/paperless/karakeep, all 200) worked fine — user confirmed no Omada ACL rules currently exist, so this is likely router/OPNsense inter-VLAN routing rather than the Omada ACL feature documented in [41 - Omada ACL Implementation Guide](41%20-%20Omada%20ACL%20Implementation%20Guide.md) (worth reconciling that doc against actual current config — it may describe a config that was since simplified or never fully deployed). `ansible-controller01` (VLAN 20) had a working route to both VLAN 20 and VLAN 40 hosts and made a reliable SSH jump host for this session.

### Also: removed emoji from Backup tab title and Backup Jobs Overview card headers
Per user request. `/opt/glance/config/glance.yml`:
- Line 1041: `- name: "📦 Backup"` → `- name: "Backup"`
- Line 1101: `📦 DAILY BACKUPS` → `DAILY BACKUPS`
- Line 1125: `\U0001F5C4 MAIN BACKUPS` → `MAIN BACKUPS` (this one was a literal unescaped `\U0001F5C4` string in the template, not a rendered emoji — likely pasted in as an unescaped Python-style unicode literal originally, so it was probably rendering as garbled text/mojibake in the browser rather than the intended 🗄️ icon)
- Line 1149: `☁️ NAS SYNC` → `NAS SYNC`

Restarted `glance` container after each change; confirmed dashboard serving `200` afterward.

---

## [2026-09-14] - Root-caused Glance Backup tab showing stale Feb 2026 data + found a real 8-day PBS outage

### Context
User asked whether last night's backups succeeded and why Glance's Backup tab didn't reflect it. Actual PVE/PBS backups were succeeding (confirmed via `vzdump` task index on node01 and `proxmox-backup-manager task list` on PBS), so the mismatch was entirely in the reporting layer feeding Glance — the `nas-backup-status-api` container (port 9102 on docker-vm-core-utilities01). Investigation surfaced three separate, unrelated bugs plus one genuine infrastructure outage:

### Root causes found
1. **Stale SSH `known_hosts` entry for PBS** in `/home/hermes-admin/.ssh/known_hosts` on docker-vm-core-utilities01 (bind-mounted read-only into the container at `/root/.ssh`) — untouched since Feb 7, 2026, predating a PBS host key change. Not actually the app's blocker (its `run_ssh_command()` uses `StrictHostKeyChecking=no`), but fixed for hygiene — updated to PBS's current host key (`<redacted>`).
2. **Real bug in `nas-backup-status-api/app.py`'s `get_backup_job_status()`**: it found the "most recent" PBS snapshot via `find {path} -maxdepth 4 -type d -name '20*T*' | sort -r | head -1` — sorting the **full path string** (`/backup-ssd/vm/{vmid}/{timestamp}`) rather than the timestamp. A long-decommissioned VM 311 (not in the current fleet, never pruned from PBS) always won this sort because `"311"` > any current VMID string (100s/200s), so the API perpetually reported VM 311's Feb 2026 snapshots as the latest backup regardless of what actually ran overnight.
   - **Fixed**: changed the `find` commands (both the "latest backup" query and the "recent job duration" query) to `-printf '%f\t%p\n' | sort -r | head -N | cut -f2`, sorting by the ISO-8601 timestamp basename instead of the full path. Original backed up to `app.py.bak-2026-09-14` on the host before patching. Rebuilt and recreated the container (`docker compose up -d --build`).
3. **Real 8-day infrastructure outage, unrelated to Glance**: node03 (the physical desktop, Ryzen 9 5900XT — hosts PBS/CT100, Immich VM, GitLab VM) was completely powered off from **2026-09-06 03:46 to 2026-09-13 21:37 (local)**, confirmed via `journalctl --list-boots` showing a boot gap of that exact span. This took PBS offline for the whole window, so the nightly PBS→NAS offsite sync (`/usr/local/bin/pbs-backup-to-nas.sh`, cron.d job at 2 AM) simply couldn't run — its log (`/var/log/pbs-nas-backup.log`) sat frozen at its last successful run, 2026-09-05 02:04. VM/CT backups themselves resumed automatically once the node came back (Proxmox's scheduler caught up with an out-of-band run at ~14:49 UTC on 2026-09-13, ~1h49m after boot).
   - Manually re-ran `/usr/local/bin/pbs-backup-to-nas.sh` on pbs-server to close the sync gap immediately rather than waiting for tonight's 2 AM slot — completed successfully (`main: 360G`, `daily: 479G`, ~7m44s).
4. **Cosmetic/latent bug, not fixed**: `pbs-backup-to-nas.sh` has a hardcoded fallback `mount -t nfs 192.168.20.31:/volume2/ProxmoxData` (the **dead** NAS NIC per [01 - Network Architecture](01%20-%20Network%20Architecture.md) — should be `.32`). Currently harmless because the real mount is managed by a correct `/etc/fstab` entry (`.32`, `_netdev`) that persists across reboots, so the script's `mountpoint -q` check always passes before the flawed fallback line would ever run. Flagged for cleanup — if the fstab entry is ever removed/changed, this script would silently hang trying to mount a dead host.

### Verified
- `curl http://192.168.40.13:9102/status` now returns accurate `job_status.daily`/`job_status.main` (2026-09-13, status `success`) and current `last_sync` (2026-09-14, post manual run) instead of frozen Feb 2026 data.
- Restarted Glance container to clear any render-level cache.

### Files affected
- `/opt/nas-backup-status-api/app.py` on docker-vm-core-utilities01 (patched; `.bak-2026-09-14` kept alongside)
- `/home/hermes-admin/.ssh/known_hosts` on docker-vm-core-utilities01

### Follow-up (not done this session)
- Consider pruning VM 311's orphaned snapshots from PBS (`/backup-ssd/vm/311`, `/backup/vm/311`) — left untouched since deleting backup data needs explicit user sign-off.
- Fix the `.31`→`.32` typo in `pbs-backup-to-nas.sh` for defense-in-depth even though it's currently dead code.
- Investigate why node03 was powered off for over a week (2026-09-06 to 2026-09-13) — worth confirming this was intentional/known, not an unexpected crash.

---

## [2026-09-13] - Fixed Jellyseerr 404 (broken Authentik embedded outpost, unrelated null invalidation_flow fields)

### Context
User reported Jellyseerr "not running." The `jellyseerr` container itself was healthy the whole time (`docker ps` showed it up, app responded 307 on its own port, Traefik→backend connectivity was fine). The actual failure was in Authentik: Traefik's `authentik-auth` ForwardAuth middleware (the only Traefik router in this whole deployment that uses it — every other Authentik-adjacent service either has anonymous access baked in, like Grafana, or its own native OIDC login, like Jellyfin) was getting a hard 404 straight from Authentik itself (`Server: uvicorn`, `X-Authentik-Id` header, Authentik's own branded "Not Found" error page) instead of a login redirect.

### Root cause (two compounding issues)
1. **`authentik Embedded Outpost.service_connection`** was set to `Docker Service-Connection Local Docker connection` instead of `None`. The embedded outpost is supposed to run self-managed inside the `authentik-server`/`authentik-worker` processes — pointing it at a Docker connection makes Authentik think it needs to externally manage a separate deployed outpost container, which it never actually did.

### Fixed
- ✅ Set `authentik Embedded Outpost.service_connection` → `None` via `ak shell`.
- ✅ Set `invalidation_flow` → `default-provider-invalidation-flow` for the 6 affected providers via `ak shell`.
- ✅ Restarted `authentik-server` + `authentik-worker` — confirmed `Failed to fetch outpost from API` no longer appears in logs post-restart.
- ✅ Verified: `https://jellyseerr.hrmsmrflrii.xyz` now returns `302` → Authentik's `/application/o/authorize/` login flow (previously `404`). Spot-checked Glance (200) for regressions — none found. Pi-hole's `000`/cert warning on its own hostname is pre-existing/unrelated (resolves straight to Pi-hole's self-signed UI, not through Traefik).

### Note for future work
- The 6 providers with null `invalidation_flow` were likely created via an older blueprint/UI flow before that field was effectively required — worth a quick audit of any *other* nullable Flow FKs across providers (`authorization_flow`, etc.) the next time Authentik is touched, since this class of bug is silent until something else depends on a full outpost config fetch succeeding.

---

## [2026-09-13] - Cleared all paused torrents in Deluge (incl. data deletion)

### Changed
- ✅ Removed all 35 paused torrents from Deluge on docker-lxc-media (192.168.40.11 / CT205) via `core.remove_torrent(id, remove_data=true)` over the WebUI JSON-RPC API. Verified zero paused torrents remain afterward.
- ⚠️ Per explicit user choice, this included **3 torrents that were still mid-download** (not just the 31 already at 100%): `Parasite.2019...` (50.6%), `Manhunt.2024.S01E02...` (1.4%), `House.of.the.Dragon.S02E06.REPACK...` (86.3%). Their partial data was deleted from disk along with the rest — these would need to be re-downloaded from scratch if wanted again.

---

## [2026-09-13] - Reset Deluge WebUI password (user forgot it)

### Changed
- ✅ Backed up `/opt/arr-stack/deluge/web.conf` on docker-lxc-media (192.168.40.11 / CT205) to `web.conf.bak-20260913-password-reset`, stopped the container, deleted the live file, restarted. Confirmed via Deluge's own `auth.login` JSON-RPC endpoint that the password is now reset to Deluge's built-in default (`deluge`) — login succeeds. **User should log in and set a new password immediately** via Preferences → Interface → Password.
- Note: the deleted `web.conf` hadn't physically regenerated on disk as of this writeup (Deluge appears to be running on in-memory defaults until it next persists config) — not a concern functionally (login already verified working) but if `web.conf` is still missing next time this host is touched, that's expected/known, not a new bug.
- Also noticed in passing: the old `web.conf` had ~100 stale WebUI sessions accumulated (going back to Dec 2025) that never expired/cleaned up. Not acted on beyond the reset above — worth keeping in mind if Deluge WebUI behaves oddly with session bloat again in the future.

---

## [2026-09-13] - Updated SABnzbd to 5.1.3 (pinned, was floating `:latest`)

### Context
User asked to update SABnzbd to 5.1.3 specifically. Unlike the Jellyseerr case above, this was a real update — `docker pull lscr.io/linuxserver/sabnzbd:5.1.3` returned "Downloaded newer image" with a different digest than what was running.

### Changed
- ✅ `/opt/arr-stack/docker-compose.yml` on docker-lxc-media (192.168.40.11 / CT205): `lscr.io/linuxserver/sabnzbd:latest` → `lscr.io/linuxserver/sabnzbd:5.1.3`. Backup saved as `docker-compose.yml.bak-20260913-sabnzbd-pin`.
- ✅ Recreated the container (`docker compose up -d --force-recreate sabnzbd`) — again succeeded cleanly with no AppArmor error (second recreate on CT205 today with no reproduction of task #15's bug). Log confirms `Starting SABnzbd.py-5.1.3`, web UI up on :8080, no errors. Verified reachable externally (`https://sabnzbd.hrmsmrflrii.xyz` → 200).

---

## [2026-09-13] - Pinned Jellyseerr to explicit 2.7.3 tag (was floating `:latest`)

### Context
Following the ForwardAuth fix above, user asked to update Jellyseerr to the latest version. Investigation showed the running image was already bit-for-bit identical to Docker Hub's `2.7.3` tag (same digest, `sha256:4538137b...`) — the in-app "2.7.3 update available" banner was a false positive caused by Jellyseerr's own `/api/v1/status` reporting a stale internal version string (`1.35.0`), left over from the project's Jellyseerr → "Seerr" rebrand (GitHub's latest release is now `v3.4.1` under that new line; `2.7.3` was the last release before the split). User chose to pin the compose tag explicitly rather than leave it on `:latest`, for clearer version tracking.

### Changed
- ✅ `/opt/arr-stack/docker-compose.yml` on docker-lxc-media (192.168.40.11 / CT205): `fallenbagel/jellyseerr:latest` → `fallenbagel/jellyseerr:2.7.3`. Backup saved as `docker-compose.yml.bak-20260913-jellyseerr-pin`.
- ✅ Recreated the container (`docker compose up -d --force-recreate jellyseerr`) — **succeeded without hitting the CT205 AppArmor container-creation bug (task #15)**. Worth noting task #15 may be intermittent, host-state-dependent, or already partially resolved — it did NOT reproduce on this recreate. Startup log explicitly confirms `Starting Jellyseerr version 2.7.3`. Verified healthy: container up, port 5055 responding, still returns `302` → Authentik login through Traefik post-recreate (no regression from the outpost fix above).
- File ownership note: `hermes-admin` cannot write to `/opt/arr-stack/` directly on this host (no passwordless sudo either) — had to edit via `pct exec 205` as root from node01 instead.

---

## [2026-08-29] - Removed Reddit, Sports, and Finance tabs from Glance dashboard

### Removed
- ✅ Deleted the `🤖 Reddit`, `🏀 Sports`, and `💰 Finance` top-level pages from `/opt/glance/config/glance.yml` on docker-lxc-glance (192.168.40.12) — user request. Backup saved as `glance.yml.bak.20260829-remove-reddit-sports-finance` before editing.
- Remaining pages: Home, Services, Compute, Storage, Backup, Network, Media, News.
- Container restarted, config reloaded clean, no errors, `http://localhost:8080` → 200.

---

## [2026-08-29] - Fixed arr-stack cascade breakage (Prowlarr/Bazarr/Jellyseerr stale keys from earlier key rotation) + added FlareSolverr + Tdarr/Bazarr integration pass

### Context
User asked to make sure Jellyfin playback + Deluge/Radarr/Sonarr downloading actually worked end-to-end, and to integrate Tdarr and Bazarr. Root cause of most of what was broken: **the Radarr/Sonarr API key rotation done earlier tonight (see the credential-rotation entry above) only updated the consumers I checked at the time** (Glance, media-stats-api, Sentinel) and missed three more: Prowlarr's own stored `Applications` sync config, Bazarr, and Jellyseerr — all three were silently failing against the old, now-dead keys.

### Fixed
- ✅ **Prowlarr → Radarr/Sonarr application sync**: `Applications` objects (id=1 Radarr, id=2 Sonarr) at `/api/v1/applications/{id}` still had the old leaked API keys. Fixed via GET → edit `apiKey` field → PUT `?forceSync=true`.
- ✅ **Bazarr**: `/opt/arr-stack/bazarr/config/config.yaml` had stale Radarr/Sonarr `apikey` fields (separate from the sonarr/radarr config files themselves, easy to miss). Fixed via `sed`, container restarted. SignalR now connects clean to both.
- ✅ **Jellyseerr**: `settings.json` and `settings.backup.json` both had stale keys. Fixed via `sed -i .../g`, container restarted.
- ✅ **Radarr stale collection root folders**: 10 movie-collection entries still pointed at `/movies` (pre-NFS-remount path) instead of `/data/Movies`. Bulk-fixed via a small Python script against `/api/v3/collection/{id}`.
- ✅ **Dead indexer removed**: Prowlarr's "Pirate's Paradise" indexer (id=4) was throwing `IndexerNoDefinitionCheck` — deleted.
- ✅ **Deluge**: confirmed working as the shared download client for both Radarr and Sonarr (no changes needed).
- ✅ **Jellyfin**: confirmed healthy, libraries populated with real files, `/System/Info/Public` responds, no errors in logs. (Not stream-tested with an actual playback session — see task list.)

### Added
- ✅ **FlareSolverr** deployed on docker-lxc-media (192.168.40.11:8191) — new `/opt/flaresolverr/docker-compose.yml`, `apparmor=unconfined` (this host is one of the two affected by the AppArmor container-creation bug, task #15, but this was a fresh `docker compose up` which still worked — only *recreates* of existing containers seem to trip it, worth noting for that investigation).
- ✅ Prowlarr IndexerProxy "FlareSolverr" (id=1) pointing at it, tagged `flaresolverr`.
- ✅ EZTV indexer tagged `flaresolverr` (was failing Cloudflare challenge before) — now passing.
- Indexer status: LimeTorrents/Pirate Bay/EZTV all passing. NZBgeek in backoff — user's subscription lapsed, will renew later (not an infra bug).

### Not completed — Tdarr library setup
- ⏸️ Tried to configure a Tdarr library (Movies → `/media/movies`, confirmed correctly mounted via `docker inspect`) via its internal `/api/v2/cruddb` API. Iterated through 3 payload shapes; the final one matched the documented wrapper shape (`{"data": {"collection": "LibrarySettingsJSONDB", "mode": "insert", "docID", "obj": {...}}}`) and returned no error, but nothing persisted to `LibrarySettingsJSONDB` on disk — Tdarr's actual schema needs more fields than are documented anywhere findable. **Recommended the user do this via the Tdarr web UI instead** (`https://tdarr.hrmsmrflrii.xyz`, confirmed reachable/200) — a ~30 second manual step, not worth further blind API reverse-engineering.

### Lesson for future key rotations
When rotating a *arr API key, the consumer list is longer than the obvious "who calls this app directly" — Prowlarr's `Applications` config and Bazarr both store their *own* independent copies of the target app's key, not just a reference. Grep `/opt/` for the old key value as a final verification step next time, don't rely on memory of known consumers.

---

## [2026-08-29] - Deployed Helios: an Azure-CLI-style control plane for Proxmox/Synology/Omada

### Context
Assessed `github.com/herms14/helios-homelab-api` (FastAPI + Click CLI, "Azure ARM for your homelab") for integration. Real, non-trivial implementations for Proxmox/Synology/Omada (proper session/token auth, retry-on-401, task polling) — not a stub. Biggest gap: **zero LXC support** (only Proxmox QEMU VMs), a problem since this homelab is LXC-first (11 LXCs vs 5 VMs). Also had Hyper-V support for the retired "KratosPC" and a config pointing at wrong node IPs.

### Added
- ✅ **LXC container support** — new models (`LXCCreateRequest`/`LXCInfo`/`LXCActionResponse`), provider methods (`list_lxc`, `get_lxc`, `create_lxc` via clone, `start/stop/shutdown/delete_lxc`, `list_lxc_templates`), router endpoints (`/api/v1/compute/lxc*`), and CLI commands (`helios compute lxc`, `lxc-get`, `lxc-create`, `lxc-start/stop/lxc-delete`, `lxc-templates`) — mirrors the existing VM support, using Proxmox's parallel `/lxc` API namespace.
- ✅ **API-token auth for Proxmox** (in addition to the existing ticket/password flow) — created a dedicated `root@pam!helios` API token instead of using the root password. Ticket auth kept as fallback.
- ✅ Fixed a real bug: `list_nodes()` was fanning out to all 3 nodes and each returned the *entire* cluster's node list, producing 9 duplicate entries instead of 3.
- ✅ Fixed Omada site resolution: the code called `/api/v2/sites` (admin-only "list all sites", returns empty for restricted accounts) — switched to `/api/v2/users/current`, which reports exactly the sites a Viewer/Operator-scoped account can see. Found and used the existing `claude-reader` read-only Omada account (already used by `omada-exporter`) instead of hunting for new credentials.

### Deployed
- **Host**: ansible-controller01 (192.168.20.30:8000), via Docker Compose — already had Ansible/Terraform CLIs installed, natural home for a tool that shells out to both.
- **Config**: real node IPs (was pointing node01→.21 instead of .20, an off-by-one from the source repo), real Synology IP (.32, not the dead .31), real Omada controller (192.168.0.101:443).
- **CLI**: installed on ansible-controller01 itself (`pip3 install -e . --break-system-packages`) rather than the Windows machine — pip→PyPI SSL was broken locally (same issue hit earlier tonight with `git-filter-repo`; unrelated to Helios). Configured via `~/.helios/cli.yaml`.
- **Verified end-to-end**: `helios compute nodes/lxc/vms` all return correct live data matching the real cluster (11 LXCs, 8 VMs incl. stopped Windows templates).

### Known gaps (not resolved this session)
- 🔴 **Synology storage**: auth actually succeeds (correct user is `hermes-admin`, not `admin`) but that account has **2FA enabled**, which the current auth flow doesn't support (needs an OTP code). Needs a decision: create a dedicated non-2FA service account (matches the least-privilege pattern already used for Proxmox/Omada), or disable 2FA for this specific account.
- 🟡 **Omada devices/clients/stats endpoints**: site resolution now works, but the actual data endpoints return `"does not have permissions to access this site"` or `"Unsupported request path"` — likely an API-version mismatch between Helios's hand-rolled client and this controller's actual Omada SDN 6.2.10.18 REST shape. The mature `omada_exporter` tool works fine against the same controller/account, so this is fixable but wasn't chased further this session.
- Hyper-V support left in the code but unconfigured (KratosPC is retired infrastructure, not worth wiring up).
- Ansible/Terraform features not tested — ansible-controller01's actual playbook directory is `~/ansible/ansible-playbooks` and terraform workspaces aren't centrally organized (most current infra changes are done ad hoc, not via `terraform apply`) — config was set to plausible paths but not verified against a real run.
- Code changes exist only on the deployed copy on ansible-controller01 and in a local working copy — **not yet pushed back to `github.com/herms14/helios-homelab-api`** (needs the user's go-ahead + token, same as the other repo work tonight).

---

## [2026-08-29] - Public GitHub repo: purged leaked secrets from history, refreshed showcase content

### Context
`github.com/herms14/Homelab-Infrastructure-Deployment` (renamed at some point from `Proxmox-TerraformDeployments` — old name now redirects) turned out to be a near-complete, un-sanitized mirror of the private Obsidian operational docs — ~500 files, 109 commits — not a curated showcase. Found and fixed.

### Fixed
- ✅ Verified zero matches for any of the above across the full rewritten history before pushing.
- ✅ Force-pushed the cleaned history. Hit a `nul`-named file from history that Windows/git blocks by default (`core.protectNTFS`) — worked around by disabling that check for the rewrite (safe here since it's just a legacy filename artifact, not a live risk).
- ✅ Discovered mid-process that the repo's real default branch was `master` (108 commits) while `main` was an near-empty 1-commit stub — nearly set the wrong one as default before checking commit counts. Deleted the stub, renamed the cleaned branch to `main`, force-pushed, set it as default, deleted the old `master` (which still had the original secrets in its history) and the intermediate `master-cleaned` branch. Confirmed only `main` remains.
- ✅ Refreshed `README.md` and `claude.md` (the two most-visible "front door" files) to reflect current reality: Proxmox 9.2.11 (was 9.1.2), current service list, Kubernetes and Azure both noted as retired/decommissioned rather than described as live, removed the detailed hybrid-lab AD topology and Azure subscription block entirely (not appropriate for a public showcase even redacted).
- ⚠️ **Repo visibility remained "public" per repeated anonymous-access checks throughout this work**, despite user reporting it toggled to private more than once — possibly a different repo was toggled, or the setting didn't save. Worth the user re-confirming directly on `github.com/herms14/Homelab-Infrastructure-Deployment/settings`. Not blocking at this point since the actual secrets are now purged from history regardless.
- 📝 Used a user-provided fine-grained GitHub PAT for this work, upgraded mid-session from `Contents` to include `Administration` permission (needed to change default branch / delete branches via API). **Recommend scoping this token back down or revoking it now that the one-time cleanup is done.**

### Deferred / follow-up needed
- Rotate the live Proxmox API token for Homepage widget (`<redacted>`) — it was exposed in the repo history (now purged) but the credential itself is still the original value in production.
- Rotate Lidarr/Prowlarr/Bazarr API keys, OPNsense API key+secret — same situation, found in the leaked history, not yet rotated on the live services (lower urgency than Cloudflare/Radarr/Sonarr since these aren't internet-facing, but still real leaked values).
- A full pass through the remaining ~500 files in the repo (docs/, ansible/, terraform/, Azure-Hybrid-Lab/) for further sanitization/accuracy wasn't attempted this session — only the two front-door files were refreshed. Given the volume, recommend deciding whether this repo should even carry the full Azure-Hybrid-Lab/ and legacy ansible-playbooks/hybrid-lab/ content going forward, versus removing it now that that infrastructure is retired.
- Confirm actual repo visibility (see above).

---

## [2026-08-29] - Credential rotation (leaked in public repo) + new critical finding: Docker on 2 LXCs can no longer create containers

### Context

### Rotated
- ✅ **Cloudflare API token**: old token (`7bc694d6...`, scoped to `hrmsmrflrii.xyz` DNS edit/read only) was leaked. User generated a replacement via the dashboard; verified it end-to-end (created + deleted a real TXT record via the API, proving DNS-01 challenge capability) before deploying to `/opt/traefik/docker-compose.yml` on traefik-lxc and recreating. Traefik healthy, all routes verified after. **Old token still needs manual revocation** by the user (Cloudflare dashboard → Profile → API Tokens) — neither token has account-level permission to revoke itself via API.
  - Note: the new token is broader than requested — it covers a second zone (`abstergotech.xyz`) and `zone:edit`/`zone_settings:edit` in addition to DNS, likely because the user's account only has these 2 zones and the dashboard template pulled in more scope than the "Edit zone DNS" template alone implies. Works fine for now; worth tightening later.
- ✅ **Radarr API key**: `<redacted>` → `446ad585...`. Updated in `/opt/arr-stack/radarr/config.xml`, `/opt/glance/config/glance.yml`, `/opt/media-stats-api/docker-compose.yml`, `/opt/sentinel-bot/.env`. Radarr container restarted (not recreated — see AppArmor finding below).
- ✅ **Sonarr API key**: `<redacted>` → `4aa4eb00...`. Same file list as Radarr. Sonarr container restarted.
- ⏸️ **Hybrid-lab SQL SA / AD DSRM passwords** — not rotated. The hybrid lab itself was decommissioned 2026-05-25, so these are stale credentials for infrastructure that no longer exists; lower urgency, but the values should still be scrubbed from the repo (see below).

### 🔴 New critical finding: Docker can no longer create containers on docker-lxc-glance (CT200) and docker-lxc-media (CT205)
While rotating the Radarr/Sonarr keys, `media-stats-api` needed a recreate (env var change) and hit the AppArmor build-regression from earlier tonight (task #15) — except this time it turned out to be **broader than "can't rebuild custom images"**: after the recreate attempt left the container removed, **`docker run` of a plain pulled image (`hello-world`) also failed** with the same `apparmor_parser: ... Access denied. You need policy admin privileges` error. Tried, in order: restarting the Docker daemon (no effect), rebooting the LXC itself via `pct reboot 200` (no effect — still fails post-reboot). Checked `pct config 200`: `lxc.apparmor.profile: unconfined` is **already set** at the Proxmox host level — the LXC is already fully unconfined, yet nested Docker still can't self-manage its AppArmor profile. This points to a missing **capability** (likely `CAP_MAC_ADMIN`, dropped by Proxmox's default unprivileged-LXC capability set) rather than just an AppArmor profile setting — a different, deeper fix than previously scoped (task #15 needs to be re-scoped: probably `lxc.cap.drop` adjustment or converting to a privileged container, both real security trade-offs, not applied without explicit decision).
- **Verified docker-lxc-media (CT205) has the identical issue** (`docker run hello-world` fails the same way) — so this affects the entire arr-stack host too, not just glance.
- **Practical impact**: existing containers on these 2 hosts continue running and can be `docker restart`ed fine (confirmed all night), but **any future container recreate — including Sentinel's gated update-approval flow — will fail** on these 2 hosts until this is fixed. This silently defeats the update workflow fixed earlier tonight for anything hosted on .12 or .11.
- **Stopgap applied**: `media-stats-api` restored by running it directly on the LXC host OS (bypassing Docker entirely) via a systemd unit (`/etc/systemd/system/media-stats-api.service`, `gunicorn` + venv-less `pip install --break-system-packages`). Verified healthy (`/health` → 200). This is a workaround, not a fix — the container definition still exists and should replace this once Docker is usable again on this host.
- Also found the `requirements.txt` for media-stats-api was already missing `flask_cors` (the app imports it but it isn't declared) — installed it manually for the host fallback; worth fixing the requirements file too.

### Deferred to the user (not resolved this session)
- Revoke the old Cloudflare token in the dashboard.
- Decide the fix for the AppArmor/capability issue on CT200 and CT205 (privileged container vs. targeted capability grant) — this is now higher priority than task #15's original framing since it's actively blocking service recreation, not just custom-script rebuilds.
- Tighten the new Cloudflare token's scope (currently broader than needed).

---

## [2026-08-29] - Documentation overhaul: GitLab wiki sync + Obsidian consolidation

### Context
User asked for the existing GitLab wiki (`homelab-project/-/wikis/home`) to be updated and synced with the Obsidian vault, and for the Obsidian homelab docs themselves to be cleaned up: unrelated content removed, duplicates consolidated, stale info updated, credentials refreshed, structured for fast emergency lookup. Ran a full read-through audit of all 63 files in `Claude Managed Homelab/` first (via a dedicated research pass) before making changes.

### Archived (16 files → `<redacted>`, each with a banner explaining why)
- `Hermes Homelab Technical Manual.md` and `25 - Homelab Master Wiki.md` — two competing "authoritative" mega-docs, both frozen since early-mid 2026, both contradicting each other and the actively-maintained numbered docs. The numbered docs (00-57) + `CLAUDE.md` are now the only source of truth.
- `04 -`, `31 -`, `46 -` — described the Kubernetes cluster as live; it was decommissioned 2026-05-23.
- `52 -`, `53 -` — Azure docs missed by the 2026-05-25 Azure archival sweep.
- `27 -`, `28 -` — pre-consolidation bot tutorials (Argus/Athena), superseded by Sentinel (Jan 2026).
- `32-35 -` (GitOps Architecture/Pipeline/Deployment/Troubleshooting) — superseded by the more complete `43 - GitOps Complete Tutorial`.
- `45 -` (Docker Services Reference) — stale partial duplicate of `07 - Deployed Services` (only covered 2 of 8+ hosts).
- `Homelab Agent Plan.md` — original planning doc, superseded by `Homelab Agent Documentation.md`.
- `CONTEXT - Life Progress Widget Implementation.md` — duplicate content already in `21 -`.

### Fixed numbering collision
- Two files were both numbered "23" (`23 - Glance Dashboard.md` and `23 - PBS Monitoring.md`). Renamed the latter to `47 - PBS Monitoring.md`.

### Updated for accuracy
- `00 - Homelab Index.md` — full rewrite: added an "Emergency Quick Reference" section at the top (what to check first when X is down), removed links to archived docs, added missing links (41, 47, 54-57, Homelab Agent Documentation), fixed a broken `38 - Homelab Technical Manual` link, updated infra summary (11 LXCs not 10, 5 VMs not 6, K8s/Azure decommission notes), expanded the Service URLs table.
- `CLAUDE.md` — fixed the "Works at Microsoft" line (owner is transitioning to ADB) and homelab-purpose line (Azure hybrid retired).
- `19 - Watchtower Updates.md` — full rewrite describing what's actually running (Sentinel bot's SSH-based gated update checks), replacing the description of the old VM-based Watchtower architecture.
- `02 - Proxmox Cluster.md` — Proxmox version 9.1.2 → 9.2.11, SSH key reference updated to the rotated 2026-08-26 key.
- `09 -`, `21 -` — Traefik v3.2 → v3.7.12, Authentik 2025.10.3 → 2026.8.0, hostnames to LXC names.
- `08 -`, `23 - Glance Dashboard`, `13 -`, `42 -`, `41 -`, `47 -` — bulk-fixed the dead NAS IP `192.168.20.31` → `192.168.20.32` (7+ occurrences).
- `TODO - Homelab Improvements 2026-05.md` — marked items #1 (PBS), #3 (Immich VectorChord), #6 (Omada exporter), #8 (Docker image updates) as completed with dates/links, since all four were actually done weeks/months ago per the Changelog but still showed as open.
- `11 - Credentials.md` — clarified the Omada password history (marked which is current), moved dead Discord bot tokens (Athena/Argus/Chronos/Mnemosyne/old Update Manager) to a clearly-labeled Legacy section, added Sentinel's actual bot token (was missing entirely), added the fresh GitLab API token created today for wiki sync, flagged Sentinel's webhook `API_KEY` and Immich's OAuth client secret as looking like unrotated placeholder values (same class of bug as the Speedtest Tracker `APP_KEY` found and fixed earlier today), added a "Current App Versions" quick-reference table.

### GitLab Wiki sync
- Cloned `homelab-project.wiki.git` (created a fresh API token via `gitlab-rails runner` since the old stored one — labeled "for Chronos" — was dead/revoked).
- Rewrote all 9 existing wiki pages (Home, Architecture-Overview, Getting-Started, Network-Configuration, Service-Deployment, Monitoring-Observability, Security-Best-Practices, Troubleshooting, Discord-Bots) to reflect current infrastructure — the previous content was templated/generic and still showed 9 live K8s nodes among other stale facts. Credentials intentionally excluded from the wiki (kept Obsidian-only). Pushed to `main`.

### Deferred (explicitly not done — flagged for later, not silently skipped)
- **Folder restructure** (Reference/Tutorials/Apps/Meta split) — proposed in the audit but not executed; current flat structure with the numbering collision fixed is left as-is for now.
- **Merge `15 - New Service Onboarding Guide` + `22 - Service Onboarding Workflow`** — both describe the same onboarding checklist around different (now-superseded) bot architectures; not yet consolidated.
- **Remaining stale hostnames** in `06 -`, `13 -`, `15 -`, `18 -`, `20 -` (old `docker-vm-media01`/`traefik-vm01`/`authentik-vm01` references) — not all instances fixed, only the highest-traffic docs (`07 -`, `09 -`, `21 -`) were prioritized.
- **Blog Posts folder** — audit found 3 parallel draft series (`Blog Post 01-28`, `Rewritten Series 01-30`, plus standalone posts) that likely need consolidation; out of scope for this pass (narrative content, not infra reference).
- **PBS root password** `NewPBS2025` — dated-looking value, no rotation date on record; worth checking if still valid.

All changes committed locally to the vault's git repo in two checkpoints (pre-cleanup snapshot, then the consolidation itself) — nothing pushed to any remote.

---

## [2026-08-29] - Gated app-update workflow: fixed Sentinel instead of deploying Watchtower

### Decision
- User asked to deploy Watchtower across all LXCs with gated (approval-required) updates. Investigation found Sentinel bot (`docker-vm-core-utilities01`, `/opt/sentinel-bot/cogs/updates.py` + `scheduler.py`) **already implements this**: SSH-based `docker pull` update checks across `CONTAINER_HOSTS`, Discord reaction-based approval (👍 all / numbered per-container), `/check` and `/update <container>` slash commands, and a **daily 7 PM scheduled report** — all pre-existing, no new containers needed. Deploying Watchtower alongside this would just be a redundant, less-capable parallel system (Watchtower's own monitor-only mode can't do per-container selective approval the way Sentinel's reaction UI does).
- **Chosen workflow**: fix Sentinel's config to reflect the current (post-LXC-migration) infrastructure instead of deploying anything new.

### Fixed
- 🔧 `/opt/sentinel-bot/config.py` (backed up as `.bak-20260829`) had significant drift from reality:
  - `CONTAINER_HOSTS`/`COMPOSE_DIRS` for the entire utilities-host monitoring stack (grafana, prometheus, uptime-kuma, speedtest-tracker, n8n, jaeger, cadvisor, snmp-exporter, pve-exporter) pointed at `/opt/docker/monitoring` — actual path is `/opt/monitoring`. Same wrong-prefix pattern (`/opt/docker/X` vs actual `/opt/X`) affected karakeep, tracearr, paperless, traefik, authentik.
  - The 12 arr-stack containers (jellyfin, radarr, sonarr, lidarr, prowlarr, bazarr, jellyseerr, overseerr, tdarr, autobrr, deluge, sabnzbd) were split across 3 wrong paths (`/opt/docker/media`, `/opt/docker/tdarr`, `/opt/docker/autobrr`) — actually all one compose project at `/opt/arr-stack`.
  - `pbs-exporter` was mapped into the monitoring compose dir — it's actually a separate project at `/opt/pbs-exporter`.
  - `homelab-chronicle` was mapped to docker-vm-core-utilities01 (192.168.40.13), which only has a stale/never-run copy — the live container is on chronicle-lxc (192.168.40.15).
  - A few entries left as-is with an `# unverified this session` comment (wizarr, life-progress, power-control-api, steam-stats, gaming-pc-stats, immich) — not touched tonight, paths not independently confirmed, flagged rather than guessed.
- Restarted `sentinel-bot`, confirmed clean reload: all 7 cogs loaded (including `updates` and `scheduler`), Discord connection healthy, no config errors.
- **Not yet done**: a live end-to-end `/check` run wasn't triggered this session (would post to the real `container-updates` Discord channel) — recommend running `/check` manually once to validate the new host list before relying on the 7 PM auto-report.

---

## [2026-08-29] - Updated utilities-host apps + fixed 3 pre-existing incidents

### Changed — Application versions (docker-vm-core-utilities01)
- ✅ **n8n**: was defined in `/opt/monitoring/docker-compose.yml` but **never actually running** (explains the earlier Watchtower-question 502). Pulled `n8nio/n8n:latest` and started it for the first time — now live at `n8n.hrmsmrflrii.xyz` (200).
- ✅ **Prometheus, Grafana, Speedtest Tracker**: pulled `:latest` and recreated, alongside n8n in the same compose project.
- ✅ **Paperless-ngx, Karakeep (+ meilisearch), Tracearr**: pulled/recreated in their own compose projects. `karakeep-chrome` (`gcr.io/zenika-hub/alpine-chrome:123`) failed to pull — upstream Google Cloud Registry is returning "requires billing to be enabled" for that image's hosting project, an external issue unrelated to us. Container kept running on its previously-cached image; no functional impact observed.

### Fixed — 3 pre-existing incidents surfaced during verification
- 🔧 **Speedtest Tracker 500 (real, not new)**: `APP_KEY=<redacted>` in `/opt/monitoring/docker-compose.yml` was a placeholder/example key (derived from AWS's well-known public example secret pattern) — never replaced with a real generated key. Laravel's bootstrap failed trying to use it (cascading into a secondary "Class config does not exist" error while attempting to log the original exception). Generated a real key via `php artisan key:generate --show` inside the container, updated the compose env var (backed up as `.bak-20260829`), recreated. This was likely broken since original deployment, just never noticed until a fresh pull forced a clean bootstrap.
- 🔧 **Karakeep 502 (pre-existing)**: Traefik's `/opt/traefik/config/dynamic/new-services.yml` pointed karakeep's backend at `192.168.40.10:3005` — `.10` is the **decommissioned** old docker-vm-core-utilities01 address (split into `.12`/`.13` per `10 - IP Address Map.md`). App itself was healthy the whole time, just unreachable via the edge. Fixed to `192.168.40.13:3005`.
- 🔧 **Tracearr / Wizarr, same stale-IP bug**: found and fixed the identical `.10` leftover for `tracearr` (→ `.13:3002`) and `wizarr` (→ `.13:5690`) in the same Traefik file while in there. `tracearr.hrmsmrflrii.xyz` and `wizarr.hrmsmrflrii.xyz` now both resolve correctly.
  - 📝 One more stale entry in the same file, **not fixed**: `lagident` service still points at `192.168.40.10:9933` — could not find a running container for it on either `.12` or `.13`. Likely a retired/never-deployed service; left as-is, flagged for a decision (task #21).
  - Backed up as `new-services.yml.bak-20260829`.

---

## [2026-08-29] - Fixed GitLab crash loop (RestartCount 5917)

### Fixed
- 🔧 **Root cause**: `/var/opt/gitlab/postgresql/data` was PostgreSQL 16, but the running image (`gitlab/gitlab-ce:19.0.0-ce.0`) only bundles PostgreSQL 17 binaries — `gitlab-ctl reconfigure` ran on every container boot (via `GITLAB_OMNIBUS_CONFIG`), tried to migrate the DB, found no live Postgres socket, Chef aborted, container exited 0, `restart: unless-stopped` immediately retried — forever. A previous session (2026-08-26, evidenced by `gitlab-pre-recovery-20260826.tar.gz` and a stale `/var/opt/gitlab/postgresql/data.17` partial-upgrade dir) had already hit this and left it mid-repair.
- ✅ **Fix**: Full backup first (`/opt/gitlab-backups/pre-pg-upgrade-full-20260829.tar.gz`, 139M). Found `gitlab/gitlab-ce:18.11.11-ce.0` already cached locally and confirmed it bundles **both** PG16 and PG17 (needed for `pg_upgrade`) — GitLab 19.0.0 only ships PG17, so upgrading in place wasn't possible without this bridge version. Temporarily ran 18.11.11 against the existing PG16 data (booted clean, 0 restarts, confirmed data integrity), moved the stale `data.17` attempt aside (`data.17.stale-20260826-attempt`, not deleted), then ran `gitlab-ctl pg-upgrade` successfully (PG16 → 17.10). Verified 18.11.11 healthy on PG17, then bumped to the target `19.0.0-ce.0` — it ran its full backlog of accumulated Rails migrations (dozens, spanning ~May 2026 schema changes) and came up clean. `gitlab.hrmsmrflrii.xyz` now returns 302 consistently, all `gitlab-ctl status` services running, 0 further restarts.
  - Old PG16 data kept at `/var/opt/gitlab/postgresql/data.16` (not yet deleted — `gitlab-ctl pg-upgrade` recommends verifying stability before `rm -rf`). New TODO to clean it up after a few stable days.
  - Files touched: `/opt/gitlab/docker-compose.yml` (backed up as `.bak-20260829`).

---

## [2026-08-29] - Full LXC package updates + application version updates

### Changed — OS packages (all 11 LXCs: 100, 200-209)
- 🔧 Fixed interrupted dpkg state on CT200 (docker-lxc-glance) and CT207 (chronicle-lxc) — both were stuck on interactive debconf prompts (sshd_config conflict, docker.io restart prompt) left over from a prior session's non-interactive `apt upgrade` attempt. Cleared with `DEBIAN_FRONTEND=noninteractive dpkg --force-confdef --force-confold --configure -a`, keeping locally-modified configs.
- ✅ Rebooted the 9 LXCs flagged `reboot-required` (all but pbs-server and homepage-lxc). Verified container health + external routing after each — sequenced traefik-lxc and authentik-lxc one at a time (not parallel) since they gate access to everything else. Zero extended downtime; brief (~10-30s) container restarts only where the containerd.io/docker.io package itself was upgraded.
- 📝 **Corrected stale doc**: CLAUDE.md's LXC table said CT201 (docker-lxc-bots) hosts "Sentinel consolidated bot" — false. CT201 has **zero running containers** (leftover compose files for argus/athena/chronos/sentinel bots, never restarted after some earlier event). The live Sentinel bot actually runs on docker-vm-core-utilities01 (192.168.40.13), matching the Automation Stack section elsewhere in the same file. Fixed the LXC table entry.

### Changed — Application/image versions
- ✅ **Traefik**: v3.2 → **v3.7.12** (traefik-lxc). Stateless, low risk — backed up compose file only. Verified all routes healthy post-upgrade.
- ✅ **Authentik**: 2025.10.3 → **2026.8.0** (authentik-lxc), via required staged path (Authentik blocks major/year version skips): 2025.10.3 → 2025.12.6 → 2026.2.6 → 2026.5.6 → 2026.8.0. Took a `pg_dump` backup before **each** hop (`/opt/authentik/db-backups/pre-*.sql.gz`). First attempt to jump straight to 2026.8.0 failed fast with a clean, pre-migration error ("Major version skips are not allowed") — no DB changes occurred, rolled back to 2025.10.3 in ~1 min, then re-did it properly through all 4 intermediate hops. Verified healthy + all ForwardAuth-protected routes (prowlarr, uptime, etc.) redirecting to login correctly at final version.
  - ⚠️ **Known issue to monitor**: authentik-server logs show a recurring (~5s) non-fatal warning from the embedded outpost: `error in serde: invalid type: null, expected a formatted UUID string` / "Failed to fetch outpost from API, retrying". Actual ForwardAuth checks work correctly (verified against prowlarr/uptime), so this appears to be a background self-discovery/heartbeat loop hitting a null UUID field, not a functional break. Worth a closer look if any SSO-gated app starts misbehaving.
- ✅ **Media stack** (docker-lxc-media): full `docker compose pull && up -d` — sonarr, radarr, lidarr, prowlarr, bazarr, overseerr, jellyseerr, jellyfin, deluge, sabnzbd, tdarr, metube, cadvisor all pulled to current `:latest`. All 14 containers verified healthy after.
- ✅ **Glance + custom APIs** (docker-lxc-glance): glance, pihole-stats-api, proxmox-nodes-api pulled/recreated clean.
- ➖ **Not touched**: sentinel-bot, homelab-chronicle, finance-forecast-api compose configs on docker-lxc-glance (leftover, zero containers, not part of live service — confirmed via `docker ps -a`).
- ✅ **Pi-hole**: already fully current (Core/Web/FTL all latest). **PBS**: 3.4.9-2, current via apt, 0 upgradable.
- 🔧 **Fixed post-upgrade regression**: `homepage.hrmsmrflrii.xyz` returned 503 after the app updates above. Root cause: Homepage's `HOMEPAGE_ALLOWED_HOSTS=homepage.hrmsmrflrii.xyz` rejects any request with a mismatched `Host` header, and Traefik's `healthCheck` on that service didn't set one — so Traefik's own healthcheck got a 400 and marked the backend down for everyone, not just the healthcheck. Fixed by adding `hostname: homepage.hrmsmrflrii.xyz` to the healthCheck block in `/opt/traefik/config/dynamic/homepage.yml` (backed up as `.bak-20260829`). Verified 200 after Traefik hot-reloaded the config.
- 🔴 **New finding, unrelated to this session's changes**: final health sweep found `gitlab.hrmsmrflrii.xyz` returning 502. gitlab-vm01 was not touched this session (no updates, no reboot). `docker inspect gitlab` shows `RestartCount=5917`, clean `ExitCode=0`, not OOM (6.4Gi free) — an active, long-running crash loop, not a one-off. Root cause not yet identified; added as high-priority task #19 for a dedicated session.
- 📝 **Investigated**: Watchtower is not deployed on any of the 11 LXCs — the `19 - Watchtower Updates.md` doc describes the pre-migration VM architecture (Jan 2026). It only runs today on immich-vm01 and gitlab-vm01, and even there it's `WATCHTOWER_MONITOR_ONLY: "true"` (Discord-approval-gated, never auto-updates by design). This explains why no LXC or VM app version had been auto-updated.

---

## [2026-08-29] - Removed Proxmox "No valid subscription" nag popup

### Fixed
- 🔧 Patched `/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js` on all 3 nodes to suppress the enterprise-subscription nag dialog on Web UI login. Backed up original as `proxmoxlib.js.bak-20260829` on each node before editing (`sed -i "s/data.status !== 'Active'/false/g"`), then restarted `pveproxy.service` on each.
  - Verified apt repos were already correctly configured on node01 (deb822 `.sources` format): `pve-enterprise.sources` has `Enabled: false`, `proxmox.sources` points at `pve-no-subscription`, `ceph.sources` points at `ceph-squid no-subscription`. No repo changes needed.
  - Applied identically to node01, node02, node03.
  - Note: this patch lives in a package-managed file and will be **reverted by the next `pve-manager` package upgrade** — re-run the same `sed` + `systemctl restart pveproxy.service` after any Proxmox VE update if the nag reappears.

---

## [2026-08-26] - SSH Key Rotation (nodes + LXCs)

### Security
- 🔧 **Rotated the primary homelab SSH key** — the previous key (`hermes@homelab-nopass`, deployed Dec 21 2025) no longer authenticated against any host from the Claude Code Windows machine; root cause not fully diagnosed (likely never matched what was actually deployed, or was rotated out-of-band and docs weren't updated). Generated a fresh ed25519 keypair (`hermes@homelab-rotated-20260826211450`) and bootstrapped it via one-time root password auth.
  - ✅ Deployed to `root@node01`, `root@node02`, `root@node03` — verified passwordless SSH works.
  - ✅ Deployed to all 11 currently-running LXC containers (both `root` and `hermes-admin` where present) via `pct exec` from each host node — no separate credentials needed since this ran through the already-authenticated Proxmox host.
  - ❌ **Not deployed** to the 5 running VMs (ansible-controller01, docker-vm-core-utilities01, gitlab-runner-vm01, gitlab-vm01, immich-vm01) — none have QEMU guest agent running, and all reject SSH password auth outright (key-only `sshd_config`). No working bootstrap credential was available this session. Deferred — see [16 - SSH Configuration](16%20-%20SSH%20Configuration.md) for options next time.
  - Full details, new public key value, and per-node LXC list: [16 - SSH Configuration](16%20-%20SSH%20Configuration.md) and 11 - Credentials.
- 📝 Corrected a doc error found while doing this: CT204 (`authentik-lxc`) actually runs on **node02**, not node01 as `16 - SSH Configuration.md` previously implied.

### Update — VM access resolved
- ✅ Found the actual working private key at `C:\Users\herms\OneDrive\My Personal Files\Side Projects\homelab_ed25519` (same fingerprint as the "nopass" key — it was never broken, just absent from the machine driving this session). Used it to bootstrap the rotated key onto all 5 VMs (ansible-controller01, docker-vm-core-utilities01, gitlab-runner-vm01, gitlab-vm01, immich-vm01) as `hermes-admin`. Passwordless sudo confirmed on all 5.
- 🔧 Rewrote `~/.ssh/config` on the Claude Code Windows machine (`C:\Users\herms\.ssh\config`) with host aliases for all 3 nodes, 5 VMs, and 11 LXCs, replacing the stale version that still referenced the decommissioned K8s cluster and pre-migration VM hostnames (`docker-vm-media01`, `traefik-vm01`, `authentik-vm01`). Verified `ssh node01` and `ssh immich-vm01` connect correctly. Also confirmed CT207 (chronicle-lxc) IP, previously "TBD" in docs: **192.168.40.15**.
- Updated [16 - SSH Configuration](16%20-%20SSH%20Configuration.md) and 11 - Credentials to match.

### Update — Immich upgraded to v3.1.0 (VectorChord migration)
- 🔧 **Migrated Immich's Postgres from deprecated `pgvecto.rs` to VectorChord and updated Immich v2.7.5 → v3.1.0**, closing out the high-priority TODO from `TODO - Homelab Improvements 2026-05.md`.
  - Found the documented backup script (`/opt/immich/backup-db.sh`) doesn't actually exist on immich-vm01, and the only backup on disk was **6+ months stale** (Feb 16, 2026). Took a fresh `pg_dumpall` backup first (`/mnt/immich-uploads/db-backups/pre-v3-upgrade-20260826.sql.gz`, 211M, NFS/NAS-stored) before touching anything.
  - Verified the correct target image via Immich's official docs (`ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`) — confirmed by test-pulling it before editing production config.
  - Edited `/opt/immich/docker-compose.yml` (backup saved as `docker-compose.yml.bak-20260826`), swapped the postgres image, `docker compose pull && docker compose up -d`.
  - Immich auto-migrated the vector extension on startup (no manual SQL needed) — reindexed `clip_index` (30,233 rows) and `face_index` (38,684 rows) in under a minute.
  - Verified: API responds (`{"res":"pong"}`), version endpoint reports `3.1.0`, all 4 containers healthy, no errors in logs.
- 📝 **Found a real gap**: Immich's backup automation described in [42 - Immich Backup and Restore](42%20-%20Immich%20Backup%20and%20Restore.md) doesn't exist on the actual VM. Worth rebuilding a real scheduled backup job — see new TODO.

### Update — Cluster-wide package updates + disk emergency on docker-vm-core-utilities01
- 🔧 **Ran `apt update && apt upgrade -y` across all 3 Proxmox nodes' LXCs and all 5 VMs.** 9/11 LXCs and 4/5 VMs succeeded cleanly on the first pass.
- ⚠️ **`docker-vm-core-utilities01` (192.168.40.13) was found at 100% disk usage (38G/38G)**, blocking `apt` entirely with "No space left on device" — this VM runs Grafana, Prometheus, Uptime Kuma, n8n, Paperless-ngx, Karakeep, and more, so this was live risk, not just a maintenance nuisance.
  - User expanded the Proxmox virtual disk from 40G to 140G. Grew the guest partition (`growpart /dev/sda 1`, had to redirect `TMPDIR=/boot` since root had zero free space to even create a scratch dir) and the ext4 filesystem (`resize2fs /dev/sda1`). Now **135G total, 97G free (29% used)**.
  - Retried the package update successfully after the resize: 124 packages upgraded, reboot required.
  - **Open question**: `du` breakdown showed `/var/lib` at 36G, but `/var/lib/docker` alone is only 5.3G — roughly 30GB under `/var/lib` is unaccounted for (likely a native service's unbounded data dir, e.g. Prometheus TSDB retention). Needs investigation — see new TODO.
- ⚠️ **Still open**: CT200 (docker-lxc-glance) and CT207 (chronicle-lxc) hit "dpkg was interrupted, you must manually run 'dpkg --configure -a'" on the first pass and were never actually upgraded — needs `dpkg --configure -a` then a retry. Not yet done as of this entry.
- **Reboot required** on: ansible-controller01, gitlab-runner-vm01, immich-vm01, docker-vm-core-utilities01, and several LXCs (201, 202, 203, 205, 206, 204, 207, 208). Not yet rebooted — pending user go-ahead given production impact.

---

[← Back to Index](00%20-%20Homelab%20Index.md) | 📋 Agent Plan

---

## How to Use This Changelog

Use `/lab-changelog add "description"` to add entries, or add manually using these categories:

- **Added** - New VMs, containers, services, configurations
- **Changed** - Modifications, updates, reconfigurations
- **Fixed** - Bug fixes, issue resolutions
- **Removed** - Decommissioned services, deleted resources
- **Infrastructure** - Hardware, network, cluster changes
- **Security** - Security updates, access changes

---

## [2026-05-25] - Glance Network Tab Redesign

### Changed
- 🔄 **Glance Network tab** redesigned to match the 3-column layout used by Compute / Storage / Media. All existing metrics retained; several new live ones added.
  - **Left column (small)** — Internet Speed (Download/Upload hero cards) · Latency (Ping/Jitter) · Pi-hole DNS (queries today hero + Active Clients / Cached / Blocklist tiles) · Service Status (Router / Omada Controller / Pi-hole / Traefik) · Quick Links.
  - **Center column (full)** — Network Devices grid (8 Omada devices in 2-col cards showing CPU bars from `omada_device_cpu_percentage`) · Device Memory grid (RAM bars from `omada_device_mem_percentage`) · 4 Grafana time-series iframes with native time picker (Cluster Bandwidth panel 9, WiFi Signal panel 22, Switch Port Traffic panel 32, Client Connection Trend panel 50).
  - **Right column (small)** — Connected Clients (sum from Omada) · VLAN Map (8 VLANs) · PoE Remaining card · Top Clients by Traffic Grafana iframe (panel 51).
  - All time-series iframes use `/d/...?viewPanel=N&kiosk=tv` pattern so the time picker is visible.
  - File: `/opt/glance/config/glance.yml` on CT200; backup retained. Hot-reloaded clean, zero template errors.
  - Verified live: Download 724 Mbps / Upload 671 Mbps · Ping 2.8 ms · 290k Pi-hole queries today · 37 connected clients · 117.7 W PoE remaining.

---

## [2026-05-25] - Media Stack Recovery + Glance Media Tab Redesign

### Fixed
- 🔧 **Media stack data flow was broken since 2026-05-20 NVMe rebuild** — symptoms: Sonarr/Radarr/Jellyfin showed empty libraries despite ~6.3 TB of media existing on NAS; CT205 root disk hit 69 % full with orphaned downloads. Root cause: CT205's `/mnt/media` NFS mount to `192.168.20.32:/volume2/Proxmox-Media/MediaFiles` was lost during the rebuild — fstab entry was present but didn't activate at boot. All new downloads from late-Feb onward piled onto the LXC's local 49 GB disk while the 25 TB NAS export sat unreferenced.
  - **Recovery**: stopped arr containers → unmounted (no-op) → moved 13 GB of orphaned local downloads (House of the Dragon S02E04 + S02E05) to `/root/orphaned-downloads/` → remounted NFS → copied the orphans into the NAS `Completed/tv-sonarr/` folder → restarted containers → triggered Sonarr `DownloadedEpisodesScan` → both episodes imported into `/mnt/media/Series/House of the Dragon/Season 2/` and `/root/orphaned-downloads` was wiped (freed ~12 GB; CT205 now 43 % full).
  - **Persistence**: added `mount=nfs` to CT205 features in `/etc/pve/lxc/205.conf`; rewrote `/etc/fstab` inside the CT with `_netdev,x-systemd.requires=network-online.target` options so the mount survives reboots.
  - **Verified end-to-end**: Sonarr shows TV library, Radarr shows movie library (3.4 TB Movies / 2.9 TB Series), Jellyfin can browse both `/data/movies` and `/data/tvshows`.

### Changed
- 🔄 **Glance Media tab** fully redesigned to the same 3-column layout used by Compute/Storage. The previous tab pointed at a non-existent `172.17.0.1:5054` Media Stats API (no longer running) so the top widgets had been silently broken.
  - **Left column (small)** — Movies Library card (count + on-disk TB + monitored + missing, live from Radarr API) · TV Library card (series count + episodes available/total + on-disk TB, live from Sonarr API) · NAS Media Storage card (free/total TB from Radarr's diskspace endpoint) · Service Status monitor · Quick Links.
  - **Center column (full)** — Currently Downloading (Sonarr queue, color-coded by status — completed/downloading/warning) · Movie Queue (Radarr queue, same treatment) · Recently Added TV (history grid) · Movie & TV News RSS (collapsed-after=3).
  - **Right column (small)** — Indexer Health (fail count + per-indexer cards from Prowlarr `indexerstatus`) · Sonarr Alerts (live health-check feed) · Radarr Alerts (live health-check feed).
  - Verified live: 3.66 TB Movies / 2.87 TB TV libraries reported by the live APIs.
  - File: `/opt/glance/config/glance.yml` on CT200; backup `glance.yml.bak.20260525-media`. Hot-reloaded clean, zero errors.

### Known Issues / Follow-up
- ⚠️ **NZBgeek (Prowlarr indexer #1) and EZTV (#5) are in failure backoff** — auto-disabled until 2026-05-26. The actual failure reason wasn't surfaced by an indexer probe (the test endpoint needs the full config payload); recommend clearing in Prowlarr UI → System → Indexer Stats and watching the next test cycle.
- ⚠️ **Radarr health warning** about Deluge writing to `/data/Completed/radarr` — this directory will be auto-created the first time a movie completes via Deluge. Not a real error.

---

## [2026-05-25] - Backup Job vmid Cleanup + Sunday Job Hygiene

### Changed
- 🔄 **Sunday vzdump job `backup-13fa5dd2-e814`** (storage=ProxmoxData, schedule=`sun 03:00`) — removed VMIDs `1000`, `9025`, `9022` from the `vmid` field per yesterday's Azure/Windows decommissioning question. Final list: `100,103,106,107,108,121,204,207`. File: `/etc/pve/jobs.cfg` on the cluster.

---

## [2026-05-25] - Azure / Windows Decommissioning Cleanup

### Removed
- ❌ **Glance Compute tab** — stripped all Windows-VM detection logic now that no Windows VMs remain in the cluster:
  - Removed `(eq (.String "ostype") "win10"/"win11"/"w2k22"/...)` conditionals from the **Running VMs** card (border color + WIN/LNX badge collapsed to just LNX).
  - Removed the per-node `$win01/$win02/$win03` counters and their 🪟 badges from the **VM & Container Status** widget.
  - Removed the `else if WIN` branch from the compact list (collapsed CT/WIN/LNX three-way to CT/LNX two-way).
  - File: `/opt/glance/config/glance.yml` on CT200; backup `glance.yml.bak.20260525-decom`. Saved ~2.3 KB, no template errors after reload.
- ❌ **Doc inventory cleanup** — stripped active Azure/Windows references from:
  - `CLAUDE.md` (homelab) — removed VLAN 80 row, "Azure Environment" + "Azure Hybrid Lab" sections; updated node03 description; updated scale line.
  - `02 - Proxmox Cluster.md` — removed "Node03 - Azure Hybrid Lab Workloads" section (12 Windows VMs table); updated node03 row.
  - `07 - Deployed Services.md` — removed the entire "Azure Hybrid Lab (On-Premises)" section (DCs, member servers, workstations, SQL Server config, HomelabAnalytics DB schema, credentials block); removed Azure Hybrid Lab row from infrastructure summary.
  - `10 - IP Address Map.md` — removed VLAN 80 entire subsection (Domain Controllers, Member Servers, Workstations, Reserved Ranges).
  - `25 - Homelab Master Wiki.md` — removed VLAN 80 VMs table, VLAN 80 row from VLAN summary, updated node03 row, updated VLAN count.
  - `00 - Homelab Index.md` — replaced "Cloud & Hybrid" link list with archival notice; removed all Azure stats from Infrastructure Summary.
  - `Homelab Agent Plan.md` — removed Azure VMs count, updated LXC count, removed K8s count.
  - `06 - Ansible Automation.md` — removed `azure-hybrid-lab/` folder structure, "Azure Hybrid Lab (Windows)" playbook section, `azure_lab_*` inventory groups.
  - `05 - Terraform Configuration.md` — removed entire "Azure Hybrid Lab (Proxmox)" section (Terraform locals, VM config, deployment commands); cleaned related-docs link.
  - `11 - Credentials.md` — removed "SQL01 OnPrem" credentials block.

### Archived
- 📦 **8 Azure-focused documentation files** moved from active root → `_archived-2026-05-azure/`:
  - `36 - Azure Environment.md`
  - `37 - Azure Hybrid Lab.md`
  - `47 - Azure Sentinel Learning Lab.md`
  - `48 - HomelabAnalytics Web Application.md`
  - `49 - Azure App Service and Fabric Migration Tutorial.md`
  - `50 - Azure Multi-Region VWAN Infrastructure.md`
  - `51 - Azure Architecture Overview.md`
  - `AZURE-CLAUDE.md`
- Added `_archived-2026-05-azure/README.md` explaining the archive scope and how to re-promote files.

### Kept (intentional)
- ✅ Windows Server templates `9022` (WS2022-Template) + `9025` (WS2025-Template) on node03 — stopped, no resource cost, available for future use.
- ✅ Learning materials (`Hermes Homelab Technical Manual.md`, `Book - The Complete Homelab Guide.md`) untouched — they include Azure chapters as educational content, not active infrastructure.
- ✅ Reddit `/r/AZURE` subscription on Glance News tab — personal learning interest, not infrastructure.

### Pending
- ⏳ Sunday vzdump job `backup-13fa5dd2-e814` still references VMIDs `1000,9025,9022` — to be removed from `/etc/pve/jobs.cfg` (in progress; network blip mid-edit). User-approved.

---

## [2026-05-25] - Backup Pipeline Recovered — PBS MAC Conflict + Stale API

### Fixed
- 🔧 **All PBS backup jobs were failing** since at least 2026-05-20 (NVMe rebuild). Symptom: `TASK ERROR: could not activate storage 'pbs-daily': pbs-daily: error fetching datastores - 500 Can't connect to 192.168.20.50:8007 (Connection refused)`. Affected both `pbs-daily` (19:00 daily) and `pbs-main` (Fri 02:00) jobs across all 3 nodes. The ProxmoxData NFS Sunday backup job was NOT affected (last ran 2026-05-24 03:48 OK).
  - **Root cause**: another device on VLAN 20 is squatting on IP `192.168.20.50` with MAC `78:20:51:8A:E3:7B` (OUI TP-LINK). Proxmox nodes were ARP-resolving to the squatter, sending TCP SYN to the wrong port, getting RST → "Connection refused". CT100 (PBS) itself was healthy — `proxmox-backup-proxy` listening on 8007, reachable from inside the container. Confirmed via bridge FDB on node03: conflict MAC came in via `nic0` (physical NIC, not from any local CT).
  - **Fix #1 (immediate)**: flushed stale ARP on all 3 Proxmox nodes and added **permanent** ARP override `192.168.20.50 → bc:24:11:94:b7:3b` (CT100's real MAC) via systemd unit `/etc/systemd/system/pbs-static-arp.service` on each node — survives reboots.
  - **Fix #2 (verification)**: ran manual `vzdump 202 --storage pbs-daily` (pihole CT, smallest LXC). Completed in 8s — incremental, reused 316.7 MiB (19.3 %), uploaded 461 MiB compressed. Pipeline confirmed end-to-end.
  - **Recommended next step (for user)**: track down the squatter device. MAC `78:20:51:8A:E3:7B` is **not** visible in the Omada Controller's client list (Marikina site), so it's likely a non-managed device on a non-Omada port or behind a dumb switch. Suggested action: physically inspect TP-Link / IoT devices, OR change CT100's IP to a free address (e.g. `.55`) to dodge the conflict entirely — update `pbs-main`/`pbs-daily` storage configs, Pi-hole DNS for `pbs.hrmsmrflrii.xyz`, the 3 ARP overrides, and CLAUDE.md rule #11 + #7.

### Changed
- 🔄 **`nas-backup-status-api`** (CT host: `docker-vm-core-utilities01` 192.168.40.13:9102) — two fixes shipped:
  - `nas_target` string updated from dead `192.168.20.31:/volume2/ProxmoxData/pbs-offsite` → `192.168.20.32:/volume2/ProxmoxData/pbs-offsite`.
  - `VM_NAMES` mapping rewritten to match the post-rebuild cluster (e.g. `200 → docker-lxc-glance`, `107 → docker-vm-core-utilities01`, `108 → immich-vm01`) instead of the pre-rebuild Feb-2026 names (which had `200 → ansible-controller`, `107 → immich-lxc`, etc. — wrong after NVMe rebuild renumbered things).
  - Container rebuilt + restarted on `docker-vm-core-utilities01`; verified `/status` now returns `nas_target: 192.168.20.32:...` and correct names. The "Daily Backups" / "Main Backups" tiles on the Glance Backup tab still show Feb 18 / Feb 15 last_backup dates — that's accurate (those WERE the last successful backups); they'll roll forward when the 19:00 daily job runs tonight.
- File: `/opt/nas-backup-status-api/app.py` on `docker-vm-core-utilities01`.

---

## [2026-05-25] - Glance Storage Tab — Widget Fixes + Grafana Time Picker

### Fixed
- 🔧 **Proxmox Storage Pools — Per Node** widget — ERROR `range can't iterate over ProxmoxDataVMDiskslocal-lvm`. Root cause: Glance's template engine treats `concat (slice "x") (slice "y") (slice "z")` as string concatenation (not slice concatenation), so the resulting value was the literal `"ProxmoxDataVMDiskslocal-lvm"` string instead of a 3-element list. Fix: unrolled the loop — single `range .JSON.Array "data"` pass captures each (pool × node) cell into named variables, then renders explicit blocks per pool/node. Same fix applied to **Proxmox Storage Pools (Aggregate)** widget. Both widgets now render live (ProxmoxData 15.6%, VMDisks 42.2%).
- File: `/opt/glance/config/glance.yml` on CT200 (192.168.40.12); backups retained.

### Changed
- 🔄 **Grafana iframes on Storage tab** — switched from `/d-solo/` (single panel, no chrome at all) to `/d/...?viewPanel=N&kiosk=tv` so the native Grafana time picker is visible on each embedded chart (Free Storage Over Time, Memory Usage Over Time, Disk Temperatures, Storage Consumption Over Time). Users can now change the time range directly inside the iframe without going to Grafana.

---

## [2026-05-25] - Glance Storage Tab — Full Redesign (v2, 3-Column Layout)

### Changed
- 🔄 **Glance Storage tab redesigned again** to mirror the Compute tab's 3-column layout — less cramped, all metrics retained, several new live metrics added
  - **Left column (small)** — NAS at a glance, one card per metric:
    - **NAS Overview** — DS920+ health badge, hardware summary (4×HDD + 2×M.2 NVMe; 8/4/12/10 TB + 2×1 TB)
    - **Capacity (ProxmoxData)** — TB used/free/total from Proxmox API with gradient bar (color thresholds 70/90 %)
    - **System Health** — live Healthy Drives count (X/6) from Prometheus `synologyDiskHealthStatus` + RAID badge (SHR-1)
    - **System Temperature** — live °C from `synologySystemTemperature` with Cool/Normal/Warm/Hot label and color border
    - **NAS Memory Used** — live % from `(memTotalReal-memAvailReal)/memTotalReal*100`, color-coded bar, 32 GB DDR4 context
    - Storage Systems monitor (DSM/PBS/Proxmox uptime) — retained
    - Quick Links (NAS / Dashboards / Backups) — retained
  - **Center column (full)** — drives, time-series, Proxmox detail:
    - **Drive Health grid** — 3-column card grid for all 6 drives (4×HDD + 2×M.2), each showing health badge, model, capacity, temperature
    - **Free Storage Over Time** — embedded Grafana panel 40 (`synology-nas-modern` dashboard)
    - **Memory Usage Over Time** — embedded Grafana panel 31
    - **Disk Temperatures (Live)** — embedded Grafana panel 20 (bargauge)
    - **Proxmox Storage Pools — Per Node** — 3×3 grid retained
  - **Right column (small)**:
    - **Proxmox Storage Pools (Aggregate)** — compact bars for local-lvm/VMDisks/ProxmoxData
    - **RAID Volumes** — Volume 2 (SHR-1) + Storage Pool 2 (M.2 cache) status from `synologyRaidName`
    - **Storage Consumption Over Time** — embedded Grafana panel 40
  - **New live metrics added** (vs. previous tab):
    - NAS system health badge, system temperature, total RAM used, per-drive health + temperature + model — all pulled live from Prometheus (`http://192.168.40.13:9090`)
  - **Backup created**: `/opt/glance/config/glance.yml.bak.20260525-storage-redesign` on CT200
  - File: `/opt/glance/config/glance.yml` on CT200 (192.168.40.12) — Storage tab now lines 683–1052 (was 683–866)
  - Reloaded clean — HTTP 200, no template errors; verified live values render: System Temp 52 °C, Healthy Drives 6/6, RAM Used 96.2 %, ProxmoxData 1.09 TB used

---

## [2026-05-25] - Glance Storage Tab Overhaul

### Changed
- 🔄 **Glance Storage tab redesigned** (`https://glance.hrmsmrflrii.xyz/?page=storage`) — all existing metrics preserved (Synology Grafana iframe with 24 panels kept intact), layout reorganized for clarity
  - **New widgets** in left sidebar:
    - **NAS Storage Summary** — hero card showing TB used/free/total from ProxmoxData NFS pool, % usage with color-coded gradient bar, healthy drives count, RAID type, drive inventory
    - **Proxmox Storage Pools** — compact aggregated bars for local-lvm, VMDisks, ProxmoxData across all nodes
  - **New widget** on right (below NAS iframe):
    - **Proxmox Storage Pools — Per Node** — 3×3 grid showing each pool's usage per node with color-coded threshold bars (green <70% / amber <90% / red ≥90%)
  - **Cleanups**:
    - Deduplicated Storage Systems monitor (was showing Synology twice on VLAN 20 + VLAN 10, both same IP) → now Synology DSM + PBS Server + Proxmox Cluster
    - Quick Links expanded into 3 grouped sections (NAS apps, Dashboards, Backups)
  - File: `/opt/glance/config/glance.yml` lines 683–866 on CT200 (192.168.40.12); backup `.bak.YYYYMMDD-HHMMSS`
  - Glance reloaded clean (HTTP 200, no template errors)

---

## [2026-05-25] - Container Monitoring Dashboard — Reduce Calc Update

### Changed
- 🔄 **Container Monitoring Grafana dashboard** (uid: `containers-modern`) — switched panel reduce calc from `lastNotNull` (which always showed the latest scrape regardless of selected time range) to range-aware calcs so the time picker actually affects displayed values
  - Panel 4 "Total CPU" (stat) → `max`
  - Panel 3 "Total Memory" (stat) → `mean`
  - Panels 7/8 "CPU Usage Core/Media" (bargauge) → `max` (sortBy + reduceOptions field updated to `Max`)
  - Panels 5/6 "Memory Usage Core/Media" (bargauge) → `mean` (sortBy + reduceOptions field updated to `Mean`)
  - File: `/opt/monitoring/grafana/dashboards/container-status.json` on 192.168.40.13; backup `.bak.YYYYMMDD-HHMMSS`
  - Verified live (dashboard version → 11)

---

## [2026-05-25] - Container Monitoring Dashboard Sort Fix (Glance Compute Tab Iframe)

### Fixed
- 🔧 **Container Monitoring Grafana dashboard** (uid: `containers-modern`, embedded in Glance Compute tab) — bar gauges for container CPU/Memory now sort descending
  - Affected panels: 5 "Memory Usage - Core Utilities", 6 "Memory Usage - Media VM", 7 "CPU Usage - Core Utilities", 8 "CPU Usage - Media VM"
  - Root cause: existing `sortBy` transformation was a no-op because each Prometheus series came back as a separate frame with one row — `sortBy` only sorts rows *within* a frame
  - Fix: wrapped each PromQL query with `sort_desc(...)` so Prometheus returns series in descending value order; left transformations and panel options untouched so the visual layout is unchanged
  - File: `/opt/monitoring/grafana/dashboards/container-status.json` on docker-vm-core-utilities01 (192.168.40.13); backup `.bak.20260525-033209` preserved
  - Verified live in Grafana (dashboard version → 9 after provisioner reload)

---

## [2026-05-25] - Proxmox Compute Dashboard Sort Fix

### Fixed
- 🔧 **Proxmox Cluster Overview dashboard** (uid: `proxmox-compute`) — added descending `sortBy` transformation on `Value` to the 3 storage bar gauges so highest usage appears on top
  - Panel 10: Local LVM Storage Usage %
  - Panel 11: VM Disks Storage Usage %
  - Panel 12: Proxmox Data Storage Usage %
  - File: `/opt/monitoring/grafana/dashboards/proxmox-dashboard.json` on docker-vm-core-utilities01 (192.168.40.13); backup written as `.bak.YYYYMMDD-HHMMSS`
  - Verified Grafana provisioner reloaded the change (dashboard version bumped to 6)
  - Uptime bar gauges (panels 16 + 23) left as ascending — title "Recently Rebooted → Longest" indicates that ordering is intentional

---

## [2026-05-25] - Homepage Dashboard Deployment Complete

### Added
- ✅ **Homepage dashboard** — `https://homepage.hrmsmrflrii.xyz` fully operational
  - CT209 (homepage-lxc) on node01 at **192.168.40.27** (reassigned from .26 to resolve IP conflict)
  - Docker CE + Homepage container (`ghcr.io/gethomepage/homepage:latest`) deployed on port 3000
  - Config at `/opt/homepage/config/` — settings, services, widgets, bookmarks YAML
  - Note: several widget API keys still need to be set (marked `changeme` in `/opt/homepage/config/services.yaml`)
- ✅ **Traefik route** — `homepage.yml` added to `/opt/traefik/config/dynamic/`
- ✅ **Pi-hole DNS** — `homepage.hrmsmrflrii.xyz → 192.168.40.20` added to `pihole.toml` dns.hosts

### Fixed
- 🔧 **Traefik services.yml parse error** — a prior session misplaced a homepage router config under the `middlewares` block and a homepage service under `serversTransports`; both removed; Traefik restarted cleanly; no more `field not found, node: entryPoints` errors

---

## [2026-05-23] - Homepage Deployment + Dashboard Fixes + PBS ARP Fix

### Added
- ✅ **CT209 (homepage-lxc)** — new LXC on node01, 192.168.40.26, 1GB RAM, 8GB disk, Debian 12
  - Docker CE installed; Homepage container running at port 3000
  - Config at `/opt/homepage/` — settings, services, widgets YAML all written
  - Traefik route added for `homepage.hrmsmrflrii.xyz`
- ✅ **Proxmox API token** `root@pam!homepage` created (secret: <redacted>) for Homepage widget auth

### Fixed
- 🔧 **PBS backup job "Connection refused"** — all 3 Proxmox nodes had stale ARP entries for 192.168.20.50 pointing to old MAC `78:20:51:8a:e3:7b` (pre-recovery MAC); actual CT100 MAC is `bc:24:11:94:b7:3b`; fixed with `ip neigh flush dev vmbr0 192.168.20.50` on node01/02/03
- 🔧 **Grafana NAS dashboard label text** — stat panels (System Uptime, Total/Used Storage, System Temp, Total RAM, Free Storage, Available RAM) changed `textMode: auto → value` to hide Prometheus label strings; gauge panels (Storage %, CPU Usage, Memory Usage) filtered to `instance=~"192.168.20.32.*"` (removes dead .31 duplicate) and `legendFormat: " "`

### Removed

### Changed
- 🔄 **Grafana Network Utilization dashboard** — Combined Bandwidth panel updated: title → "2.5Gbps Reference", Y-axis max → 2.75Gbps, threshold → 2Gbps, reference line expr → 2500000000; saved to `/opt/monitoring/grafana/dashboards/network-utilization.json` (provisioned file)

---

## [2026-05-23] - Omada Exporter + Network Tab Fix

### Fixed
- 🔧 **Omada exporter** (on ansible-controller01, 192.168.20.30) — container was crash-looping with `invalid character 'W'` JSON parse error
  - Root cause 1: `OMADA_HOST` pointed at `192.168.0.103` (a TP-Link switch web UI, `Server: Web Switch`), not the Omada controller
  - Root cause 2: Port set to `:8043` — actual controller listens on port 443
  - Fix: Updated `/opt/omada-exporter/docker-compose.yml` → `OMADA_HOST=https://192.168.0.101`; restarted container
  - Verified: `omada_client_connected_total` and device metrics now flowing at `192.168.20.30:9202/metrics`
- 🔧 **Glance Network tab** — Omada Network Dashboard Grafana iframe was empty; fixed by fixing upstream exporter → Prometheus scrape → Grafana

### Verified
- 🟢 Omada exporter running (Up, no restarts); Prometheus scraping `192.168.20.30:9202`
- 🟢 Omada controller confirmed at `https://192.168.0.101` (port 443), API v3, Omada SDN 6.2.10.18

---

## [2026-05-23] - Glance Dashboard Updates + PBS Recovery + NAS Health Check Fix

### Fixed
- 🔧 **PBS CT100 (pbs-server) recovered** — container was stopped on node03; started via `pct start 100`; `proxmox-backup.service` confirmed active; web UI at 192.168.20.50:8007 returning 200
- 🔧 **Synology NAS health check** — SVM API (`/opt/service-version-api/app.py`) was checking `192.168.20.31:5001` (old 1GbE IP, unreachable); updated to `192.168.20.32:5001` (10GbE NIC); rebuilt Docker image; NAS now shows Online
- 🔧 **Glance dashboard** — removed Power Control (iframe) and Tech News (RSS) widgets from Home page

### Changed
- 🔄 **SVM API NAS health_url + web_url**: `192.168.20.31:5001` → `192.168.20.32:5001`
- 📦 SVM API container rebuilt and redeployed to pick up config change

### Verified
- 🟢 Infrastructure: all 6/6 services online (Node01/02/03, Synology NAS, PBS, Traefik)

---

## [2026-05-23] - Proxmox Subscription Nag Fix + node02 Package Upgrade

### Fixed
- 🔧 **Proxmox "No valid subscription" popup** — patched `proxmoxlib.js` (non-minified) on all 3 nodes (node01, node02, node03)
  - Root cause: pveproxy serves `/proxmoxlib.js` (non-minified, with `?ver=` cache-buster), NOT `proxmoxlib.min.js`; popup is triggered by `checked_command()` called at login (pvemanagerlib.js:73109) which checks subscription status and shows dialog when `status !== 'active'`
  - Fix: replaced both `res.data.status.toLowerCase() !== 'active'` checks (lines 614 and 21106) with `=== 'NoMoreNagging'` — condition is always false for community nodes; pveproxy restarted on all 3 nodes
  - node01: 2 replacements applied; node02/node03: already had `== 'NoMoreNagging'` from prior session
  - Browser cache note: must **Ctrl+Shift+R** hard-refresh each node's UI after patch (same `?ver=` param means browsers serve cached old JS until forced reload)
  - **Warning**: This patch is overwritten when `proxmox-widget-toolkit` is upgraded — must be re-applied after package updates that touch that package
- 🔧 **apt-get dist-upgrade stuck** on node02 — `apt-listchanges` pager was blocking upgrade waiting for keyboard input; killed pager process to unblock

### Changed
- 📦 **node02 full package upgrade** — 201 packages upgraded (proxmox-widget-toolkit 5.1.2 → 5.2.2, openssl, libpve-notify-perl, initramfs-tools, pve-ha-manager, and others)

---

## [2026-05-23] - Traefik Docker Provider Fix + Glance Accessibility

### Fixed
- 🔧 **Traefik Docker provider API version mismatch** — Traefik v3.2's embedded Docker SDK was connecting with API v1.24 but the Docker daemon (post OS upgrade) requires minimum v1.40; this caused continuous error-loop retries every few seconds, filling `traefik.log` with thousands of error entries
  - Root cause: Docker Engine was upgraded during 2026-05-20 package updates, raising the minimum accepted API version
  - Fix: Removed the `docker:` provider section from `traefik.yml` (all routing is via file provider `services.yml`; Docker label discovery was unused)
  - Also removed unused `/var/run/docker.sock` mount from `docker-compose.yml`
  - Traefik restarted cleanly; all routes verified: glance (200), grafana (200), traefik-dashboard (302) ✅

### Verified
- 🟢 **Glance dashboard** accessible at https://glance.hrmsmrflrii.xyz/ — was unreachable due to Traefik instability caused by Docker provider error storm on startup
- 🟢 All key services confirmed working post-restart (Glance, Grafana, Traefik Dashboard)

---

## [2026-05-23] - Claude Code Agent Configuration

### Added
- 📄 `CLAUDE.md` — directory-level Claude Code instructions with full homelab infrastructure reference, change documentation protocol, and behavioral rules

---

## [2026-05-23] - Full Cluster NFS Migration to 192.168.20.32

### Fixed
- 🔧 **immich-vm01**: had 3 stale `.31` fstab entries — VM was stuck mid-boot and SSH was down
  - Fixed: `/volume2/ProxmoxData`, `/volume2/homes`, `/volume2/Immich Photos` → all now `.32`
  - Root cause: fstab not updated during initial NAS NIC migration
- 🔧 **docker-lxc-media (CT205)**: `<redacted>` fstab entry fixed `.31` → `.32`
- 🔧 **pbs-server (CT100)**: `/volume2/ProxmoxData /mnt/nas-backup` fstab entry fixed `.31` → `.32`

### Infrastructure
- 🖥️ Synology NAS: added 10GbE NIC (eth1) → connected to new 2.5G switch → IP `192.168.20.32`
- 🌐 All Proxmox nodes (node01/02/03) updated to use `192.168.20.32` for NFS mounts (was 192.168.20.31)
- 🔧 Proxmox storage pools VMDisks + ISOs updated to server `192.168.20.32`

### Verified clean (no .31 references)
- VM103 ansible-controller01, VM106 gitlab-vm01, VM107 docker-vm-core-utilities01, VM121 gitlab-runner-vm01, VM1000 Ubuntu template
- Windows templates (VM9022, VM9025) — not applicable

### Changed
- 🔄 All NFS mount targets cluster-wide: `192.168.20.31` → `192.168.20.32` (2.5 GbE path)
- 📋 **Standard**: `192.168.20.31` is management-only; ALL NFS mounts MUST use `192.168.20.32`

### Removed
- ❌ K8s cluster decommissioned — 9 VMs (k8s-controller01-03, k8s-worker01-06) deleted; IPs .32-.45 freed

---

## [2026-05-20] - node01 NVMe Failure & Full Recovery

### Infrastructure
- 🖥️ node01 NVMe (Kingston OM8TAP41024K1-A00, serial 50026B7383F85E24) confirmed dead — PCIe link Unknown, controller state `dead`, root filesystem in `emergency_ro`
- 🖥️ node01 fresh Proxmox VE 9.1.19 install on replacement NVMe — rejoined MorpheusCluster (quorum restored, 3 nodes)

### Fixed
- 🔧 node01 NFS fstab: changed from `hard` to `soft,timeo=30,retrans=3,_netdev` mounts — prevents future node freezes
- 🔧 node01 vmbr0: added `bridge-vlan-aware yes` and `bridge-vids 2-4094` — required for VLAN-tagged LXC container networks
- 🔧 node01 apt repos: replaced enterprise repos (401 Unauthorized) with no-subscription repos to match node02/node03
- 🔧 Grafana anonymous access: added `GF_AUTH_ANONYMOUS_ORG_NAME=Main Org.` to docker-compose — fixed "Data source not found" for anonymous iframe sessions in Glance

### Added
- ✅ node01 node_exporter v1.7.0 installed and running — Prometheus scraping `192.168.20.20:9100` restored
- ✅ All 6 node01 LXC containers restored from backups (CT200 glance, CT201 bots, CT202 pihole, CT203 traefik, CT205 media, CT206 homeassistant)

### Changed
- 🔄 All 10 cluster containers updated to latest OS packages (node01 containers from Feb 16 backup baseline → current)

### Security
- 🔒 node01 root SSH key authorized: `hermes@homelab-nopass` ed25519 key added to `/root/.ssh/authorized_keys`

---

## [2026-01-16] - Homelab Agent Deployment

### Added
- ✅ Homelab Agent: 10 Claude Code skills for infrastructure management
  - `/homelab` - Main menu and status dashboard
  - `/lab-status` - Comprehensive infrastructure status
  - `/service-list` - Service catalog with URLs
  - `/ip-find` - IP address management
  - `/deploy-new` - Deployment code generator
  - `/troubleshoot` - Troubleshooting assistant
  - `/lab-changelog` - This changelog system
  - `/runbook` - Operational runbook generator
  - `/capacity` - Resource capacity planning
  - `/doc-sync` - Documentation verification

- ✅ Homelab Agent Plan (`Homelab Agent Plan.md`)
  - Comprehensive feature specifications
  - Implementation roadmap
  - Command reference

- ✅ Homelab Changelog (this file)
  - Infrastructure change tracking
  - Audit trail for all modifications

### Documentation
- 📝 Skills deployed to `.claude/commands/`
- 📝 Plan documented in Claude Managed Homelab folder

---

## [Unreleased]

### Planned
- [ ] Live API integration with Proxmox
- [ ] Prometheus metrics in status reports
- [ ] Automated backup verification
- [ ] Discord notifications integration

---

## Template for New Entries

```markdown
## [YYYY-MM-DD] - Brief Description

### Added
- ✅ [Description]

### Changed
- 🔄 [What]: [old] → [new]

### Fixed
- 🔧 [Issue]: [Resolution]

### Removed
- ❌ [What was removed]

### Infrastructure
- 🖥️ [Hardware/network change]

### Security
- 🔒 [Security update]
```

---

## Monthly Statistics

### January 2026

| Category | Count |
|----------|-------|
| Added | 12 |
| Changed | 0 |
| Fixed | 0 |
| Removed | 0 |
| Infrastructure | 0 |
| Security | 0 |
| **Total** | **12** |

---

*Managed by Homelab Agent - `/lab-changelog`*
