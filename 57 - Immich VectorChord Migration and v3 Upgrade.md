---
tags: [type/note, status/evergreen, domain/technical/homelab, source/claude]
created: 2026-08-26
updated: 2026-10-10
type: note
status: evergreen
---

# Immich VectorChord Migration + v3 Upgrade

> Step-by-step record of migrating Immich's Postgres extension from the deprecated `pgvecto.rs` to VectorChord, and updating Immich v2.7.5 → v3.1.0 on `immich-vm01`. Performed 2026-08-26. Closes [TODO - Homelab Improvements 2026-05](TODO%20-%20Homelab%20Improvements%202026-05.md) item #3.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [42 - Immich Backup and Restore](42%20-%20Immich%20Backup%20and%20Restore.md) | [07 - Deployed Services](07%20-%20Deployed%20Services.md) | 11 - Credentials

---

## Why

Immich dropped support for the `pgvecto.rs` Postgres extension (`vectors`, v0.2.0) in favor of VectorChord. The old image (`tensorchord/pgvecto-rs:pg14-v0.2.0`) would break on the next Immich update if left in place. This tutorial covers doing the migration and the version update together, since Immich handles the extension swap automatically as part of its own startup migration.

## Prerequisites

- SSH access to the target VM as a user with Docker + sudo (see [16 - SSH Configuration](16%20-%20SSH%20Configuration.md))
- Enough free disk space for a full `pg_dumpall` backup (roughly the size of your Postgres data — was ~211MB compressed for this library)
- Know your current Postgres image tag (`docker compose ps` or `grep image: docker-compose.yml`)

## Steps

### 1. Check current state

```bash
ssh immich-vm01
sudo docker compose -f /opt/immich/docker-compose.yml ps
sudo docker logs immich-server --tail 5   # look for "Immich Server is listening on ... [vX.Y.Z]"
sudo docker exec immich-postgres psql -U postgres -d immich -c "SELECT extname, extversion FROM pg_extension;"
```

Look for an extension named `vectors` — that's the old pgvecto.rs extension that needs migrating.

### 2. Take a fresh backup — don't trust old ones

Check for an existing backup script/schedule first; **verify it's actually current**, don't assume the docs are right:

```bash
ls -lh /mnt/immich-uploads/db-backups/   # or wherever your compose file points backups
```

If the most recent backup is more than a few days old, take a new one before doing anything else:

```bash
sudo docker exec immich-postgres pg_dumpall -U postgres | gzip > /mnt/immich-uploads/db-backups/pre-v3-upgrade-$(date +%Y%m%d).sql.gz
ls -lh /mnt/immich-uploads/db-backups/
```

Confirm the backup destination is durable (NFS/NAS), not just the VM's local disk:

```bash
df -h /mnt/immich-uploads
mount | grep immich-uploads
```

### 3. Verify the target Postgres image before touching production

Don't guess the VectorChord image tag — it's versioned and changes over time. Check Immich's official upgrade docs (`https://docs.immich.app/install/upgrading`), then test-pull the image before editing anything live:

```bash
sudo docker pull ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
```

(This tag was current as of 2026-08-26 — check the docs again if you're doing this later, the version numbers move.)

### 4. Edit docker-compose.yml

Back up the compose file first, then swap the postgres image:

```bash
sudo cp /opt/immich/docker-compose.yml /opt/immich/docker-compose.yml.bak-$(date +%Y%m%d)
sudo sed -i 's#docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0#ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0#' /opt/immich/docker-compose.yml
grep -n 'image:' /opt/immich/docker-compose.yml   # sanity check
```

`immich-server` and `immich-machine-learning` were already tracking the `:release` tag, so no edit needed there — a plain pull picks up the latest release.

### 5. Pull and recreate

```bash
cd /opt/immich
sudo docker compose pull
sudo docker compose up -d
```

### 6. Watch the automatic migration

**No manual SQL needed** — Immich detects the extension change and migrates on startup:

```bash
sudo docker logs -f immich-server
```

Expect to see:
```
Initializing Immich vX.Y.Z
Creating VectorChord extension
targetLists=1, current=1 for clip_index of N rows
Reindexing clip_index (This may take a while, do not restart)
targetLists=1, current=1 for face_index of N rows
Reindexing face_index (This may take a while, do not restart)
```

For large libraries (100k+ assets) this can take minutes — don't restart the container while it's reindexing. For reference, ~30k photos + ~39k faces took under a minute on this cluster.

### 7. Verify

```bash
curl -s http://localhost:2283/api/server/ping        # expect {"res":"pong"}
curl -s http://localhost:2283/api/server/version      # confirm new version
sudo docker compose -f /opt/immich/docker-compose.yml ps
sudo docker logs immich-server --since 5m | grep -iE 'error|fail|exception'   # should be empty
```

Then check the web UI (`https://photos.hrmsmrflrii.xyz`) loads, and that search/faces still work — the reindex is what backs both features.

## Gotchas encountered

- **The documented backup automation didn't actually exist.** `42 - Immich Backup and Restore.md` references `/opt/immich/backup-db.sh` — this file was not present on `immich-vm01` when checked. The only backup on disk was 6+ months old. Always verify backup freshness directly rather than trusting docs.
- **`immich-server` shows `health: starting` / `unhealthy` in `docker compose ps` even when the app is fine.** The container's built-in healthcheck (`wget --spider`) can lag or fail for reasons unrelated to actual app health. Trust `curl .../api/server/ping` over the Docker health status.
- **`qm agent <vmid> ping` reported "not running"** even though `agent: 1` is set in the VM's Proxmox config — the QEMU guest agent was never actually installed inside the guest. Doesn't block this migration (done entirely over SSH), but means `qm guest exec` isn't available as a management path for this VM. Worth fixing separately.

## Follow-up

- [ ] Consider installing `qemu-guest-agent` on immich-vm01 (and the other VMs) so Proxmox-level management (`qm guest exec`) works without SSH.

## Later update: v3.2.4 (2026-10-03)

Routine point-release bump, v3.2.2(ish) → v3.2.4, well after the VectorChord migration above. No Postgres extension change, no `docker-compose.yml` edit — `immich-server` and `immich-machine-learning` both still track the `:release` tag, so this was a plain pull:

```bash
sudo docker exec immich-postgres pg_dumpall -U postgres | gzip > /mnt/immich-uploads/db-backups/pre-v3.2.4-upgrade-$(date +%Y%m%d).sql.gz
cd /opt/immich
sudo docker compose pull
sudo docker compose up -d
```

Verified via `curl -s http://localhost:2283/api/server/version` → `{"major":3,"minor":2,"patch":4,"prerelease":null}`. All 4 containers healthy post-upgrade (`immich-server` briefly showed `health: starting` — known false-flag, `ping`/`version` both responded fine).

Checked release notes for v3.2.0 (first minor bump since v3.1.0) and v3.2.2→v3.2.4: only user-facing change in range was an opt-in "cluster groups" cross-user facial recognition feature (v3.2.0) — not used here, no action needed.

## Later update: v3.3.0 + failed-rollback recovery (2026-10-10)

The SRE agent's automated v3.2.4 → v3.3.0 update ([GitLab issue #24](https://gitlab.hrmsmrflrii.xyz/homelab/agentic-patch-management/-/issues/24)) failed its health check — the compose healthcheck used `wget`, which the image doesn't ship, so it always reported unhealthy. The agent then rolled the image back to v3.2.4, but v3.3.0 had **already migrated the DB**, so v3.2.4 crash-looped with:

```
Migration "<redacted>" was already applied to this database
but is not in this version of Immich (3.2.4). ... Downgrades are not supported.
```

> [!WARNING]
> Immich migrations are forward-only. Never roll back only the image after a newer version has started — either roll forward, or restore the pre-update DB dump together with the old image.

Fix (roll forward):

```bash
sudo docker exec immich-postgres pg_dumpall -U postgres | gzip > /tmp/pre-fix.sql.gz
sudo mv /tmp/pre-fix.sql.gz /mnt/immich-uploads/db-backups/pre-v3.3.0-fix-20261010.sql.gz
cd /opt/immich && sudo cp docker-compose.yml docker-compose.yml.bak-20261010
# server + ML: :release -> :v3.3.0; delete the wget healthcheck block under immich-server
sudo docker compose up -d
curl -s http://localhost:2283/api/server/version   # {"major":3,"minor":3,"patch":0,...}
```


---

## Related Documentation

- [42 - Immich Backup and Restore](42%20-%20Immich%20Backup%20and%20Restore.md) — original backup/restore doc (needs a refresh given the gap found above)
- [16 - SSH Configuration](16%20-%20SSH%20Configuration.md) — how SSH access to immich-vm01 was recovered this session
- [07 - Deployed Services](07%20-%20Deployed%20Services.md) — Immich service entry
- [Homelab Changelog](Homelab%20Changelog.md) — full changelog entry for this migration
