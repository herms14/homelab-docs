---
created: 2026-10-15
updated: 2026-10-16
type: project
status: seed
project-status: done
tags: [rfc, change-management, medic]
rfc-status: "Implemented"
approval: 44
---
# RFC-2026-008 · Fix incident #40: Master Hermes — docker-lxc-media startup verification race

| | |
|---|---|
| **Status** | Implemented |
| **Performed by** | ✨ Corhyn (ITOps Lead · Infrastructure Operations & SRE) |
| **Governance** | Ranni |
| **Approver** | Master Hermes (✅ on Roundtable approval #44) |
| **Filed** | 2026-10-16 04:13 |

## What will be done

1. Do not restart or repair the media services or packages; they are currently healthy.
2. In an approved maintenance change, back up `/home/hermes-admin/sre_agent.py` before editing it.
3. Give post-Docker-restart verification its own 180-second deadline, poll every 10 seconds, treat `starting` as pending, and require every container with a health check to become healthy before reporting success. Continue requiring containers without health checks to be running and not restarting.
4. If the deadline expires, collect health history for affected containers and the last 200 lines of their application logs, then report the specific containers and evidence.
5. Check syntax, activate the updated verifier, and confirm the service is active. Verify the behavior on the next authorized patch run. If the change fails, restore the backup and restart the verifier service.

Commands:
```bash
Already run, read-only: ssh docker-lxc-media 'sudo -n docker ps --format "{{.Names}}\t{{.Status}}"'
Proposed after approval: cp -a /home/hermes-admin/sre_agent.py /home/hermes-admin/sre_agent.py.bak-incident-341-20261015
Proposed after approval: sudoedit /home/hermes-admin/sre_agent.py
Proposed after approval: python3 -m py_compile /home/hermes-admin/sre_agent.py
Proposed after approval: sudo systemctl restart sre-agent.service
Proposed after approval: sudo systemctl is-active sre-agent.service
On a future verification deadline: ssh docker-lxc-media 'sudo -n docker inspect jellyfin cadvisor --format "{{.Name}} {{json .State.Health}}"; sudo -n docker logs --tail 200 jellyfin; sudo -n docker logs --tail 200 cadvisor'
Rollback after approval: cp -a /home/hermes-admin/sre_agent.py.bak-incident-341-20261015 /home/hermes-admin/sre_agent.py
Rollback after approval: sudo systemctl restart sre-agent.service
```

## What will change

Jellyfin was briefly unavailable during initialization, meeting the user-facing app impact criterion. The earlier triage reports recovery about 41 seconds after startup. At the current check, all 16 containers are running and every container with a health check is healthy; there is no ongoing application outage or sign of data risk.

Host: `docker-lxc-media` · expected downtime: None for media services. The verifier watcher may be briefly unavailable during its service restart. · severity high.

## Recovery plan

Restore `/home/hermes-admin/sre_agent.py` from the incident backup and restart `sre-agent.service`. This rolls back verifier behavior without changing media containers, packages, or volumes.

Verification:
- Current read-only check: all 16 containers are running; Jellyfin, cAdvisor, and MeTube report healthy.
- After the verifier change, confirm `sre-agent.service` is active.
- On the next authorized run, confirm `starting` is tolerated during polling and success is reported only after all health-checked containers are healthy.
- If the 180-second deadline expires, confirm the report includes Docker health history and application log excerpts for the affected containers.

## Risk

No service change is needed on docker-lxc-media. The verifier change could delay a genuine failure report by up to 180 seconds; requiring all health-checked containers to become healthy and collecting diagnostics at expiry limits that risk. Restarting the verifier service may briefly pause its watcher, but should not interrupt media containers.

## Closure

**Implemented** — closed by Ranni on 2026-10-16 05:19.


Register: [RFC Register](RFC%20Register.md)
