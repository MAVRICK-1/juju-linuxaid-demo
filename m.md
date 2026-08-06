# Kubernetes Is Not a Platform: Building One with Juju + LinuxAid
## The Complete Guide — Talk, Deep Dive, Comparisons, and Everything Explained

**Speaker:** Rishi Mondal — SRE at Obmondo | CNCF KubeStellar Maintainer | Docker Captain

---

## Table of Contents

1. [The Problem — Why Kubernetes Is Not Enough](#part-1--the-problem)
2. [Juju — What It Is, How It Works, Everything](#part-2--juju-an-operator-for-everything-above-the-os)
3. [The OS Gap Juju Ignores](#part-3--the-gap-juju-politely-ignores)
4. [LinuxAid — What It Is, How It Works, Does It Reboot?](#part-4--linuxaid-the-bottom-half-nobody-wanted-to-own)
5. [The Combination — Where It Becomes a Platform](#part-5--where-this-stops-being-a-cluster-and-starts-being-a-platform)
6. [Before vs After — Real Scenarios](#part-6--before-vs-after)
7. [Key Takeaways](#part-7--key-takeaways)
8. [Juju vs Helm vs Ansible — Full Comparison](#juju-vs-helm-vs-ansible--full-comparison)
9. [LinuxAid vs Ansible vs Cloud-Init — Full Comparison](#linuxaid-vs-ansible-vs-cloud-init--full-comparison)
10. [Does LinuxAid Reboot Your Server?](#does-linuxaid-reboot-your-server)
11. [How the Agent Run Works — Step by Step](#how-the-agent-run-works--step-by-step)
12. [Glossary](#glossary--every-term-explained)

---

# Part 1 — The Problem

## What Kubernetes Actually Gives You

### The VS Code Analogy

Imagine VS Code gives you a text editor.
That's genuinely useful. But you still need to install:
- A language extension (Python, Go, TypeScript)
- A linter, a formatter, a debugger
- A Git extension, a terminal

VS Code is not an IDE out of the box. It is the foundation for one.

**Kubernetes is exactly like this.**

Kubernetes gives you a way to run containers. Genuinely useful.
But you still need:

| Missing piece | Why it hurts |
|---|---|
| Hardened OS on every node | Security starts at the host, not the YAML |
| Day-2 ops for stateful apps | Upgrades/backups/rotation = manual labor forever |
| App-to-app wiring | Every secret typed by hand, every endpoint hardcoded |
| Observability | You are now a full-time Prometheus plumber |

Kubernetes gave you the floor. Enjoy building the rest of the house yourself.

---

## How Teams "Solve" This

```
OS hardening      → Ansible playbook (runs once, ages like milk)
Node exporters    → Another role (forgotten by Q2)
K8s               → kubeadm, manually upgraded, manually feared
App deploy        → Helm (installs great, day-2 is your problem)
App wiring        → env vars + Secrets, hand-maintained forever
Observability     → Prometheus + Grafana, duct-taped together
```

Six layers. Zero of them talk to each other.
It works — right up until 3 AM, when nobody knows which layer broke.

---

# Part 2 — Juju: An Operator for Everything Above the OS

## What Juju Actually Is

Juju is Canonical's **model-driven application operator**.

It deploys **Charmed Operators** — packages that know their *entire* lifecycle,
not just "install and pray":

```
Install → Configure → Relate → Scale → Upgrade → Backup
```

**400+ charms on Charmhub:** PostgreSQL, Kafka, OpenSearch, Vault, Grafana, MicroK8s...

### The VS Code Extension Analogy

If Kubernetes is VS Code (the base platform), Juju Charms are extensions.
But unlike regular extensions that just add features,
Juju Charms also:
- Wire themselves to other extensions automatically (like if Python ext auto-configured your debugger)
- Update their own configuration when the environment changes
- Know how to back themselves up
- Coordinate upgrades with dependent extensions

---

## The Juju Controller — The Brain

When you set up Juju, the first thing you create is a **Controller**.
It is a small VM or set of VMs that:
- Tracks the state of every application you deployed
- Sends instructions to every Juju agent
- Stores history of every operation
- Manages relations between applications

You create it once. Everything else flows through it.

```
You (CLI/UI)
    │
    ▼
Juju Controller  ←── always running, the source of truth
    │
    ├── Model: production
    │     ├── postgresql       ← Juju agent on each unit
    │     └── my-app           ← Juju agent on each unit
    │
    └── Model: monitoring
          ├── prometheus
          └── grafana
```

---

## What is a Juju Model?

A **Model** is a logical namespace for a group of applications that work together.

Think of it like a Kubernetes namespace — except it also:
- Tracks relations between apps
- Manages credentials shared between apps
- Gives you an application-level status view

```
Juju Controller
├── model: k8s          → MicroK8s cluster
├── model: monitoring   → Prometheus + Grafana + Loki + Alertmanager
├── model: apps         → PostgreSQL + my-app (related)
└── model: staging      → exact copy of apps, isolated
```

Applications in the same model relate directly.
Applications in different models use **cross-model relations** (CMR).

---

## The Relation — Juju's Most Important Concept

This is what makes Juju genuinely different from every other deployment tool.

### Without Juju — the manual pain

You deploy PostgreSQL. You want your app to connect to it.

```
Step 1: kubectl get secret postgresql -o jsonpath='{.data.password}' | base64 -d
Step 2: Copy that password somewhere
Step 3: kubectl create secret generic my-app-db --from-literal=password=<paste>
Step 4: Edit your app Deployment: add env vars DB_HOST, DB_USER, DB_PASSWORD
Step 5: kubectl apply -f deployment.yaml
Step 6: PostgreSQL upgrades, password rotates → repeat steps 1–5
Step 7: For staging environment → repeat everything from scratch
Step 8: For the third environment → repeat again
```

Every time PostgreSQL changes, you chase it.
Every new environment is a manual redo.
If you forget a step, the app breaks silently.

### With Juju — one command

```bash
juju relate my-app postgresql
```

Juju automatically:
1. Tells PostgreSQL to create a dedicated database `my-app-db`
2. Creates a user `my-app` with a strong random password
3. Passes `host`, `port`, `dbname`, `user`, `password` to `my-app`'s charm
4. `my-app`'s charm receives them and configures the application
5. On PostgreSQL upgrade → charm sends updated credentials → `my-app` reconfigures
6. For staging → same `juju relate` → identical wiring, separate credentials

**You never see the password. You never copy it. You never manage it.**

---

## Juju vs Helm — Head to Head

| Capability | Helm | Juju |
|---|---|---|
| Install | ✅ | ✅ |
| Upgrade | You babysit values drift | Charm owns it |
| Scale | Edit replicas by hand | `juju add-unit` |
| Connect two apps | Manual secrets + env vars | `juju relate` |
| Rotate credentials | Edit → redeploy → restart | Charm handles it |
| Backup | Write your own CronJob | `juju run postgresql/0 create-backup` |
| Status | `kubectl get pods` (infra view) | `juju status` (application-aware) |
| Continuous reconciliation | ❌ | ✅ (controller watches) |
| Cross-app dependency tracking | ❌ | ✅ (relation graph) |

**Helm is a package manager. Juju is an operator. Different sports.**

Helm answers: "How do I install this?"
Juju answers: "How do I *operate* this — today, and every day after?"

---

## COS Lite — The Observability Stack as a Charm Bundle

COS Lite (Canonical Observability Stack Lite) is a single Juju bundle
that deploys and wires together a complete observability stack.

```bash
juju deploy cos-lite --trust
juju relate cos-lite microk8s     # auto-wires cluster scraping
```

| Component | Role | Manual alternative |
|-----------|------|--------------------|
| Prometheus | Metrics + alert evaluation | Deploy manually, write prometheus.yml by hand |
| Grafana | Dashboards | Deploy manually, add datasources by clicking |
| Alertmanager | Alert routing + dedup | Deploy manually, configure receivers by hand |
| Loki | Log aggregation | Deploy manually, configure log shippers |
| Traefik | Ingress for all components | Write an Ingress manifest for each |

**Why COS Lite is different:**

Without it, after deploying all 5 components you *still* have to:
- Add Prometheus as a datasource in Grafana (manually)
- Add Loki as a datasource in Grafana (manually)
- Configure Prometheus → Alertmanager (manually)
- Configure Prometheus scrape jobs (manually)
- Write Ingress for each service (manually)

With COS Lite, all of this is wired via **relations** automatically.
Add a new application, relate it to COS Lite → its dashboards appear.

---

## MicroK8s as a Charm — What This Means

Juju doesn't just deploy applications *on* Kubernetes.
It can deploy **Kubernetes itself** as a charmed operator.

```bash
juju deploy microk8s --channel 1.28/stable   # control plane
juju add-unit microk8s                        # add worker
juju add-unit microk8s                        # add another worker
juju config microk8s channel=1.30/stable      # upgrade the cluster
```

The MicroK8s charm handles:
- Multi-node bootstrap and join token management automatically
- High availability control plane
- Channel-based upgrades — no manual kubeadm steps
- Addons (DNS, storage, ingress) managed as charm config

Your entire Kubernetes cluster lifecycle is now a Juju model:
version-controlled in Git, reproducible on any cloud.

---

# Part 3 — The Gap Juju Politely Ignores

## What Juju Won't Touch

Juju rules the cluster and the applications on it.
**The OS underneath is not its problem.**

Every node still needs:

```
SSH hardening        → who logs in, and how
Firewall             → kernel-level traffic rules
Auto security updates → patches without a human awake
Prometheus exporters → host-level CPU/mem/disk/systemd metrics
Compliance           → CIS benchmarks, GDPR, NIS2
```

Ansible or cloud-init can do this — once.
Then the node drifts silently, and nobody notices until the audit.

**That is the gap.**

---

# Part 4 — LinuxAid: The Bottom Half Nobody Wanted to Own

## What LinuxAid Is — Full Explanation

LinuxAid is an **open-source OS management platform** that manages the
operating system on Linux servers — not the containers, not Kubernetes, the actual OS.

Built on **OpenVox** (the community fork of Puppet — configuration management
that has existed since 2005 and runs on millions of servers worldwide).

**The core idea:** You describe what you want the OS to look like in YAML files stored in Git.
LinuxAid continuously reconciles every node toward that description — every 30 minutes.

### The VS Code Settings Sync Analogy

Imagine VS Code settings sync.
You configure VS Code exactly right (theme, fonts, extensions, keybindings).
You push that config to GitHub.
On every new machine, VS Code pulls the config and configures itself identically.
If you accidentally change a setting, next sync it gets corrected.

**LinuxAid does this for Linux servers.**

Your config lives in Git.
Every node continuously pulls and applies it.
If someone manually changes SSH config on a server → next agent run: corrected.
If a new server joins → first agent run: fully configured in minutes.

---

## The Two Repositories

LinuxAid uses two separate Git repositories:

**Repo 1: LinuxAid (the code — you don't edit this)**
```
github.com/Obmondo/LinuxAid
```
Contains the Puppet modules — the *logic* of how to configure things.
It knows: "how to harden SSH", "how to install node_exporter", "how to set up a firewall".

**Repo 2: Your config repo (the data — you only write YAML here)**
```
github.com/MAVRICK-1/juju-linuxaid-demo
```
Contains what you want each node to look like. No code. Just YAML.

```
your-linuxaid-config/
├── data/
│   └── global.yaml          # applies to ALL nodes
└── agents/
    ├── node-1.demo.yaml     # applies ONLY to node-1
    └── node-2.demo.yaml     # applies ONLY to node-2
```

---

## Hiera — How Configuration is Layered

**Hiera** is the configuration lookup system. The word means "hierarchy."

It resolves which value applies to a given node by checking from most specific to least:

```
Priority 1 (most specific):   agents/node-1.demo.yaml       ← only node-1
Priority 2:                   agents/tags/web/*.yaml         ← all "web" tagged nodes
Priority 3:                   facts/os.family=Debian.yaml    ← all Debian nodes
Priority 4 (least specific):  data/global.yaml               ← every node
```

**Example:**

`data/global.yaml` sets SSH port 22 for all nodes.
`agents/node-1.demo.yaml` sets SSH port 2222 for node-1 only.

Result: node-1 uses 2222, everything else uses 22.
More specific always wins. You never duplicate config — only override where needed.

---

## What `role::basic` Installs and Configures

One line in your YAML file:

```yaml
# agents/node-1.demo.yaml
classes:
  - role::basic
```

Buys you all of this:

### SSH Hardening
- Disables root login (`PermitRootLogin no`)
- Disables password authentication (key-only)
- Sets idle session timeout (no forgotten open sessions)
- Restricts cryptographic algorithms to modern ones

### Firewall
- Installs iptables
- Rules are declared in your YAML — not manually applied on the server
- If someone manually adds or removes a firewall rule, the next agent run restores it

### Automatic Security Updates
- Installs `unattended-upgrades`
- Security patches apply automatically on a schedule
- Kernel packages: updated but **not rebooted** (see next section)
- You control which packages are excluded via `blacklist`

### Prometheus Exporters (installed as GPG-signed packages)

| Exporter | Port | What it measures |
|----------|------|-----------------|
| `node_exporter` | 9100 | CPU, memory, disk, network, load average, filesystem |
| `systemd_exporter` | 9558 | State of every systemd unit (active/failed/inactive) |
| `process_exporter` | 9256 | Per-process CPU and memory usage |
| `iptables_exporter` | 9455 | Firewall rule hit counts |

Each exporter:
- Installed from a GPG-signed managed repo (not a random GitHub binary)
- Runs as a dedicated non-root system user
- Managed as a systemd service
- Configured from your hiera YAML

---

## How the Agent Run Works — Step by Step

Every 30 minutes, this happens automatically on every node:

```
Step 1  puppet agent wakes up (systemd timer or daemon)
Step 2  sends its certname (e.g. node-1.demo) to the OpenVox server
Step 3  OpenVox server reads your Git config repo (via r10k)
Step 4  compiles a "catalog" — complete list of resources that
        should exist on this node
        (files, packages, services, users, cron jobs, firewall rules...)
Step 5  catalog is sent back to the node
Step 6  puppet agent compares current state to desired state
Step 7  for each difference, puppet applies the change
Step 8  puppet reports what changed (or confirms nothing changed)
```

**What is a "resource"?**
A resource is one managed thing. Examples:
- File `/etc/ssh/sshd_config` with specific content
- Package `node-exporter` must be installed at version X
- Service `node-exporter` must be running and enabled at boot
- User `ubuntu` must have this specific SSH public key
- Cron job that runs at 02:00 every night

**LinuxAid manages hundreds of resources per node**
but you only declare the *outcome* in YAML.
The LinuxAid code figures out how to achieve it on Debian, Ubuntu, RHEL, Rocky, etc.

---

## Does LinuxAid Reboot Your Server?

**No. Not by default. Reboots are opt-in.**

This is verified from LinuxAid source code (`common/manifests/system/updates.pp`):

```yaml
# Default values in LinuxAid source
common::system::updates::manage: false    # updates disabled by default
common::system::updates::enable: false    # updates disabled by default
common::system::updates::reboot: false    # NEVER reboots by default
```

### To enable updates without reboots (most common):
```yaml
# data/global.yaml
common::system::updates::manage: true
common::system::updates::enable: true
common::system::updates::reboot: false   # default — no reboot
```

### To enable updates with reboots (explicit opt-in):
```yaml
common::system::updates::reboot: true   # you choose this deliberately
```

### What about service restarts?

When LinuxAid changes a config file (e.g. SSH config, exporter config),
it restarts only the **affected service** — not the server.

- SSH config changed → `sshd` restarts (your existing SSH session stays connected)
- `node_exporter` config changed → `node_exporter` service restarts (takes <1 second)
- Firewall rules changed → iptables rules reload (no restart, no downtime)

**The server itself never reboots unless you explicitly set `reboot: true`.**

### What if an agent run fails?

- The agent logs the error
- Stops applying that catalog at the point of failure
- Resources applied before the error remain
- On the next run (30 min), it tries again from the full catalog
- The server keeps running normally — a failed puppet run never crashes anything

---

# Part 5 — Where This Stops Being a Cluster and Starts Being a Platform

## Two Operators, One Philosophy

```
LinuxAid                        Juju
Owns: the OS                    Owns: the apps
Tool: OpenVox (Puppet)          Tool: Charmed Operators
Apply: agent, every 30 min      Apply: charm, continuously
       ◄──── same philosophy ────►
   Declare desired state. Operator converges. You sleep.
```

They meet at exactly one point: **Prometheus.**

LinuxAid puts `node_exporter` on every host.
Juju's COS Lite Prometheus scrapes it.

Host-to-application observability. Zero manual ServiceMonitors.
Zero manual `prometheus.yml` entries.

---

## The Full Stack — One Diagram

```
Git (LinuxAid config repo)
├── data/global.yaml           → SSH policy, firewall, exporters for ALL nodes
└── agents/node-1.demo.yaml   → role::basic for node-1
          │
          │ r10k deploys to OpenVox on push
          ▼
    OpenVox Server
    compiles catalog for each node
          │
          │ agent pull every 30 min
    ┌─────┴──────┐
    ▼            ▼
Node 1         Node 2
──────         ──────
SSH hardened   SSH hardened
Firewall up    Firewall up
node_exporter  node_exporter   ← :9100 scraped by COS Lite Prometheus
systemd_exp    systemd_exp
Juju agent     Juju agent
MicroK8s CP    MicroK8s Worker
    │            │
    └─────┬──────┘
          ▼
    Juju Controller
    ├── model: k8s
    │   └── MicroK8s (cluster lifecycle)
    │
    ├── model: monitoring (COS Lite)
    │   ├── Prometheus  ◄── scrapes :9100 on both nodes
    │   ├── Grafana     ◄── datasource auto-wired via relation
    │   ├── Alertmanager
    │   └── Loki
    │
    └── model: apps
        ├── PostgreSQL ◄──── related ────┐
        └── my-app ──────────────────────┘  (no manual secrets)
```

**Two Git repos. Zero manual wiring. One Grafana showing everything.**

---

## Who Owns What — Clean Boundaries

| Concern | Owner | Why the boundary is here |
|---------|-------|--------------------------|
| SSH policy, firewall, OS packages | LinuxAid | OS-level — Juju doesn't touch the host OS |
| Prometheus exporters on the host | LinuxAid | Must run on bare metal, not in a container |
| OS compliance (CIS/GDPR/NIS2) | LinuxAid | Applied at OS level, reconciled every 30 min |
| Kubernetes cluster lifecycle | Juju (MicroK8s charm) | Cluster bootstrap and upgrades = charm's job |
| Stateful application deployment | Juju (Charms) | Install, scale, upgrade = charm's job |
| Cross-app credentials | Juju (Relations) | No Secrets written or copied by hand |
| Observability stack | Juju (COS Lite) | Prometheus/Grafana as managed operators |
| Host metrics in Prometheus | LinuxAid → Juju | The integration point — exporter on host, scrape from cluster |

**Every layer has one owner. No layer is unowned. No tool overreaches.**

---

## Day One of a New Node — Full Timeline

```
T+0      Node boots

T+1min   linuxaid-install runs:
         - installs OpenVox agent
         - registers with OpenVox server (cert request)

T+5min   First catalog applied:
         - SSH hardened
         - Firewall configured
         - node_exporter running on :9100
         - systemd_exporter running
         - process_exporter running
         - Juju agent installed (if configured in global.yaml)

T+6min   Juju agent registers with Juju controller
         - Node joins MicroK8s cluster automatically

T+7min   COS Lite Prometheus discovers new scrape target
         - Metrics flow into Prometheus
         - Grafana dashboards show the new node

T+30min  Second puppet catalog run
         - Confirms everything is correct (idempotent)
         - Only applies changes if something drifted
```

**Total human input: one YAML line + three Juju commands.**

---

# Part 6 — Before vs After

## Scenario 1: Onboarding a Node

**Old way:**
```
Day 1   Run Ansible for SSH hardening
Day 1   Run another Ansible role for exporters
Day 1   Hand-edit prometheus.yml, reload, hope
Day 2   Realize compliance was skipped, run another playbook
Day 7   Node drifts. Silently. Forever.
Day 30  Audit asks for evidence — you grep bash history like a caveman
```

**New way:**
```
Day 1   One line in agents/node-1.yaml:  classes: [role::basic]
        → SSH + firewall + 4 exporters + CIS, done, metrics live in Grafana
Day 30  Audit: git log — every change, every author, every timestamp
```

---

## Scenario 2: Wiring PostgreSQL to Your App

**Without Juju:** 7 manual steps: copy password from a Secret, create another Secret,
edit Deployment with env vars, redeploy, then redo this for every environment.
And again every time PostgreSQL upgrades.

**With Juju:**
```bash
juju deploy postgresql
juju deploy my-app
juju relate my-app postgresql
```
Upgrades rotate credentials automatically.
Staging is identical to prod because it uses the same `juju relate`.

---

## Scenario 3: Debugging at 3 AM

**Without LinuxAid:**
Alert fires. SSH in. systemd looks "fine."
`node_exporter` has been dead for 6 hours — nobody noticed.
Your alert was firing on stale data the entire time.
Root cause: unknown. Sleep: also unknown.

**With LinuxAid:**
Alert fires. Open Grafana.
`nginx.service` shows failed at 02:47 (from `systemd_exporter`).
`node_exporter` restarted itself at 02:47 (LinuxAid agent reconciled it).
Root cause visible without touching SSH.

**The exporter didn't vanish quietly. LinuxAid caught and fixed it before you woke up.**

---

## Scenario 4: Scaling 5 Nodes to 50

**Old way:**
45 new VMs → 45× Ansible hardening → 45× exporter install
→ hand-edit `prometheus.yml` → reload → pray nothing was missed.

**New way:**
```
Provision 45 VMs → linuxaid-install runs on each
Add 45 lines to agents/ in Git → push → r10k → all nodes converge
juju add-unit microk8s × N
Prometheus auto-discovers new scrape targets
```

**Git is truth. Nodes converge to it. Scale is a commit, not a campaign.**

---

# Part 7 — Key Takeaways

1. **Kubernetes schedules containers. That's it.** Everything else is on you.
2. **Juju owns the app layer.** Relations kill manual secrets. Charms kill manual day-2.
3. **Juju stops at the OS.** SSH, firewall, exporters, compliance — nobody covers that by default.
4. **LinuxAid covers it — same philosophy, different layer.** Git → reconcile → repeat, forever.
5. **They meet at Prometheus.** OS + app, both self-healing, both feeding one dashboard.
   That is not a cluster. That is a platform.

---

# Juju vs Helm vs Ansible — Full Comparison

## What Each Tool Is For

| Tool | Designed for | Model |
|------|-------------|-------|
| Ansible | Execute tasks on remote machines | Push-based, runs on demand |
| Helm | Package and install Kubernetes applications | Package manager, runs on install/upgrade |
| Juju | Operate the full lifecycle of applications | Continuous operator, always watching |

These are not competing. They operate at different depths.

## Full Lifecycle Comparison

| Stage | Ansible | Helm | Juju |
|-------|---------|------|------|
| Install | ✅ playbook | ✅ `helm install` | ✅ `juju deploy` |
| Configure | ✅ vars | ✅ values.yaml | ✅ `juju config` |
| Connect to another app | ❌ manual | ❌ manual | ✅ `juju relate` |
| Scale | ❌ manual | Partial (replicas) | ✅ `juju add-unit` |
| Upgrade safely | ❌ manual | ❌ you own values drift | ✅ charm handles it |
| Rotate credentials | ❌ manual | ❌ manual | ✅ relation update |
| Backup | ❌ write it yourself | ❌ write it yourself | ✅ `juju run pg/0 create-backup` |
| Continuous reconciliation | ❌ runs on demand | ❌ | ✅ controller watches |
| Dependency tracking | ❌ | ❌ | ✅ relation graph |

## The PostgreSQL Example Across All Three

**Ansible:**
```yaml
- name: install postgresql
  package: name=postgresql state=present
- name: create user
  community.postgresql.postgresql_user:
    name: myapp password: "{{ vault_password }}"
# Then: manually pass that password to your app somehow
# On upgrade: re-run playbook and hope nothing breaks
```

**Helm:**
```bash
helm install pg bitnami/postgresql --set auth.password=mysecretpassword
# Manually extract password:
kubectl get secret pg-postgresql -o jsonpath='{.data.postgres-password}' | base64 -d
# Manually create Secret for your app
# Manually add env vars to app Deployment
# Manually redo all of this for staging
```

**Juju:**
```bash
juju deploy postgresql
juju deploy my-app
juju relate my-app postgresql
# Done. Credentials exist inside the relation. You never see them.
# Upgrades keep wiring intact. Staging uses same command.
```

---

# LinuxAid vs Ansible vs Cloud-Init — Full Comparison

## What Each Tool Is For

| Tool | Model | Runs |
|------|-------|------|
| cloud-init | Run-once bootstrap at VM creation | Once, at first boot |
| Ansible | Push-based task runner | When you decide to run it |
| LinuxAid | Pull-based, continuous reconciliation | Every 30 minutes, forever |

## The Drift Problem

This is the fundamental difference between these tools.

**cloud-init:** Runs at first boot. After that, the server is on its own.
Someone adds a firewall rule manually? It stays.
SSH config changes? It stays.
A month later, your servers are all slightly different from each other — nobody knows why.
This is **configuration drift**.

**Ansible:** Runs when you run it.
If you run it daily, drift lasts up to 24 hours.
If you forget to run it after a change, drift persists forever.
Also push-based: if your laptop is off, nothing gets applied.

**LinuxAid:** Pull-based and continuous.
Every node pulls its config every 30 minutes.
- Someone manually changes SSH config? Next run: restored.
- Exporter crashes and stops? Next run: restarted.
- New node joins? First run: fully configured in minutes.
- You update Git repo? Every node picks it up within 30 minutes.

## Full Comparison Table

| Capability | cloud-init | Ansible | LinuxAid |
|-----------|-----------|---------|---------|
| Initial node setup | ✅ | ✅ | ✅ |
| Continuous reconciliation | ❌ | ❌ | ✅ every 30 min |
| Detects and fixes drift | ❌ | ❌ | ✅ |
| Config in Git | Partial | ✅ playbooks | ✅ hiera YAML |
| Role-based config | ❌ | Partial | ✅ role::basic, role::db, etc. |
| Per-node overrides | Partial | ✅ host_vars | ✅ agents/certname.yaml |
| Group-based config | ❌ | ✅ inventory groups | ✅ tags in hiera |
| Prometheus exporters built-in | ❌ | ❌ write yourself | ✅ 11+ exporters |
| Compliance profiles | ❌ | ❌ write yourself | ✅ CIS, GDPR, NIS2 |
| Preview changes before apply | ❌ | `--check` (unreliable) | ✅ changeset calculation |
| Works when your laptop is off | ✅ | ❌ | ✅ |
| Audit trail | ❌ | ❌ | ✅ Git history |
| Scale to 1000+ nodes | ❌ | Partial | ✅ proven at 20,000+ |

## When to Use Each

**Use cloud-init when:** One-time bootstrap at VM creation — install one package, set hostname.

**Use Ansible when:** One-off tasks across many servers (migration, emergency patch).

**Use LinuxAid when:** You want continuous OS state enforcement, built-in monitoring,
compliance profiles, and the OS to follow the same GitOps workflow as your applications.

> **They are not mutually exclusive.**
> Common pattern: cloud-init for bootstrap → LinuxAid takes over from the first agent run.

---

# Glossary — Every Term Explained

| Term | What it means |
|------|--------------|
| **Catalog** | Compiled list of resources a node should have — files, packages, services, users |
| **certname** | Unique identity of a node: `hostname.customerid` e.g. `node-1.demo` |
| **Charm** | Package with application + code for full lifecycle (install, relate, scale, upgrade, backup) |
| **Charmhub** | Marketplace for Charms — like Docker Hub but for operators |
| **COS Lite** | Canonical Observability Stack Lite — Prometheus + Grafana + Alertmanager + Loki + Traefik as charms |
| **Cross-model relation** | Juju relation between applications in different models |
| **Drift** | When a server's actual state diverges from its declared intended state |
| **ENC** | External Node Classifier — tells OpenVox server what parameters apply to each node |
| **Hiera** | LinuxAid's hierarchical config lookup — resolves which value applies to which node |
| **Idempotent** | Running the same operation twice produces the same result — safe to repeat |
| **Juju Controller** | The Juju brain — manages all models, agents, and operations |
| **Juju Model** | Namespace for a group of related applications in Juju |
| **Juju Relation** | Mechanism by which two applications automatically exchange credentials and config |
| **LinuxAid** | Open-source OS management platform — manages Linux nodes continuously via OpenVox |
| **MicroK8s** | Canonical's lightweight Kubernetes distribution |
| **noop** | Dry-run mode — calculates what would change without applying it |
| **OpenVox** | Community fork of Puppet — the config management engine LinuxAid is built on |
| **Puppet** | Configuration management system — declare desired state, agents converge toward it |
| **r10k** | Tool that deploys Puppet code from Git onto the OpenVox server |
| **Reconciliation** | Compare desired state to current state, apply changes to close the gap |
| **role::basic** | LinuxAid role: SSH hardening + firewall + updates + all Prometheus exporters |
| **Unit** | One instance of a deployed Juju application (like a Kubernetes pod) |

---

# References

| Resource | URL |
|----------|-----|
| Juju docs | https://juju.is/docs |
| Charmhub | https://charmhub.io |
| MicroK8s Charm | https://charmhub.io/microk8s |
| COS Lite | https://charmhub.io/cos-lite |
| Charmed PostgreSQL | https://charmhub.io/postgresql |
| OpenVox | https://voxpupuli.org/openvox |
| LinuxAid | https://github.com/Obmondo/LinuxAid |
| Demo repo | https://github.com/MAVRICK-1/juju-linuxaid-demo |
