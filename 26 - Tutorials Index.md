---
updated: 2026-10-11
---

# Homelab Tutorials Index

> Comprehensive step-by-step tutorials for homelab tasks and configurations.

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | 25 - Homelab Master Wiki | [Book - The Complete Homelab Guide](Book%20-%20The%20Complete%20Homelab%20Guide.md)

---

## Available Tutorials

### Avatar-Guided Homelab Learning

| Tutorial | Description | Difficulty |
|----------|-------------|------------|
| Roundtable teaching terminal | Signed-in HTTPS → Voice → Learn in a real terminal. Hewg teaches Linux/Proxmox/storage, Torrent networking, Radahn automation; explained preview, actual bounded SSH and learner Run/Pause/Next | Beginner–Intermediate |

Works as a Mac/Windows browser pop-out, not local Terminal/PowerShell control. Keep the parent portal
open and allow pop-ups; Talk is a separate paid voice call. Writes need owner Discord approval; Hewg's
visual configuration is limited to a new fixed learning directory, not unrestricted homelab changes.
[Recovery and limits](12%20-%20Troubleshooting.md#roundtable-portal-teaching-and-discord).

### Discord & Automation

| Tutorial | Description | Difficulty |
|----------|-------------|------------|
| 27 - Discord Bot Deployment Tutorial | Complete guide to creating, deploying, and managing Discord bots in Docker containers | Advanced |
| 28 - Athena Bot Tutorial | Setting up Athena task queue bot with Claude integration | Intermediate |

### Authentication & Security

| Tutorial | Description | Difficulty |
|----------|-------------|------------|
| [29 - Immich Authentik SSO Tutorial](29%20-%20Immich%20Authentik%20SSO%20Tutorial.md) | Integrating Immich with Authentik for OAuth/SSO authentication | Intermediate |
| [14 - Authentik Google SSO Setup](14%20-%20Authentik%20Google%20SSO%20Setup.md) | Configuring Google OAuth for Authentik services | Intermediate |

### Infrastructure

| Tutorial | Description | Difficulty |
|----------|-------------|------------|
| [30 - LXC Migration Tutorial](30%20-%20LXC%20Migration%20Tutorial.md) | Migrating Docker services from VM to LXC containers | Intermediate |
| 31 - Kubernetes Glance Tutorial | Deploying Glance dashboard on Kubernetes | Advanced |
| [39 - PBS Deployment Tutorial](39%20-%20PBS%20Deployment%20Tutorial.md) | Complete PBS deployment with dual-datastore architecture | Advanced |
| [40 - PBS Disaster Recovery](40%20-%20PBS%20Disaster%20Recovery.md) | Full system recovery when both PBS drives fail | Advanced |
| [42 - Immich Backup and Restore](42%20-%20Immich%20Backup%20and%20Restore.md) | Immich database backup and disaster recovery procedures | Intermediate |
| [57 - Immich VectorChord Migration and v3 Upgrade](57%20-%20Immich%20VectorChord%20Migration%20and%20v3%20Upgrade.md) | Migrating Immich's Postgres from pgvecto.rs to VectorChord and updating to v3 | Intermediate |

### Kubernetes & Cloud

| Tutorial | Description | Difficulty |
|----------|-------------|------------|
| 46 - Kubernetes CKA CKS Learning Lab | **Comprehensive guide**: CKA/CKS certification prep with homelab migration scenarios, hands-on labs, and AKS migration path | Advanced |

### CI/CD & GitOps

| Tutorial | Description | Difficulty |
|----------|-------------|------------|
| [43 - GitOps Complete Tutorial](43%20-%20GitOps%20Complete%20Tutorial.md) | **Comprehensive guide**: GitOps theory, design patterns, and full implementation with GitLab CI/CD | Advanced |
| [44 - Glance GitOps Onboarding Tutorial](44%20-%20Glance%20GitOps%20Onboarding%20Tutorial.md) | **Hands-on guide**: Step-by-step migration of Glance to GitOps with GitLab CI/CD | Intermediate |
| [20 - GitLab CI-CD Automation](20%20-%20GitLab%20CI-CD%20Automation.md) | GitLab CI/CD pipeline configuration and automation | Intermediate |

### Developer Tooling

| Tutorial | Description | Difficulty |
|----------|-------------|------------|
| [56 - Claude Code Statusline Setup](56%20-%20Claude%20Code%20Statusline%20Setup.md) | Context window progress bar + Oh My Posh in Claude Code terminal | Beginner |

### Service Configuration

| Tutorial | Description | Difficulty |
|----------|-------------|------------|
| [13 - Service Configuration Guide](13%20-%20Service%20Configuration%20Guide.md) | Detailed service setup with command explanations | Beginner |
| [15 - New Service Onboarding Guide](15%20-%20New%20Service%20Onboarding%20Guide.md) | Complete workflow for adding new services | Intermediate |
| [21 - Application Configurations](21%20-%20Application%20Configurations.md) | Detailed app setup and configuration | Intermediate |

---

## Tutorial Difficulty Levels

| Level | Description |
|-------|-------------|
| **Beginner** | Basic Linux/Docker knowledge required |
| **Intermediate** | Familiarity with Proxmox, networking, and containerization |
| **Advanced** | Deep understanding of infrastructure, Kubernetes, and automation |

---

## External Documentation

Tutorials are also available in:
- **docs/** folder in the GitHub repository
- **GitHub Wiki**: https://github.com/herms14/Proxmox-TerraformDeployments/wiki

---

*Last updated: 2026-10-11*
