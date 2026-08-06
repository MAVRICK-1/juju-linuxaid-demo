# Deep Dive: Juju + LinuxAid — Everything Explained From Zero

> **Assume you know nothing.** This document explains every concept, every tool,
> every decision — with analogies, comparisons, and technical detail.
> Read this before looking at the talk slides.

---

## Table of Contents

1. [What Problem Are We Actually Solving?](#1-what-problem-are-we-actually-solving)
2. [Juju — Complete Explanation](#2-juju--complete-explanation)
3. [LinuxAid — Complete Explanation](#3-linuxaid--complete-explanation)
4. [Does LinuxAid Reboot Your Server?](#4-does-linuxaid-reboot-your-server)
5. [How the Two Tools Work Together](#5-how-the-two-tools-work-together)
6. [Juju vs Helm vs Ansible — Full Comparison](#6-juju-vs-helm-vs-ansible--full-comparison)
7. [LinuxAid vs Ansible vs Cloud-Init — Full Comparison](#7-linuxaid-vs-ansible-vs-cloud-init--full-comparison)
8. [Glossary — Every Term Explained](#8-glossary--every-term-explained)

---

## 1. What Problem Are We Actually Solving?

### The VS Code analogy

Imagine VS Code gives you a text editor.
That's genuinely useful. But you still need to install:
- A language extension (Python, Go, TypeScript)
- A linter
- A formatter
- A debugger
- A Git extension
- A terminal

VS Code is not an IDE out of the box. It's the foundation for one.

**Kubernetes is exactly like this.**

Kubernetes gives you a way to run containers.
That's genuinely useful. But you still need:
- A hardened, patched operating system on every node
- Something to deploy your database with proper backups
- Something to wire your app to that database automatically
- Something to scrape metrics from every node and service
- Something to apply firewall rules continuously

Kubernetes is not a platform out of the box. It's the foundation for one.

This talk is about the two tools that complete the platform:
- **Juju** — handles applications and their relationships (the VS Code extensions)
- **LinuxAid** — handles the OS on every node (the underlying OS that VS Code runs on)

---

## 2. Juju — Complete Explanation

### 2.1 What is Juju, really?

Juju is an **application lifecycle operator** made by Canonical (the Ubuntu company).

The simplest way to understand it:

> Kubernetes manages containers.
> Juju manages the applications that run inside (or alongside) Kubernetes.

But Juju doesn't just *install* applications. It operates them through their **entire life**:
install → configure → connect to other apps → scale up/down → upgrade → run backups → retire.

### 2.2 What is a Charmed Operator?

A **Charm** is a package that contains:
1. The application itself (or instructions to download it)
2. Code that knows *how* to install it correctly
3. Code that knows *how* to configure it
4. Code that knows *how* to connect it to other applications
5. Code that knows *how* to upgrade it safely
6. Code that knows *how* to run backups

Think of a Charm like a very smart Docker image — except instead of just running a container,
it also knows how to manage the application's entire operational life.

**Analogy:** A Helm chart is a recipe. A Charm is a chef.
The recipe tells you what ingredients and steps to follow. Once.
The chef knows the recipe, watches the dish while it cooks, adjusts seasoning,
handles substitutions, and knows when something is going wrong.

### 2.3 The Juju Controller

When you set up Juju, the first thing you create is a **Controller**.

The Controller is Juju's brain. It's a small set of VMs or containers that:
- Track the state of every application you've deployed
- Send instructions to every Juju agent
- Store the history of every operation
- Manage relations between applications

You only set up the Controller once per cloud or cluster.
Everything else is managed through it.

```
You (CLI/UI)
    │
    ▼
Juju Controller  ←── the brain, always running
    │
    ├── Model: production
    │     ├── postgresql  ← Juju agent on each unit
    │     └── my-app      ← Juju agent on each unit
    │
    └── Model: monitoring
          ├── prometheus
          └── grafana
```

### 2.4 What is a Juju Model?

A **Model** is a logical namespace for a group of applications that work together.

Think of it like a Kubernetes namespace — except it also tracks relations between apps,
manages configuration, and gives you a unified status view of everything in it.

You might have:
- `model: production` — your app + database
- `model: monitoring` — Prometheus + Grafana + Loki
- `model: staging` — exact copy of production for testing

Applications in the same model can be related to each other directly.
Applications in different models can be related via **cross-model relations**.

### 2.5 The Relation — Juju's Most Important Concept

This is the thing that makes Juju genuinely different from Helm or plain Kubernetes.

**The problem without Juju:**

You deploy PostgreSQL. You want your app to connect to it.
Here's what you have to do manually:

```
1. kubectl get secret postgresql-password -o jsonpath='{.data.password}' | base64 -d
2. Copy that password
3. Create a new Secret: kubectl create secret generic my-app-db --from-literal=password=<paste>
4. Edit your app's Deployment to add env vars: DB_HOST, DB_USER, DB_PASSWORD
5. kubectl apply -f deployment.yaml
6. Next time PostgreSQL upgrades and rotates passwords: repeat steps 1-5
7. For staging environment: repeat everything from scratch
```

Every time PostgreSQL changes, you chase it manually.
Every new environment is a manual re-do.
If you forget a step, your app breaks silently.

**With Juju:**

```bash
juju relate my-app postgresql
```

That single command triggers:
1. PostgreSQL's charm creates a dedicated database called `my-app-db`
2. PostgreSQL's charm creates a user `my-app` with a strong random password
3. PostgreSQL's charm passes `host`, `port`, `dbname`, `user`, `password` to my-app's charm
4. my-app's charm receives these and configures the application automatically
5. On PostgreSQL upgrade: charm sends updated credentials → my-app reconfigures itself
6. On scaling: new units automatically get the correct connection info

**You never see the password. You never copy it. You never manage it.**

### 2.6 Juju Models vs Kubernetes Namespaces

| Feature | Kubernetes Namespace | Juju Model |
|---------|---------------------|------------|
| Isolates resources | ✅ | ✅ |
| Tracks app relationships | ❌ | ✅ |
| Manages credentials between apps | ❌ | ✅ (via relations) |
| Gives app-level status | ❌ | ✅ (`juju status`) |
| Manages lifecycle events | ❌ | ✅ (charms handle it) |
| Cross-model communication | ❌ (needs NetworkPolicy) | ✅ (cross-model relations) |

### 2.7 COS Lite — The Observability Stack as a Charm Bundle

**COS Lite** (Canonical Observability Stack Lite) is a pre-packaged bundle of 5 charms
that gives you a complete observability stack:

| Component | What it does | Your alternative |
|-----------|-------------|-----------------|
| Prometheus | Collects metrics, evaluates alert rules | Manually deploy + configure |
| Grafana | Dashboards | Manually deploy + add datasources by hand |
| Alertmanager | Routes alerts to Slack/email/PagerDuty | Manually deploy + configure receivers |
| Loki | Collects and queries logs | Manually deploy + configure log shippers |
| Traefik | Gives each component an HTTPS URL | Manually write Ingress for each |

**Why COS Lite is different:**

Without COS Lite, after deploying all 5 components, you still have to:
- Manually add Prometheus as a datasource in Grafana
- Manually add Loki as a datasource in Grafana
- Manually configure Prometheus to send alerts to Alertmanager
- Manually configure Prometheus scrape jobs
- Manually create Ingress for each service

With COS Lite, all of this is wired via **relations**:

```bash
juju deploy cos-lite --trust
juju relate cos-lite microk8s       # Prometheus auto-scrapes the K8s cluster
```

Every time you add a new application and relate it to COS Lite,
its dashboards and scrape config appear automatically.

### 2.8 MicroK8s as a Charm — What This Means

Normally, to set up a Kubernetes cluster you'd:
- Install MicroK8s on each node manually
- Configure the control plane
- Join worker nodes with a token
- Enable addons (DNS, storage, ingress) manually
- Upgrade: hope nothing breaks

With the MicroK8s charm:

```bash
juju deploy microk8s --channel 1.28/stable   # deploys K8s control plane
juju add-unit microk8s                        # adds a worker, joins automatically
juju config microk8s channel=1.30/stable      # upgrades the cluster
```

The charm handles token generation, node join, HA setup, and addon configuration.
Your entire cluster lifecycle is now expressed as Juju config — version-controlled in Git.

---

## 3. LinuxAid — Complete Explanation

### 3.1 What is LinuxAid?

LinuxAid is an **open-source OS management platform** that manages the
operating system on Linux servers — not the containers, not Kubernetes, the actual OS.

It is built on **OpenVox**, which is the community fork of Puppet
(Puppet is a configuration management tool that has existed since 2005).

**The core idea:**
You describe what you want the OS to look like (in YAML files stored in Git).
LinuxAid continuously reconciles every node toward that description — every 30 minutes.

### 3.2 The VS Code analogy for LinuxAid

Imagine VS Code settings sync.
You configure VS Code exactly how you want it (theme, fonts, extensions, keybindings).
You push that config to GitHub.
On every new machine, VS Code pulls that config and makes itself look exactly right.
If you accidentally change a setting, next sync it gets corrected back.

LinuxAid does exactly this — but for Linux servers.
Your config lives in Git. Every node continuously pulls and applies it.
If someone manually changes SSH config on a server, next agent run: corrected.
If a new server joins, first agent run: fully configured in minutes.

### 3.3 The Two Repositories

LinuxAid uses two separate Git repos:

**Repo 1: LinuxAid (the code)**
```
github.com/Obmondo/LinuxAid
```
This is the puppet code — the *logic* of how to configure things.
You don't edit this (unless you're contributing to LinuxAid itself).
It contains modules like:
- "how to harden SSH"
- "how to install and configure node_exporter"
- "how to set up unattended security updates"

**Repo 2: Your config repo (the data)**
```
github.com/MAVRICK-1/juju-linuxaid-demo   ← this is yours
```
This is the *data* — what you want each node to look like.
You only write YAML here. No code.

```
your-linuxaid-config/
├── data/
│   └── global.yaml        # applies to ALL nodes
└── agents/
    ├── node-1.demo.yaml   # applies ONLY to node-1
    └── node-2.demo.yaml   # applies ONLY to node-2
```

### 3.4 How the Agent Run Works — Step by Step

Every 30 minutes (by default), this happens on every node:

```
Step 1: puppet agent wakes up
Step 2: sends its "certname" (e.g. node-1.demo) to the OpenVox server
Step 3: OpenVox server reads your Git config repo
Step 4: OpenVox compiles a "catalog" — a complete list of resources
        that should exist on this node (files, packages, services, users...)
Step 5: catalog is sent back to the node
Step 6: puppet agent compares current state to desired state
Step 7: for each difference, puppet applies the change
Step 8: puppet reports what changed (or didn't)
```

**What is a "resource"?**
A resource is a single managed thing. Examples:
- A file at `/etc/ssh/sshd_config` with specific contents
- A package `node-exporter` that must be installed
- A service `node-exporter` that must be running and enabled
- A user `ubuntu` that must have a specific SSH key
- A cron job that runs at a specific time

**LinuxAid manages hundreds of resources per node** — but you only declare the *outcome*
in your YAML. The code in LinuxAid figures out how to achieve it.

### 3.5 Hiera — How Configuration is Layered

**Hiera** is the configuration lookup system LinuxAid uses.

The word "hiera" means "hierarchy" — it resolves which value applies to a given node
by checking a priority chain from most specific to least specific:

```
Priority 1 (most specific):   agents/node-1.demo.yaml       ← only node-1
Priority 2:                   agents/tags/web/*.yaml         ← all "web" nodes
Priority 3:                   facts/os.family=Debian.yaml    ← all Debian nodes
Priority 4 (least specific):  data/global.yaml               ← every single node
```

**Example:**

`data/global.yaml` sets `ssh_port: 22` for all nodes.
`agents/node-1.demo.yaml` sets `ssh_port: 2222` for node-1.

Result: node-1 uses port 2222, everything else uses port 22.

The node-specific value wins. More specific always beats less specific.
You never duplicate config — you only override where needed.

### 3.6 What `role::basic` Installs and Configures

When you assign `role::basic` to a node, LinuxAid manages:

**SSH hardening:**
- Disables root login (`PermitRootLogin no`)
- Disables password authentication (key-only)
- Sets idle session timeout
- Configures allowed algorithms

**Firewall:**
- Installs and enables iptables rules
- Rules are declared in your hiera YAML — not set manually on the server
- If someone manually changes a firewall rule, next agent run restores it

**Automatic security updates:**
- Installs `unattended-upgrades`
- Security patches apply automatically (on a schedule you control)
- **Reboot: `false` by default** (see next section for full detail)

**Prometheus exporters (all installed as packages from a managed repo):**

| Exporter | Port | What it exposes |
|----------|------|----------------|
| `node_exporter` | 9100 | CPU, memory, disk, network, load average, filesystem |
| `systemd_exporter` | 9558 | State of every systemd service (active/failed/inactive) |
| `process_exporter` | 9256 | Per-process CPU and memory usage |
| `iptables_exporter` | 9455 | Firewall rule hit counts |

Each exporter is:
- Installed from a GPG-signed package (not a random GitHub release)
- Running as a dedicated non-root system user
- Managed as a systemd service
- Configured via the hiera file you write

---

## 4. Does LinuxAid Reboot Your Server?

**Short answer: No, not by default. Reboots are opt-in and you control them.**

Here is the exact behavior (verified from source code):

### 4.1 Automatic security updates

The `common::system::updates` module manages unattended upgrades.

```yaml
# default values (from LinuxAid source)
common::system::updates::manage: false    # disabled by default
common::system::updates::enable: false    # disabled by default
common::system::updates::reboot: false    # NEVER reboots by default
```

To enable updates **without** reboots (most common setup):
```yaml
# data/global.yaml
common::system::updates::manage: true
common::system::updates::enable: true
common::system::updates::reboot: false   # default — no reboot
```

To enable updates **with** reboots (opt-in, e.g. for kernel patches):
```yaml
common::system::updates::reboot: true   # you explicitly choose this
```

### 4.2 What about service restarts?

When LinuxAid changes a configuration file (e.g. SSH config, an exporter config),
it does restart the **affected service** — but only that service.

Examples:
- SSH config changed → `sshd` restarts (your existing SSH session is NOT dropped)
- `node_exporter` config changed → `node_exporter` service restarts
- `iptables` rules changed → rules are reloaded (no restart, no downtime)

**The server itself never reboots** unless you explicitly set `reboot: true`
in the updates module.

### 4.3 What if the agent run fails?

If the puppet agent encounters an error during a catalog run:
- It logs the error
- It **stops applying** that catalog
- Resources that were successfully applied before the error remain
- On the next run (30 min later), it tries again
- The server keeps running normally — a failed puppet run never crashes a server

### 4.4 What about the first agent run on a new node?

The first run can take 2–5 minutes and installs several packages.
No reboot happens. Services start running as packages are installed.

---

## 5. How the Two Tools Work Together

### 5.1 The handshake point: Prometheus

LinuxAid and Juju don't directly communicate.
They share exactly one integration point: **Prometheus scrape targets**.

```
LinuxAid installs node_exporter on port 9100
                      ↓
COS Lite Prometheus scrapes http://<node-ip>:9100/metrics
                      ↓
Grafana shows the dashboard
```

That's it. No API call. No shared config file. No Juju-LinuxAid integration needed.

### 5.2 Separation of concerns — why it's clean

```
┌─────────────────────────────────────────────────────────────┐
│  LinuxAid layer                                             │
│  ─────────────────────────────────────────────────────────  │
│  Manages: SSH, firewall, packages, exporters, compliance    │
│  Scope:   The Linux OS on bare metal or VMs                 │
│  Does NOT know about: Kubernetes, Juju, containers          │
├─────────────────────────────────────────────────────────────┤
│  Juju layer                                                 │
│  ─────────────────────────────────────────────────────────  │
│  Manages: Kubernetes cluster, apps, relations, monitoring   │
│  Scope:   Applications running on/in the nodes              │
│  Does NOT know about: SSH config, firewall rules, exporters │
└─────────────────────────────────────────────────────────────┘
```

Neither tool interferes with the other.
LinuxAid doesn't know Juju exists.
Juju doesn't know LinuxAid exists.
They just happen to run on the same nodes and produce complementary results.

### 5.3 Timeline of a new node

```
T+0min    Node boots for the first time

T+1min    linuxaid-install runs:
          - installs puppet agent
          - registers with OpenVox server
          - first catalog pull begins

T+5min    First catalog applied:
          - SSH hardened
          - Firewall configured
          - node_exporter running on :9100
          - systemd_exporter running
          - process_exporter running
          - Juju agent installed (if in global.yaml)

T+6min    Juju agent registers with Juju controller
          - Node joins MicroK8s cluster

T+7min    COS Lite Prometheus discovers new scrape target
          - Metrics start flowing into Prometheus
          - Grafana dashboards show the new node

T+30min   Second puppet catalog run confirms everything is correct
          (idempotent — only applies changes if something drifted)
```

---

## 6. Juju vs Helm vs Ansible — Full Comparison

### 6.1 What each tool is designed for

| Tool | Designed for | Level |
|------|-------------|-------|
| Ansible | Executing tasks on remote machines | Command runner |
| Helm | Packaging and installing Kubernetes applications | Package manager |
| Juju | Operating the full lifecycle of applications | Application operator |

These are not competing tools. They operate at different depths.
You can use all three. But for day-2 operations on Kubernetes, Juju goes deeper.

### 6.2 The lifecycle comparison

| Lifecycle stage | Ansible | Helm | Juju |
|----------------|---------|------|------|
| Install | ✅ (playbook) | ✅ (`helm install`) | ✅ (`juju deploy`) |
| Configure | ✅ (vars) | ✅ (values.yaml) | ✅ (`juju config`) |
| Connect to another app | ❌ manual | ❌ manual | ✅ (`juju relate`) |
| Scale up | ❌ manual | Partial (replicas) | ✅ (`juju add-unit`) |
| Upgrade with safety | ❌ manual | ❌ manual | ✅ (charm handles it) |
| Rotate credentials | ❌ manual | ❌ manual | ✅ (relation update) |
| Run a backup | ❌ write it yourself | ❌ write it yourself | ✅ (`juju run pg/0 create-backup`) |
| Continuous reconciliation | ❌ runs once | ❌ runs once | ✅ (controller watches) |
| Cross-app dependency tracking | ❌ | ❌ | ✅ (relation graph) |

### 6.3 The PostgreSQL example across all three tools

**Ansible:**
```yaml
# You write a playbook
- name: install postgresql
  package: name=postgresql state=present
- name: create database
  community.postgresql.postgresql_db: name=myapp
- name: create user
  community.postgresql.postgresql_user: name=myapp password={{ vault_password }}
# Now you manually pass the password to your app somehow
# On upgrade: re-run playbook, hope nothing breaks
```

**Helm:**
```bash
helm install pg bitnami/postgresql --set auth.password=mysecretpassword
# Now manually:
kubectl get secret pg-postgresql -o jsonpath='{.data.postgres-password}' | base64 -d
# Create a secret for your app manually
# Add env vars to your app's deployment manually
```

**Juju:**
```bash
juju deploy postgresql
juju deploy my-app
juju relate my-app postgresql
# Done. Credentials exist inside the relation.
# You never see the password.
```

---

## 7. LinuxAid vs Ansible vs Cloud-Init — Full Comparison

### 7.1 What each tool is designed for

| Tool | Model | Runs |
|------|-------|------|
| cloud-init | Run-once bootstrap on VM creation | Once, at first boot |
| Ansible | Push-based: you run it, it changes the server | When you run it |
| LinuxAid | Pull-based, continuous: node pulls config every 30 min | Forever, on a schedule |

### 7.2 The drift problem

This is the fundamental difference.

**cloud-init:** Runs at first boot. After that, the server is on its own.
Someone adds a firewall rule manually? It stays. SSH config changes? It stays.
A month later, your servers are all slightly different from each other.
Nobody knows why. This is called **configuration drift**.

**Ansible:** Runs when you run it. If you run it daily, drift lasts up to 24 hours.
If you forget to run it after a change, drift persists forever.
Ansible is also **push-based** — you push changes from your laptop.
If your laptop is off, nothing gets applied.

**LinuxAid:** Pull-based and continuous. Every node pulls its config every 30 minutes.
- Someone manually changes `/etc/ssh/sshd_config`? Next run: restored to declared state.
- A new node joins the cluster? First run: fully configured in minutes.
- You update your Git repo? Every node picks it up within 30 minutes.

### 7.3 Full comparison table

| Capability | cloud-init | Ansible | LinuxAid |
|-----------|-----------|---------|---------|
| Initial node setup | ✅ | ✅ | ✅ |
| Continuous reconciliation | ❌ | ❌ (runs on demand) | ✅ (every 30 min) |
| Detects and fixes drift | ❌ | ❌ | ✅ |
| Config in Git | Partial (user-data) | ✅ (playbooks) | ✅ (hiera data) |
| Role-based config | ❌ | Partial (roles) | ✅ (role::basic, role::db::mysql...) |
| Per-node overrides | Partial | Partial (host_vars) | ✅ (agents/certname.yaml) |
| Group-based config | ❌ | ✅ (inventory groups) | ✅ (tags) |
| Prometheus exporters built-in | ❌ | ❌ (write yourself) | ✅ (11+ exporters) |
| Compliance profiles | ❌ | ❌ (write yourself) | ✅ (CIS, GDPR, NIS2) |
| Preview changes before apply | ❌ | `--check` (unreliable) | ✅ (changeset calculation) |
| Works when your laptop is off | ✅ | ❌ | ✅ |
| Audit trail | ❌ | ❌ | ✅ (Git history) |
| Scale to 1000+ nodes | ❌ (too slow) | Partial | ✅ (proven at 20,000+) |

### 7.4 When to use each

**Use cloud-init** when:
- You need a one-time bootstrap script on VM creation (install one package, set hostname)
- You have no ongoing config management and don't need it

**Use Ansible** when:
- You need to run a one-off task across many servers (e.g. "upgrade this package today")
- You're doing a migration (run once, then it's done)
- You want human-initiated changes, not continuous reconciliation

**Use LinuxAid** when:
- You want continuous enforcement of OS state
- You want built-in monitoring exporters without extra work
- You want compliance (CIS/GDPR/NIS2) without writing your own playbooks
- You have more than a handful of servers
- You want the OS layer to follow the same GitOps workflow as your applications

> **They are not mutually exclusive.**
> Many teams use cloud-init for initial bootstrap, then LinuxAid takes over.

---

## 8. Glossary — Every Term Explained

| Term | What it means |
|------|--------------|
| **Charm** | A package containing an application + code for its full lifecycle (install, relate, scale, upgrade, backup) |
| **Charmed Operator** | Same as Charm |
| **Charmhub** | The marketplace for Charms — like Docker Hub but for Juju operators |
| **COS Lite** | Canonical Observability Stack Lite — a Juju bundle: Prometheus + Grafana + Alertmanager + Loki + Traefik |
| **Catalog** | In LinuxAid/Puppet: the compiled list of resources a node should have |
| **certname** | The unique identity of a node in LinuxAid, format: `hostname.customerid` (e.g. `node-1.demo`) |
| **Cross-model relation** | A Juju relation between applications in different models |
| **Drift** | When a server's actual state diverges from its declared/intended state |
| **ENC** | External Node Classifier — tells the Puppet server what parameters apply to each node |
| **Hiera** | LinuxAid's hierarchical config lookup system — resolves which value applies to which node |
| **Hiera datapath** | The path to your customer's config in the hiera data directory |
| **Idempotent** | Running the same operation twice produces the same result — no harm in repeating |
| **Juju Controller** | The Juju brain — manages all models, agents, and operations |
| **Juju Model** | A namespace for a group of related applications in Juju |
| **Juju Relation** | The mechanism by which two applications automatically exchange credentials and configuration |
| **LinuxAid** | Open-source OS management platform — manages Linux node configuration continuously |
| **MicroK8s** | Canonical's lightweight Kubernetes distribution — installable as a single package |
| **noop** | "No operation" — a dry-run mode where LinuxAid calculates what would change without applying it |
| **OpenVox** | Community fork of Puppet — the config management engine LinuxAid is built on |
| **Puppet** | Configuration management system — declares desired state, agents converge toward it |
| **r10k** | Tool that deploys Puppet code from Git onto the OpenVox server |
| **Reconciliation** | The process of comparing desired state to current state and applying changes to close the gap |
| **role::basic** | LinuxAid role that enables SSH hardening, firewall, updates, and all Prometheus exporters |
| **Sealed Secret** | A Kubernetes Secret encrypted with a cluster-specific key — safe to store in Git |
| **Unit** | In Juju, one instance of a deployed application (like a Kubernetes pod) |

---

## References

| Resource | URL |
|----------|-----|
| Juju documentation | https://juju.is/docs |
| Charmhub | https://charmhub.io |
| MicroK8s Charm | https://charmhub.io/microk8s |
| COS Lite bundle | https://charmhub.io/cos-lite |
| Charmed PostgreSQL | https://charmhub.io/postgresql |
| OpenVox | https://voxpupuli.org/openvox |
| LinuxAid source | https://github.com/Obmondo/LinuxAid |
| Demo config repo | https://github.com/MAVRICK-1/juju-linuxaid-demo |
