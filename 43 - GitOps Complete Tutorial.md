# GitOps Complete Tutorial: From Theory to Implementation

> A comprehensive guide to understanding and implementing GitOps in a homelab environment. This tutorial teaches GitOps from first principles, covering the philosophy, design patterns, and hands-on implementation using GitLab CI/CD.

Related: [26 - Tutorials Index](26%20-%20Tutorials%20Index.md) | [20 - GitLab CI-CD Automation](20%20-%20GitLab%20CI-CD%20Automation.md) | [06 - Ansible Automation](06%20-%20Ansible%20Automation.md) | [09 - Traefik Reverse Proxy](09%20-%20Traefik%20Reverse%20Proxy.md)

**Difficulty**: Advanced
**Time to Complete**: 4-6 hours (reading + implementation)
**Prerequisites**: Basic Git, Docker, Linux command line

---

## Table of Contents

1. [Chapter 1: What is GitOps?](#chapter-1-what-is-gitops)
2. [Chapter 2: The Philosophy Behind GitOps](#chapter-2-the-philosophy-behind-gitops)
3. [Chapter 3: Core Principles](#chapter-3-core-principles)
4. [Chapter 4: Design Patterns](#chapter-4-design-patterns)
5. [Chapter 5: Architecture for Homelab](#chapter-5-architecture-for-homelab)
6. [Chapter 6: The GitOps Workflow](#chapter-6-the-gitops-workflow)
7. [Chapter 7: Implementation Guide](#chapter-7-implementation-guide)
8. [Chapter 8: Creating Your First GitOps Service](#chapter-8-creating-your-first-gitops-service)
9. [Chapter 9: Advanced Topics](#chapter-9-advanced-topics)
10. [Chapter 10: Troubleshooting](#chapter-10-troubleshooting)

---

# Part I: Understanding GitOps

## Chapter 1: What is GitOps?

### The Definition

**GitOps** is an operational framework that applies DevOps best practices—specifically version control, collaboration, compliance, and CI/CD—to infrastructure automation. The core idea is simple but powerful:

> **Git is the single source of truth for your infrastructure.**

In a GitOps model:
- All infrastructure is defined **declaratively** (as code)
- All changes are made through **Git commits**
- The actual state is **automatically reconciled** to match the desired state in Git

### Traditional Operations vs GitOps

```
TRADITIONAL OPERATIONS
======================

Developer → SSH to server → Run commands → Hope it works → Document somewhere

Problems:
• No audit trail of what changed
• Configuration drift over time
• "It works on my machine" syndrome
• Snowflake servers (each one unique)
• Manual rollback is painful


GITOPS OPERATIONS
=================

Developer → Git commit → Pipeline auto-deploys → System matches Git state

Benefits:
• Complete audit trail (git log)
• Reproducible infrastructure
• Self-documenting changes
• Cattle, not pets (disposable servers)
• Rollback = git revert
```

### Real-World Analogy

Think of GitOps like a **blueprint for a house**:

| Traditional Ops | GitOps |
|-----------------|--------|
| Building a house by talking to workers | Having detailed blueprints |
| Each room might be slightly different | Every room matches the plan exactly |
| Changes happen ad-hoc | Changes update the blueprint first |
| Hard to rebuild if destroyed | Can rebuild identically from plans |
| "Ask Bob, he knows how the plumbing works" | Everything documented in blueprints |

### The GitOps Equation

```
GitOps = Infrastructure as Code + Git + Automation
```

- **Infrastructure as Code (IaC)**: Define everything in text files (YAML, HCL, JSON)
- **Git**: Version control those files, track changes, enable collaboration
- **Automation**: Automatically apply changes when Git state changes

---

## Chapter 2: The Philosophy Behind GitOps

### Origin Story

GitOps was coined by **Weaveworks** in 2017, originally for Kubernetes deployments. However, the principles apply to any infrastructure:

> "We believe that operating in a GitOps way is not just for the cloud natives. Traditional companies operating more stateful systems can benefit equally from this approach."
> — Alexis Richardson, CEO of Weaveworks

### The Three Pillars

#### Pillar 1: Declarative Configuration

**Declarative** means describing **what** you want, not **how** to achieve it.

```yaml
# DECLARATIVE (GitOps way)
# "I want a Grafana container running on port 3030"
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3030:3000"

# IMPERATIVE (traditional way)
# "Step 1: SSH to server. Step 2: Run docker pull. Step 3: Run docker run..."
```

The system figures out how to achieve the desired state. If Grafana is already running, nothing happens. If it's missing, it gets created.

#### Pillar 2: Version Controlled & Immutable

Every change is:
- **Recorded** in Git history
- **Attributed** to a user
- **Timestamped**
- **Reversible** via git revert

```bash
# Who changed what and when?
git log --oneline

a1b2c3d (HEAD -> main) Update Grafana to v10.2.3
e4f5g6h Add Prometheus datasource
i7j8k9l Initial Grafana deployment

# What exactly changed?
git show a1b2c3d

# Who is responsible?
git blame docker-compose.yml
```

#### Pillar 3: Automatically Applied

Changes in Git **automatically** trigger deployment. No manual intervention needed.

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Git Push  │ ───▶ │  Pipeline   │ ───▶ │   Deploy    │
│             │      │  Triggers   │      │   Changes   │
└─────────────┘      └─────────────┘      └─────────────┘
```

### Why This Matters

| Problem | GitOps Solution |
|---------|-----------------|
| "Who changed the firewall rules?" | `git log firewall.yml` |
| "Why is production different from staging?" | Diff the Git branches |
| "How do I rollback yesterday's change?" | `git revert HEAD` |
| "How do I set up a new environment?" | Clone the repo, run the pipeline |
| "Is this server configured correctly?" | Compare actual vs Git state |

---

## Chapter 3: Core Principles

### Principle 1: Git as Single Source of Truth

Everything that defines your infrastructure lives in Git:

```
infrastructure-repo/
├── services/
│   ├── grafana/
│   │   ├── docker-compose.yml
│   │   └── config/
│   ├── prometheus/
│   │   ├── docker-compose.yml
│   │   └── prometheus.yml
│   └── traefik/
│       ├── docker-compose.yml
│       └── dynamic/
├── terraform/
│   └── proxmox/
│       └── main.tf
└── ansible/
    └── playbooks/
```

**Rule**: If it's not in Git, it doesn't exist (officially).

### Principle 2: Desired State vs Actual State

```
┌─────────────────────────────────────────────────────────────────┐
│                      THE RECONCILIATION LOOP                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    ┌──────────────┐                    ┌──────────────┐         │
│    │ DESIRED STATE│                    │ ACTUAL STATE │         │
│    │  (Git Repo)  │                    │  (Servers)   │         │
│    │              │                    │              │         │
│    │ grafana:v10  │       Compare      │ grafana:v9   │         │
│    │ port: 3030   │ ◄───────────────── │ port: 3030   │         │
│    │              │                    │              │         │
│    └──────────────┘                    └──────────────┘         │
│            │                                   ▲                 │
│            │         ┌──────────────┐         │                 │
│            └────────▶│   OPERATOR   │─────────┘                 │
│                      │ (CI/CD, Flux,│                           │
│                      │   ArgoCD)    │                           │
│                      │              │                           │
│                      │ "Make actual │                           │
│                      │  match       │                           │
│                      │  desired"    │                           │
│                      └──────────────┘                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

The **operator** (GitLab CI/CD, ArgoCD, Flux) continuously:
1. Observes the **desired state** (Git)
2. Observes the **actual state** (running infrastructure)
3. Takes action to make **actual = desired**

### Principle 3: Changes Through Pull/Merge Requests

Never make changes directly. Always:

```
Feature Branch → Pull Request → Review → Merge → Auto-Deploy

Example workflow:
1. git checkout -b update-grafana-version
2. Edit docker-compose.yml (change image tag)
3. git commit -m "Update Grafana to v10.2.3"
4. git push origin update-grafana-version
5. Create Merge Request in GitLab
6. Reviewer approves (optional: run tests)
7. Merge to main
8. Pipeline automatically deploys
```

### Principle 4: Self-Healing Systems

If someone manually changes something on a server:
1. GitOps detects **drift** (actual ≠ desired)
2. GitOps **automatically corrects** to match Git

```
Manual SSH change detected!
┌─────────────────────────────────────────────────────────────┐
│ Git says: grafana:v10                                       │
│ Server has: grafana:v9 (someone ran docker pull manually)   │
│                                                             │
│ GitOps action: Redeploy grafana:v10 to match Git           │
└─────────────────────────────────────────────────────────────┘
```

This prevents **configuration drift** and ensures environments stay consistent.

---

## Chapter 4: Design Patterns

### Pattern 1: Push-Based Deployment

```
┌──────────────────────────────────────────────────────────────┐
│                    PUSH-BASED GITOPS                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│    Developer          Git Server         CI/CD        Target  │
│        │                  │                │            │     │
│        │   git push       │                │            │     │
│        │─────────────────▶│                │            │     │
│        │                  │   webhook      │            │     │
│        │                  │───────────────▶│            │     │
│        │                  │                │   deploy   │     │
│        │                  │                │───────────▶│     │
│        │                  │                │            │     │
│                                                               │
│   CI/CD PUSHES changes to the target environment             │
│                                                               │
│   Examples: GitLab CI/CD, GitHub Actions, Jenkins            │
└──────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- CI/CD server has credentials to deploy
- Triggered by webhooks on git push
- Common in traditional CI/CD pipelines
- **We use this pattern** in our homelab

**Pros:**
- Simple to understand and implement
- Works with any infrastructure
- No agent needed on target servers

**Cons:**
- CI/CD needs access credentials
- Not continuous (only runs on push)
- No automatic drift detection

### Pattern 2: Pull-Based Deployment

```
┌──────────────────────────────────────────────────────────────┐
│                    PULL-BASED GITOPS                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│    Developer          Git Server              Target          │
│        │                  │                     │             │
│        │   git push       │                     │             │
│        │─────────────────▶│                     │             │
│        │                  │                     │             │
│        │                  │◀────────────────────│             │
│        │                  │   agent polls       │             │
│        │                  │   for changes       │             │
│        │                  │                     │             │
│        │                  │────────────────────▶│             │
│        │                  │   agent pulls       │             │
│        │                  │   and applies       │             │
│                                                               │
│   Agent in cluster PULLS changes from Git                    │
│                                                               │
│   Examples: ArgoCD, Flux CD                                  │
└──────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- Agent runs inside the target environment
- Agent continuously polls Git for changes
- Agent has permissions to modify local resources
- Git server doesn't need access to target

**Pros:**
- More secure (no external credentials)
- Continuous reconciliation
- Automatic drift detection and correction
- Better for Kubernetes environments

**Cons:**
- Requires agent deployment
- More complex setup
- Overkill for Docker Compose workloads

### Pattern 3: Repository Structure Patterns

#### Monorepo (All services in one repo)

```
homelab-gitops/
├── services/
│   ├── grafana/
│   ├── prometheus/
│   ├── jellyfin/
│   └── traefik/
├── infrastructure/
│   └── terraform/
└── .gitlab-ci.yml
```

**Pros:** Easy to see everything, atomic changes across services
**Cons:** Large repo, pipeline runs for all changes

#### Polyrepo (One repo per service)

```
grafana-homelab/          prometheus-homelab/
├── docker-compose.yml    ├── docker-compose.yml
├── service.yml           ├── service.yml
└── .gitlab-ci.yml        └── .gitlab-ci.yml
```

**Pros:** Isolated changes, independent deployments, clear ownership
**Cons:** More repos to manage, harder to see big picture

**We use Polyrepo** because:
- Each service deploys independently
- Clear separation of concerns
- Easier to grant access per-service
- Smaller, focused pipelines

### Pattern 4: Environment Branching Strategies

#### Branch Per Environment

```
main        → Production
staging     → Staging environment
development → Development environment
```

#### Directory Per Environment

```
environments/
├── production/
│   └── values.yml
├── staging/
│   └── values.yml
└── development/
    └── values.yml
```

**For homelab**, we keep it simple: `main` branch = production.

---

## Chapter 5: Architecture for Homelab

### Our GitOps Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HOMELAB GITOPS ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              GITLAB SERVER                                   │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │  GitLab CE (gitlab.hrmsmrflrii.xyz)                                 │  │
│    │                                                                      │  │
│    │  ┌─────────────────────────────────────────────────────────────┐   │  │
│    │  │                    SERVICE REPOSITORIES                      │   │  │
│    │  │                                                              │   │  │
│    │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │  │
│    │  │  │ grafana  │ │prometheus│ │ jellyfin │ │ sentinel │       │   │  │
│    │  │  │ -homelab │ │ -homelab │ │ -homelab │ │   -bot   │       │   │  │
│    │  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘       │   │  │
│    │  │       │            │            │            │              │   │  │
│    │  └───────┼────────────┼────────────┼────────────┼──────────────┘   │  │
│    │          │            │            │            │                   │  │
│    │          ▼            ▼            ▼            ▼                   │  │
│    │  ┌─────────────────────────────────────────────────────────────┐   │  │
│    │  │                    GITLAB CI/CD RUNNER                       │   │  │
│    │  │                                                              │   │  │
│    │  │  Tags: homelab, docker                                      │   │  │
│    │  │  Executor: Docker                                           │   │  │
│    │  │                                                              │   │  │
│    │  │  Has: SSH keys, API credentials, network access            │   │  │
│    │  └──────────────────────────┬──────────────────────────────────┘   │  │
│    └─────────────────────────────┼──────────────────────────────────────┘  │
│                                  │                                          │
├──────────────────────────────────┼──────────────────────────────────────────┤
│                                  │ SSH / API                                │
│                                  ▼                                          │
│                         TARGET INFRASTRUCTURE                               │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │                                                                      │  │
│    │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │  │
│    │  │ docker-utils    │  │ docker-media    │  │ docker-glance   │     │  │
│    │  │ 192.168.40.13   │  │ 192.168.40.11   │  │ 192.168.40.12   │     │  │
│    │  │                 │  │                 │  │                 │     │  │
│    │  │ • Grafana       │  │ • Jellyfin      │  │ • Glance        │     │  │
│    │  │ • Prometheus    │  │ • Radarr        │  │ • APIs          │     │  │
│    │  │ • n8n           │  │ • Sonarr        │  │                 │     │  │
│    │  └─────────────────┘  └─────────────────┘  └─────────────────┘     │  │
│    │                                                                      │  │
│    │  ┌─────────────────┐  ┌─────────────────┐                          │  │
│    │  │ traefik-lxc     │  │ OPNsense        │                          │  │
│    │  │ 192.168.40.20   │  │ 192.168.91.30   │                          │  │
│    │  │                 │  │                 │                          │  │
│    │  │ Reverse Proxy   │  │ DNS API         │                          │  │
│    │  │ (auto-config)   │  │ (auto-records)  │                          │  │
│    │  └─────────────────┘  └─────────────────┘                          │  │
│    │                                                                      │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                              NOTIFICATIONS                                   │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │  Discord Webhooks → #deployment-notifications                       │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Component Breakdown

| Component | Role | Location |
|-----------|------|----------|
| **GitLab CE** | Source of truth, CI/CD orchestration | 192.168.40.23 (gitlab-vm01) |
| **GitLab Runner** | Executes pipeline jobs | Registered on GitLab VM |
| **Service Repos** | One per service, contains config | GitLab projects |
| **Docker Hosts** | Run containerized services | VLAN 40 (192.168.40.x) |
| **Traefik** | Reverse proxy, auto-configured | 192.168.40.20 |
| **OPNsense** | DNS, auto-record creation | 192.168.91.30 |
| **Discord** | Deployment notifications | Webhook integration |

### Data Flow

```
1. Developer commits to service repo
         │
         ▼
2. GitLab detects change, triggers pipeline
         │
         ▼
3. Pipeline stages execute:
   ┌────────────────────────────────────────────┐
   │ validate → deploy → configure → verify    │
   └────────────────────────────────────────────┘
         │
         ▼
4. Deploy stage:
   - SSH to target Docker host
   - Copy docker-compose.yml
   - Run docker compose up -d
         │
         ▼
5. Configure stage:
   - Create Traefik route config
   - Add DNS record via OPNsense API
         │
         ▼
6. Verify stage:
   - Poll health endpoint until healthy
         │
         ▼
7. Notify stage:
   - Send Discord embed with status
```

---

## Chapter 6: The GitOps Workflow

### The Service Repository Structure

Every service follows the same structure:

```
service-name/
│
├── .gitlab-ci.yml              # Pipeline definition
│   └── (What to do when code changes)
│
├── service.yml                 # Service metadata
│   └── (Where to deploy, what ports, routing config)
│
├── config/
│   ├── docker-compose.yml      # Container definition
│   │   └── (The actual Docker container config)
│   │
│   └── .env.example            # Environment reference
│       └── (Template for environment variables)
│
├── README.md                   # Documentation
│
└── .gitignore                  # Git ignore rules
```

### Understanding service.yml

The `service.yml` file is the **brain** of GitOps. It tells the pipeline everything it needs to know:

```yaml
# service.yml - The GitOps Configuration File

# ═══════════════════════════════════════════════════════════════
# SECTION 1: Service Identity
# ═══════════════════════════════════════════════════════════════
service:
  name: grafana                    # Unique identifier (used in paths, DNS, etc.)
  display_name: Grafana            # Human-readable (used in notifications)
  description: "Dashboards and observability"
  category: monitoring             # For organization (monitoring, media, utility)
  version: "10.2.3"                # Current version (documentation)

# ═══════════════════════════════════════════════════════════════
# SECTION 2: Deployment Target
# ═══════════════════════════════════════════════════════════════
deployment:
  # WHERE to deploy this service
  target_host: docker-vm-core-utilities01
  #
  # Available hosts:
  # ┌──────────────────────────────────┬──────────────┬─────────────┐
  # │ Host Name                        │ IP           │ Purpose     │
  # ├──────────────────────────────────┼──────────────┼─────────────┤
  # │ docker-lxc-media                 │ 192.168.40.11│ Media stack │
  # │ docker-lxc-glance                │ 192.168.40.12│ Dashboard   │
  # │ docker-vm-core-utilities01       │ 192.168.40.13│ Utilities   │
  # │ docker-lxc-bots                  │ 192.168.40.14│ Discord bots│
  # └──────────────────────────────────┴──────────────┴─────────────┘

  # Port configuration
  port: 3030                       # Port exposed on host
  container_port: 3000             # Port inside container (Grafana's default)
  install_path: /opt/grafana       # Where files go on the host

  # Environment variables (non-sensitive)
  environment:
    TZ: America/New_York
    GF_SERVER_ROOT_URL: https://grafana.hrmsmrflrii.xyz

  # Secrets - values come from GitLab CI/CD Variables
  secrets:
    - name: GF_SECURITY_ADMIN_PASSWORD   # Env var name in container
      source: GRAFANA_ADMIN_PASSWORD     # GitLab variable name

  # Health check configuration
  healthcheck:
    enabled: true
    endpoint: /api/health          # Path to check
    port: 3030                     # Port to check
    interval: 30                   # Seconds between checks
    timeout: 10                    # Max wait time
    retries: 3                     # Attempts before failing
    expected_status: [200]         # What HTTP codes mean "healthy"

# ═══════════════════════════════════════════════════════════════
# SECTION 3: Traefik Reverse Proxy
# ═══════════════════════════════════════════════════════════════
traefik:
  enabled: true                    # Create routing rules
  subdomain: grafana               # → grafana.hrmsmrflrii.xyz
  entrypoints: [websecure]         # HTTPS only
  tls:
    enabled: true
    cert_resolver: letsencrypt     # Automatic SSL certificates
  middlewares: []                  # Optional: authentik-auth, rate-limit
  headers:
    frame_deny: false              # Allow embedding in iframes (for Glance)

# ═══════════════════════════════════════════════════════════════
# SECTION 4: DNS Configuration
# ═══════════════════════════════════════════════════════════════
dns:
  enabled: true                    # Auto-create DNS record
  hostname: grafana                # Usually same as subdomain
  # IP defaults to Traefik (192.168.40.20)

# ═══════════════════════════════════════════════════════════════
# SECTION 5: Integrations
# ═══════════════════════════════════════════════════════════════
authentik:
  enabled: false                   # Require SSO login?
  method: forward_auth             # How to integrate

glance:
  enabled: true                    # Add to dashboard?
  page: compute                    # Which Glance page
  bookmark:
    enabled: true
    group: Monitoring
    description: "Observability dashboards"

watchtower:
  enabled: true                    # Auto-update when new image available
  notify_on_update: true           # Send Discord notification

notifications:
  discord:
    enabled: true                  # Deployment notifications
```

### Understanding .gitlab-ci.yml

The `.gitlab-ci.yml` file defines **what happens** when you push code:

```yaml
# .gitlab-ci.yml - The Pipeline Definition

# ═══════════════════════════════════════════════════════════════
# PIPELINE STAGES
# ═══════════════════════════════════════════════════════════════
# Stages run in order. Each stage can have multiple jobs.
#
#   validate  →  deploy  →  configure  →  verify  →  notify
#      │           │           │            │          │
#      │           │           │            │          └─ Discord
#      │           │           │            └─ Health check
#      │           │           └─ Traefik + DNS
#      │           └─ SSH deploy container
#      └─ Check YAML syntax

stages:
  - validate
  - deploy
  - configure
  - verify
  - notify

# ═══════════════════════════════════════════════════════════════
# WHEN TO RUN
# ═══════════════════════════════════════════════════════════════
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"      # Auto-run on main
    - if: $CI_PIPELINE_SOURCE == "web"      # Manual trigger via UI
    - if: $CI_PIPELINE_SOURCE == "trigger"  # API trigger

# ═══════════════════════════════════════════════════════════════
# STAGE 1: VALIDATE
# ═══════════════════════════════════════════════════════════════
# Purpose: Catch errors BEFORE deploying

validate:service-yml:
  stage: validate
  script:
    - yq eval '.' service.yml > /dev/null  # Parse YAML
    - # Check required fields exist
    - SERVICE_NAME=$(yq '.service.name' service.yml)
    - if [ "$SERVICE_NAME" = "null" ]; then exit 1; fi

validate:docker-compose:
  stage: validate
  script:
    - docker compose -f config/docker-compose.yml config --quiet

# ═══════════════════════════════════════════════════════════════
# STAGE 2: DEPLOY
# ═══════════════════════════════════════════════════════════════
# Purpose: Copy files to target host and start container

deploy:container:
  stage: deploy
  before_script:
    # Setup SSH with the private key from GitLab variables
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_ed25519
    - chmod 600 ~/.ssh/id_ed25519

  script:
    # 1. Parse service.yml to know where to deploy
    - TARGET_HOST=$(yq '.deployment.target_host' service.yml)
    - # Map hostname to IP...

    # 2. Generate .env file from secrets
    - python3 generate_env.py

    # 3. Copy files to target
    - scp config/docker-compose.yml user@$TARGET_IP:/opt/service/
    - scp config/.env user@$TARGET_IP:/opt/service/

    # 4. Deploy container
    - ssh user@$TARGET_IP "cd /opt/service && docker compose up -d"

# ═══════════════════════════════════════════════════════════════
# STAGE 3: CONFIGURE
# ═══════════════════════════════════════════════════════════════
# Purpose: Set up routing and DNS

configure:traefik:
  stage: configure
  script:
    # Generate Traefik config from service.yml
    - |
      cat > /tmp/traefik-$SERVICE.yml << EOF
      http:
        routers:
          $SERVICE:
            rule: "Host(\`$SUBDOMAIN.$DOMAIN\`)"
            service: $SERVICE
        services:
          $SERVICE:
            loadBalancer:
              servers:
                - url: "http://$TARGET_IP:$PORT"
      EOF

    # Copy to Traefik
    - scp /tmp/traefik-$SERVICE.yml root@192.168.40.20:/opt/traefik/config/dynamic/

configure:dns:
  stage: configure
  script:
    # Add DNS via OPNsense API
    - curl -X POST "https://192.168.91.30/api/unbound/settings/addHostOverride" ...

# ═══════════════════════════════════════════════════════════════
# STAGE 4: VERIFY
# ═══════════════════════════════════════════════════════════════
# Purpose: Make sure deployment succeeded

verify:health:
  stage: verify
  script:
    # Poll health endpoint until it responds
    - |
      for i in $(seq 1 30); do
        if curl -sf "http://$TARGET_IP:$PORT/health"; then
          echo "Healthy!"
          exit 0
        fi
        sleep 5
      done
      exit 1  # Failed

# ═══════════════════════════════════════════════════════════════
# STAGE 5: NOTIFY
# ═══════════════════════════════════════════════════════════════
# Purpose: Tell humans what happened

notify:success:
  stage: notify
  when: on_success
  script:
    - curl -X POST "$DISCORD_WEBHOOK_URL" -d '{"embeds":[...]}'

notify:failure:
  stage: notify
  when: on_failure
  script:
    - curl -X POST "$DISCORD_WEBHOOK_URL" -d '{"embeds":[...]}'
```

### The Complete Flow Visualized

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GITOPS DEPLOYMENT FLOW                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  DEVELOPER                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                        │  │
│  │   1. Edit docker-compose.yml                                          │  │
│  │      └─ Change image: grafana/grafana:10.2.3                          │  │
│  │                                                                        │  │
│  │   2. Commit and push                                                   │  │
│  │      └─ git commit -m "Update Grafana to v10.2.3"                     │  │
│  │      └─ git push origin main                                          │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│  GITLAB                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                        │  │
│  │   3. Webhook triggers pipeline                                        │  │
│  │      └─ .gitlab-ci.yml defines stages                                 │  │
│  │                                                                        │  │
│  │   4. Runner picks up job                                              │  │
│  │      └─ Tagged: homelab, docker                                       │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│  PIPELINE EXECUTION                                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                        │  │
│  │   Stage 1: VALIDATE                                                   │  │
│  │   ┌─────────────────────────────────────────────────────────────────┐│  │
│  │   │ ✓ service.yml syntax valid                                      ││  │
│  │   │ ✓ docker-compose.yml syntax valid                               ││  │
│  │   │ ✓ Required fields present                                       ││  │
│  │   └─────────────────────────────────────────────────────────────────┘│  │
│  │                              │                                        │  │
│  │                              ▼                                        │  │
│  │   Stage 2: DEPLOY                                                     │  │
│  │   ┌─────────────────────────────────────────────────────────────────┐│  │
│  │   │ 1. Parse service.yml → target: docker-vm-core-utilities01       ││  │
│  │   │ 2. Generate .env from GitLab variables                          ││  │
│  │   │ 3. SSH to 192.168.40.13                                         ││  │
│  │   │ 4. Copy docker-compose.yml to /opt/grafana/                     ││  │
│  │   │ 5. Run: docker compose pull                                     ││  │
│  │   │ 6. Run: docker compose up -d                                    ││  │
│  │   └─────────────────────────────────────────────────────────────────┘│  │
│  │                              │                                        │  │
│  │                              ▼                                        │  │
│  │   Stage 3: CONFIGURE                                                  │  │
│  │   ┌─────────────────────────────────────────────────────────────────┐│  │
│  │   │ Traefik:                                                        ││  │
│  │   │ • Generate grafana.yml with routing rules                       ││  │
│  │   │ • Copy to /opt/traefik/config/dynamic/                          ││  │
│  │   │ • Traefik auto-reloads (file watcher)                           ││  │
│  │   │                                                                  ││  │
│  │   │ DNS:                                                             ││  │
│  │   │ • Call OPNsense API                                             ││  │
│  │   │ • Create: grafana.hrmsmrflrii.xyz → 192.168.40.20               ││  │
│  │   └─────────────────────────────────────────────────────────────────┘│  │
│  │                              │                                        │  │
│  │                              ▼                                        │  │
│  │   Stage 4: VERIFY                                                     │  │
│  │   ┌─────────────────────────────────────────────────────────────────┐│  │
│  │   │ Health Check Loop:                                              ││  │
│  │   │ • Attempt 1: curl http://192.168.40.13:3030/api/health → 503   ││  │
│  │   │ • Attempt 2: curl http://192.168.40.13:3030/api/health → 503   ││  │
│  │   │ • Attempt 3: curl http://192.168.40.13:3030/api/health → 200 ✓ ││  │
│  │   │ • Service is healthy!                                           ││  │
│  │   └─────────────────────────────────────────────────────────────────┘│  │
│  │                              │                                        │  │
│  │                              ▼                                        │  │
│  │   Stage 5: NOTIFY                                                     │  │
│  │   ┌─────────────────────────────────────────────────────────────────┐│  │
│  │   │ Discord Webhook:                                                ││  │
│  │   │ ┌─────────────────────────────────────────────────────────────┐││  │
│  │   │ │ ✅ Deployed: Grafana                                        │││  │
│  │   │ │ Service deployed successfully via GitLab CI/CD              │││  │
│  │   │ │                                                             │││  │
│  │   │ │ 🌐 URL: https://grafana.hrmsmrflrii.xyz                    │││  │
│  │   │ │ 🔗 Pipeline: #12345                                        │││  │
│  │   │ │ 👤 Triggered by: hermes                                    │││  │
│  │   │ └─────────────────────────────────────────────────────────────┘││  │
│  │   └─────────────────────────────────────────────────────────────────┘│  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  RESULT                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                        │  │
│  │   ✓ Grafana v10.2.3 running on docker-utils                          │  │
│  │   ✓ Accessible at https://grafana.hrmsmrflrii.xyz                    │  │
│  │   ✓ DNS resolves correctly                                            │  │
│  │   ✓ Team notified via Discord                                         │  │
│  │   ✓ Full audit trail in Git history                                   │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Part II: Implementation

## Chapter 7: Implementation Guide

### Prerequisites

Before implementing GitOps, ensure you have:

| Requirement | Status | Notes |
|-------------|--------|-------|
| GitLab CE running | ✅ | gitlab.hrmsmrflrii.xyz |
| GitLab Runner configured | ✅ | Tags: homelab, docker |
| Docker hosts accessible | ✅ | SSH with key authentication |
| Traefik running | ✅ | 192.168.40.20 |
| OPNsense API access | ✅ | For DNS automation |
| Discord webhook | ✅ | For notifications |

### Step 1: Configure GitLab Group Variables

Variables set at the **group level** are inherited by all projects in that group.

1. Go to GitLab → Your Group → Settings → CI/CD → Variables

2. Add these variables:

| Variable | Type | Value | Protected | Masked |
|----------|------|-------|-----------|--------|
| `SSH_PRIVATE_KEY` | File | Contents of `~/.ssh/homelab_ed25519` | Yes | No |
| `DISCORD_WEBHOOK_URL` | Variable | Your Discord webhook URL | Yes | Yes |
| `OPNSENSE_API_KEY` | Variable | OPNsense API key | Yes | No |
| `OPNSENSE_API_SECRET` | Variable | OPNsense API secret | Yes | Yes |

> **Important**: `SSH_PRIVATE_KEY` must be type **File**, not Variable!

### Step 2: Verify GitLab Runner

Check that your runner is available:

```bash
# On GitLab VM
gitlab-runner list

# Should show something like:
# homelab-runner   Executor=docker   Tags=homelab,docker
```

Verify runner is connected:
1. GitLab → Admin → CI/CD → Runners
2. Should show green "online" status

### Step 3: Create Your First Service Repository

1. **Create new project** in GitLab:
   - GitLab → New Project → Create blank project
   - Name: `grafana-homelab` (or your service name)
   - Create in your homelab group

2. **Clone locally**:
   ```bash
   git clone <email>:homelab/grafana-homelab.git
   cd grafana-homelab
   ```

3. **Copy template files** (from this repo):
   ```bash
   # Copy from gitops-templates/gitlab-service-template/
   cp -r /path/to/gitops-templates/gitlab-service-template/* .
   ```

4. **Customize files** (see next chapter)

---

## Chapter 8: Creating Your First GitOps Service

Let's create a complete GitOps deployment for Grafana step by step.

### Step 1: Create service.yml

```yaml
# service.yml
service:
  name: grafana
  display_name: Grafana
  description: "Observability and monitoring dashboards"
  category: monitoring
  version: "10.2.3"

deployment:
  target_host: docker-vm-core-utilities01
  port: 3030
  container_port: 3000
  install_path: /opt/grafana

  environment:
    TZ: America/New_York
    GF_SERVER_ROOT_URL: https://grafana.hrmsmrflrii.xyz
    GF_SERVER_HTTP_PORT: "3000"
    GF_AUTH_DISABLE_LOGIN_FORM: "false"

  secrets:
    - name: GF_SECURITY_ADMIN_PASSWORD
      source: GRAFANA_ADMIN_PASSWORD

  healthcheck:
    enabled: true
    endpoint: /api/health
    port: 3030
    interval: 30
    timeout: 10
    retries: 3
    expected_status: [200]

traefik:
  enabled: true
  subdomain: grafana
  entrypoints: [websecure]
  tls:
    enabled: true
    cert_resolver: letsencrypt
  middlewares: []
  headers:
    frame_deny: false  # Allow embedding in Glance

dns:
  enabled: true
  hostname: grafana

authentik:
  enabled: false

glance:
  enabled: true
  page: compute
  bookmark:
    enabled: true
    group: Monitoring
    description: "Dashboards and visualizations"
  monitor:
    enabled: true
    widget: monitor
    health_endpoint: /api/health

watchtower:
  enabled: true
  notify_on_update: true

notifications:
  discord:
    enabled: true

metadata:
  maintainer: hermes
  docs: https://grafana.com/docs/
  tags: [monitoring, dashboards, observability]
```

### Step 2: Create docker-compose.yml

```yaml
# config/docker-compose.yml
services:
  grafana:
    image: grafana/grafana:10.2.3
    container_name: grafana
    restart: unless-stopped

    ports:
      - "${SERVICE_PORT:-3030}:3000"

    environment:
      - TZ=${TZ:-America/New_York}
      - GF_SERVER_ROOT_URL=${GF_SERVER_ROOT_URL:-https://grafana.hrmsmrflrii.xyz}
      - GF_SERVER_HTTP_PORT=3000
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GF_SECURITY_ADMIN_PASSWORD:-admin}
      - GF_AUTH_DISABLE_LOGIN_FORM=${GF_AUTH_DISABLE_LOGIN_FORM:-false}
      - GF_USERS_ALLOW_SIGN_UP=false

    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning:ro

    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

    labels:
      - "com.centurylinklabs.watchtower.enable=true"
      - "homelab.service=grafana"
      - "homelab.managed-by=gitlab-gitops"

    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  grafana-data:
```

### Step 3: Create .gitlab-ci.yml

Use the complete pipeline from the template. The key sections:

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - deploy
  - configure
  - verify
  - notify

# ... (full pipeline from template)
```

### Step 4: Add Project-Specific Secrets

1. Go to your project → Settings → CI/CD → Variables
2. Add:
   - `GRAFANA_ADMIN_PASSWORD` = your secure password (masked)

### Step 5: Push and Deploy

```bash
# Add all files
git add .

# Commit
git commit -m "Initial Grafana GitOps configuration"

# Push to main (triggers pipeline)
git push origin main
```

### Step 6: Watch the Pipeline

1. Go to GitLab → Your Project → CI/CD → Pipelines
2. Click the running pipeline
3. Watch each stage execute
4. Check Discord for the notification

### Step 7: Verify Deployment

```bash
# Check container is running
ssh hermes-admin@192.168.40.13 "docker ps | grep grafana"

# Check health endpoint
curl http://192.168.40.13:3030/api/health

# Check via Traefik
curl https://grafana.hrmsmrflrii.xyz/api/health

# Check DNS
nslookup grafana.hrmsmrflrii.xyz
```

### Making Changes (GitOps Way)

To update Grafana version:

```bash
# 1. Edit docker-compose.yml
# Change: image: grafana/grafana:10.2.3
# To:     image: grafana/grafana:10.3.0

# 2. Commit and push
git add config/docker-compose.yml
git commit -m "Update Grafana to v10.3.0"
git push origin main

# 3. Pipeline automatically:
#    - Deploys new version
#    - Keeps same configuration
#    - Notifies Discord
```

### Rollback (If Something Goes Wrong)

**Option 1: Git Revert**
```bash
# Revert the last commit
git revert HEAD
git push origin main
# Pipeline deploys the reverted state
```

**Option 2: Manual Rollback Job**
1. Go to CI/CD → Pipelines
2. Find the current pipeline
3. Click "rollback" job → Run

This restores from the `.bak` files created during deployment.

---

## Chapter 9: Advanced Topics

### Topic 1: Multi-Environment GitOps

For staging/production environments:

```
grafana-homelab/
├── environments/
│   ├── production/
│   │   ├── service.yml       # target: docker-utils
│   │   └── values.yml        # Production settings
│   └── staging/
│       ├── service.yml       # target: docker-staging
│       └── values.yml        # Staging settings
└── .gitlab-ci.yml
```

Pipeline logic:
```yaml
deploy:production:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  environment: production

deploy:staging:
  rules:
    - if: $CI_COMMIT_BRANCH == "staging"
  environment: staging
```

### Topic 2: Secret Rotation

When you need to rotate a secret:

1. Update GitLab CI/CD Variable with new value
2. Trigger pipeline (or wait for next deployment)
3. New secret is automatically deployed

```bash
# Force redeploy to pick up new secret
git commit --allow-empty -m "Rotate GRAFANA_ADMIN_PASSWORD"
git push origin main
```

### Topic 3: Conditional Deployments

Deploy only when specific files change:

```yaml
deploy:container:
  rules:
    - changes:
        - service.yml
        - config/**/*
    - if: $CI_PIPELINE_SOURCE == "web"  # Always allow manual
```

### Topic 4: Scheduled Deployments

Auto-update services on a schedule:

```yaml
# In .gitlab-ci.yml
scheduled:update:
  stage: deploy
  script:
    - docker compose pull
    - docker compose up -d
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"

# Then in GitLab:
# Settings → CI/CD → Schedules → New Schedule
# - Description: Weekly auto-update
# - Interval: 0 3 * * 0 (Sunday 3am)
# - Target branch: main
```

### Topic 5: Pipeline Includes

Share common CI/CD configuration across repos:

```yaml
# In each service repo's .gitlab-ci.yml
include:
  - project: 'homelab/gitops-templates'
    file: '/ci/deploy-docker-service.yml'
    ref: main

# Variables specific to this service
variables:
  SERVICE_NAME: grafana
```

### Topic 6: Deployment Approval Gates

Require manual approval before production:

```yaml
deploy:production:
  stage: deploy
  environment:
    name: production
    action: start
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # Requires click to deploy
```

---

## Chapter 10: Troubleshooting

### Common Issues and Solutions

#### Issue 1: Pipeline Fails at SSH

**Error:** `Permission denied (publickey)`

**Diagnosis:**
```bash
# Check if variable exists
# GitLab → Project → Settings → CI/CD → Variables
# Look for SSH_PRIVATE_KEY (should be File type)

# Test manually
ssh -i /path/to/key hermes-admin@192.168.40.13
```

**Solutions:**
1. Verify `SSH_PRIVATE_KEY` variable is type **File**, not Variable
2. Check the key matches `~/.ssh/authorized_keys` on target
3. Verify key has no extra whitespace or newlines

#### Issue 2: Container Starts But Health Check Fails

**Error:** `Health check failed after 30 attempts`

**Diagnosis:**
```bash
# SSH to host
ssh hermes-admin@192.168.40.13

# Check container status
docker ps -a | grep grafana

# Check logs
docker logs grafana --tail 50

# Try health endpoint manually
curl -v http://localhost:3030/api/health
```

**Solutions:**
1. Container may need more startup time → increase `start_period` in compose
2. Health endpoint path may be wrong → check service documentation
3. Port mapping may be incorrect → verify compose ports

#### Issue 3: Traefik Route Not Working

**Error:** 404 or "Bad Gateway" when accessing URL

**Diagnosis:**
```bash
# Check Traefik config exists
ssh root@192.168.40.20 "ls /opt/traefik/config/dynamic/"

# Check Traefik logs
ssh root@192.168.40.20 "docker logs traefik --tail 50"

# Verify backend is accessible from Traefik
ssh root@192.168.40.20 "curl http://192.168.40.13:3030/api/health"
```

**Solutions:**
1. Config file syntax error → check YAML formatting
2. Backend not accessible → check Docker network/firewall
3. Traefik didn't reload → restart Traefik container

#### Issue 4: DNS Record Not Created

**Error:** `nslookup` returns NXDOMAIN

**Diagnosis:**
```bash
# Check OPNsense API credentials
curl -sk -u "$KEY:$SECRET" \
  "https://192.168.91.30/api/unbound/settings/searchHostOverride"

# Check if record exists
curl -sk -u "$KEY:$SECRET" \
  "https://192.168.91.30/api/unbound/settings/searchHostOverride" \
  -d "searchPhrase=grafana"
```

**Solutions:**
1. API credentials incorrect → regenerate in OPNsense
2. Record already exists → pipeline skips creation
3. Unbound not reconfigured → call reconfigure API endpoint

#### Issue 5: Secrets Not Being Injected

**Error:** Service starts with default/empty credentials

**Diagnosis:**
```bash
# Check .env file on target
ssh hermes-admin@192.168.40.13 "cat /opt/grafana/.env"

# Should contain:
# GF_SECURITY_ADMIN_PASSWORD=<redacted>
```

**Solutions:**
1. GitLab variable name doesn't match `source` in service.yml
2. Variable not accessible to pipeline (check Protected/scopes)
3. Python script generating .env has error → check pipeline logs

### Debug Pipeline Locally

You can test pipeline steps locally:

```bash
# SSH to a test machine
ssh hermes-admin@192.168.40.13

# Simulate what the pipeline does
cd /tmp
git clone your-repo
cd your-repo

# Parse service.yml (like pipeline does)
SERVICE_NAME=$(yq '.service.name' service.yml)
echo "Service: $SERVICE_NAME"

# Test docker compose
docker compose -f config/docker-compose.yml config
```

### Pipeline Debug Mode

Add debug output to pipeline:

```yaml
deploy:container:
  script:
    - set -x  # Enable debug output
    - echo "SERVICE_NAME=$SERVICE_NAME"
    - echo "TARGET_IP=$TARGET_IP"
    - cat service.yml
    - # ... rest of script
```

---

# Appendices

## Appendix A: Complete File Templates

### Template: service.yml

Located at: `gitops-templates/gitlab-service-template/service.yml`

### Template: .gitlab-ci.yml

Located at: `gitops-templates/gitlab-service-template/.gitlab-ci.yml`

### Template: docker-compose.yml

Located at: `gitops-templates/gitlab-service-template/config/docker-compose.yml`

## Appendix B: GitLab CI/CD Variable Reference

| Variable | Scope | Type | Description |
|----------|-------|------|-------------|
| `SSH_PRIVATE_KEY` | Group | File | SSH key for deployment |
| `DISCORD_WEBHOOK_URL` | Group | Variable | Discord notifications |
| `OPNSENSE_API_KEY` | Group | Variable | DNS automation |
| `OPNSENSE_API_SECRET` | Group | Variable | DNS automation (masked) |
| `GRAFANA_ADMIN_PASSWORD` | Project | Variable | Service-specific secret |

## Appendix C: Target Host Reference

| Host | IP | SSH User | Purpose |
|------|-----|----------|---------|
| `docker-lxc-media` | 192.168.40.11 | hermes-admin | Media stack |
| `docker-lxc-glance` | 192.168.40.12 | root | Dashboard |
| `docker-vm-core-utilities01` | 192.168.40.13 | hermes-admin | Monitoring/utilities |
| `docker-lxc-bots` | 192.168.40.14 | root | Discord bots |
| `traefik-lxc` | 192.168.40.20 | root | Reverse proxy |
| `authentik-lxc` | 192.168.40.21 | root | SSO |

## Appendix D: Glossary

| Term | Definition |
|------|------------|
| **GitOps** | Using Git as the source of truth for infrastructure |
| **Declarative** | Describing what you want, not how to achieve it |
| **Reconciliation** | Process of making actual state match desired state |
| **Drift** | When actual state differs from desired state |
| **Pipeline** | Automated sequence of jobs triggered by Git events |
| **Runner** | Agent that executes pipeline jobs |
| **Artifact** | File or data passed between pipeline stages |
| **Push-based** | CI/CD pushes changes to target (our model) |
| **Pull-based** | Agent pulls changes from Git (ArgoCD/Flux model) |

---

## Summary

You've learned:

1. **What GitOps is**: Using Git as the single source of truth for infrastructure
2. **Why it matters**: Audit trails, reproducibility, self-documenting changes
3. **Core principles**: Declarative config, version control, automatic reconciliation
4. **Design patterns**: Push vs pull, monorepo vs polyrepo
5. **Architecture**: GitLab + Runner + Docker hosts + Traefik + DNS
6. **Implementation**: service.yml, .gitlab-ci.yml, docker-compose.yml
7. **Workflow**: Commit → Pipeline → Deploy → Configure → Verify → Notify
8. **Advanced topics**: Multi-environment, secret rotation, approval gates
9. **Troubleshooting**: Common issues and debug techniques

### Next Steps

1. **Create your first GitOps service** using the templates
2. **Migrate existing services** one at a time
3. **Set up monitoring** of your GitOps pipelines
4. **Document your services** as you migrate them

### Related Documentation

- [20 - GitLab CI-CD Automation](20%20-%20GitLab%20CI-CD%20Automation.md) - GitLab CI/CD details
- [06 - Ansible Automation](06%20-%20Ansible%20Automation.md) - Ansible integration
- [09 - Traefik Reverse Proxy](09%20-%20Traefik%20Reverse%20Proxy.md) - Traefik configuration
- [15 - New Service Onboarding Guide](15%20-%20New%20Service%20Onboarding%20Guide.md) - Adding new services

---

*Last updated: January 14, 2026*
