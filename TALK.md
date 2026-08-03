# "Kubernetes Is Not a Platform: Building One with Juju + LinuxAid"

> Talk brief, slide outline, and architecture reference

---

## The Problem (Slide 1 — Hook)

> "Everyone says Kubernetes IS the platform. It's not."

Kubernetes gives you:
- Container scheduling ✅
- Networking ✅
- Storage ✅

Kubernetes does NOT give you:
- OS hardening ❌
- Node lifecycle management ❌
- Application day-2 ops (upgrades, backups, scaling) ❌
- Observability wired up automatically ❌

**You still have to build the platform. This talk shows how.**

---

## The Stack (Slide 2 — Architecture Overview)

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR PLATFORM                         │
├─────────────────┬───────────────────────────────────────┤
│   LinuxAid      │  Juju                                 │
│   (OS Layer)    │  (Application + K8s Layer)            │
│                 │                                       │
│ • SSH hardening │ • MicroK8s (multi-node K8s)           │
│ • Firewall      │ • COS Lite (Prometheus + Grafana)     │
│ • NTP           │ • PostgreSQL with replication         │
│ • node_exporter │ • Kafka, OpenSearch                   │
│ • 11+ exporters │ • Day-2: upgrade, scale, relate       │
└─────────────────┴───────────────────────────────────────┘
         ↑                        ↑
   OpenVox Server           Juju Controller
   (puppet catalog)         (charm orchestrator)
```

---

## Layer 1 — LinuxAid: The OS Layer (Slides 3–5)

### What it is
LinuxAid is a GitOps-driven OS management platform built on OpenVox (open-source Puppet-compatible). It manages the **machine**, not the containers.

### What it does on every node

```
linuxaid-install --certname node-1.demo --puppet-server <openvox-ip>:8140
```

One command. Then puppet applies:

| Feature | What it does |
|---------|-------------|
| SSH hardening | Disables root login, enforces key auth, sets idle timeout |
| Firewall | iptables rules via Hiera config — declarative, GitOps |
| NTP | chrony configured and synced |
| node_exporter | CPU, RAM, disk, network metrics on `:9100` |
| iptables_exporter | Firewall rule metrics |
| systemd_exporter | Service health metrics |
| process_exporter | Per-process resource usage |

### GitOps config structure

```
MAVRICK-1/juju-linuxaid-demo/     ← your config repo
├── data/
│   └── global.yaml               ← shared config ALL nodes
└── agents/
    ├── node-1.demo.yaml          ← node-1 specific role
    └── node-2.demo.yaml          ← node-2 specific role
```

**global.yaml** — enables exporters on all nodes:
```yaml
---
monitor::enable: true
common::monitor::exporter::node::enable: true
common::monitor::exporter::iptables::enable: true
common::monitor::exporter::systemd::enable: true
```

**agents/node-1.demo.yaml** — assigns role to a node:
```yaml
---
classes:
  - role::basic
```

> Push to GitHub → r10k deploys → puppet applies in 30 min. No SSH required.

---

## Layer 2 — Juju: The Application Layer (Slides 6–9)

### What it is
Juju is a model-driven application operator. It deploys **Charmed Operators** — software that knows how to install, configure, scale, upgrade, and relate applications.

### What makes it different from Helm

| | Helm | Juju |
|--|------|------|
| Install | `helm install` | `juju deploy` |
| Upgrade | `helm upgrade` (manual values) | `juju upgrade-charm` |
| Connect apps | Manual env vars / secrets | `juju relate app1 app2` |
| Scale | Manual replica count | `juju add-unit` |
| Day-2 ops | You write it | Built into the charm |
| Backup | You configure it | `juju run postgresql/0 create-backup` |

### Deploy the full stack

```bash
# Step 1 — Kubernetes cluster
juju add-model k8s-demo
juju deploy microk8s --channel 1.28/stable
juju add-unit microk8s          # add worker node

# Step 2 — Observability (auto-wires to MicroK8s)
juju add-model cos
juju deploy cos-lite --trust    # Prometheus + Grafana + AlertManager + Loki
juju relate cos-lite microk8s   # auto-configures scraping

# Step 3 — Add LinuxAid node metrics to Prometheus
juju config prometheus scrape-jobs="
- job_name: linuxaid-nodes
  static_configs:
    - targets: ['<node1-ip>:9100', '<node2-ip>:9100']
"

# Step 4 — Deploy apps with automatic connections
juju deploy postgresql --channel 14/stable
juju deploy my-app
juju relate my-app postgresql   # automatic connection — no manual config
```

### The "relate" magic — demo moment

```bash
# Before relate: my-app has no database
juju status
# App    Status   Message
# my-app waiting  waiting for database relation

juju relate my-app postgresql

# After relate: Juju automatically:
# - Creates database user
# - Passes credentials to my-app via relation data
# - Restarts my-app with correct DB_URL
juju status
# App    Status  Message
# my-app active  ready
```

---

## Full Architecture Diagram (Slide 10)

```
┌─────────────────────────────────────────────────────────────────┐
│  GitHub: MAVRICK-1/juju-linuxaid-demo                           │
│  (hiera config — agents/*.yaml, data/global.yaml)               │
└───────────────────────┬─────────────────────────────────────────┘
                        │ git pull (r10k)
                        ▼
            ┌───────────────────────┐
            │  Hetzner CX22         │
            │  OpenVox Server       │
            │  (puppet catalog)     │
            │  port 8140            │
            └──────────┬────────────┘
                       │ puppet agent (30 min interval)
          ┌────────────┴────────────┐
          ▼                         ▼
┌─────────────────┐       ┌─────────────────┐
│ Hetzner CX32    │       │ Hetzner CX22    │
│ node-1.demo     │       │ node-2.demo     │
│                 │       │                 │
│ LinuxAid:       │       │ LinuxAid:       │
│ • SSH hardened  │       │ • SSH hardened  │
│ • Firewall      │       │ • Firewall      │
│ • node_exporter │       │ • node_exporter │
│   :9100         │       │   :9100         │
│                 │       │                 │
│ Juju agent      │       │ Juju agent      │
│ MicroK8s CP     │       │ MicroK8s Worker │
└────────┬────────┘       └────────┬────────┘
         └──────────┬──────────────┘
                    ▼
         ┌─────────────────────┐
         │   Juju Controller   │
         │                     │
         │  model: k8s-demo    │
         │  └── MicroK8s       │
         │                     │
         │  model: cos         │
         │  ├── Prometheus ────┼──── scrapes :9100 on both nodes
         │  ├── Grafana        │
         │  ├── AlertManager   │
         │  └── Loki           │
         │                     │
         │  model: apps        │
         │  ├── PostgreSQL     │
         │  ├── Kafka          │
         │  └── my-app ────────┼──── auto-related to PostgreSQL
         └─────────────────────┘
```

---

## Demo Flow (Slide 11 — Live Demo Script)

```
1. Show Hetzner console — 3 nodes running
2. SSH into node-1 — run: curl localhost:9100/metrics | grep node_cpu
   → "LinuxAid installed node_exporter — metrics flowing"

3. Show Grafana — node dashboards showing CPU/RAM/disk
   → "Juju's COS Lite scraped those metrics automatically"

4. juju status — show all applications and their relations

5. juju relate my-app postgresql
   → "Watch my-app go from waiting → active in 30 seconds"
   → "No manual secret passing. No env var config. Just relate."

6. juju add-unit postgresql
   → "Scaling with one command"

7. git push to config repo → 30 min later puppet applies
   → "GitOps for OS layer — same principle as Kubernetes"
```

---

## Key Takeaways (Slide 12)

1. **Kubernetes is not a platform** — it's infrastructure for a platform
2. **LinuxAid** gives you a managed, observable, secure OS layer — GitOps style
3. **Juju** gives you application lifecycle management that Helm can't match
4. **Together**: every layer is managed, observable, and GitOps-driven
5. **One operator pattern** from OS → K8s → Apps

> "You don't just deploy software. You operate it."

---

## Talk Metadata

| Field | Value |
|-------|-------|
| Title | Kubernetes Is Not a Platform: Building One with Juju + LinuxAid |
| Duration | 25–30 min |
| Level | Intermediate |
| Demo | Live (Hetzner + EC2) |
| Tags | Kubernetes, Juju, LinuxAid, Prometheus, GitOps, Platform Engineering |
| Speaker | Rishi Mondal — SRE at Obmondo, CNCF KubeStellar Maintainer, Docker Captain |

---

## References

| Resource | URL |
|----------|-----|
| LinuxAid | https://github.com/Obmondo/LinuxAid |
| LinuxAid CLI | https://github.com/Obmondo/linuxaid-cli |
| OpenVox | https://voxpupuli.org/openvox |
| Juju | https://juju.is/docs |
| MicroK8s Charm | https://charmhub.io/microk8s |
| COS Lite | https://charmhub.io/cos-lite |
| Charmed PostgreSQL | https://charmhub.io/postgresql |
| Demo Repo | https://github.com/MAVRICK-1/juju-linuxaid-demo |
