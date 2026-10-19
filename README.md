# homelab-docs

Documentation for my homelab, **MorpheusCluster**: a 3-node Proxmox VE cluster with Omada SDN (8 VLANs), Traefik + Authentik, Technitium DNS, a Synology NAS, Proxmox Backup Server, OpenBao and an AI agent team (the Roundtable).

This is a **public, redacted copy** of the notes I keep in my Obsidian vault. Start at [00 - Homelab Index](00%20-%20Homelab%20Index.md), or read the long-form [Book - The Complete Homelab Guide](Book%20-%20The%20Complete%20Homelab%20Guide.md).

## What was removed

- All passwords, API keys, tokens, Wi-Fi keys, key fingerprints, OAuth client ids, e-mail addresses and Discord ids. They appear as `<redacted>`, `<email>` or `<id>`. Real secrets live in OpenBao, never in these notes.
- The credentials note, the agent task registry, the agent call transcripts and the interactive Lab Atlas diagrams.
- Everything about my personal and specialized assistants (finance, health, study and so on) and the projects behind them. Some notes therefore skip numbers.
- Links to notes that are not in this copy are left as plain text.

Commands, IPs and hostnames are for my private network (`192.168.x.x`, internal DNS names) and are kept as reference.

## Related repos

| Repo | What |
|---|---|
| [homelab-hub](https://github.com/herms14/homelab-hub) | Hub workspace that pulls all homelab repos together |
| [roundtable](https://github.com/herms14/roundtable) | The Roundtable homelab agent team and its portal |
| [vlan-flow](https://github.com/herms14/vlan-flow) | Interactive VLAN traffic-flow diagram |
| [glance-dashboard](https://github.com/herms14/glance-dashboard) | Glance dashboard config and widget APIs |
| [Clustered Thoughts](https://herms14.github.io/Clustered-Thoughts/) | The homelab blog |

## Format

The notes were written in Obsidian. Internal links were converted to normal relative Markdown links, so they work on GitHub. Some Obsidian-only syntax (callouts, Dataview blocks) shows up as plain text.
