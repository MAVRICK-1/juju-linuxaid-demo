# Kubernetes Is Not a Platform: Building One with Juju + LinuxAid

**Presentation Overview**
Speaker: Rishi Mondal — SRE at Obmondo | CNCF KubeStellar Maintainer | Docker Captain

---

## Part 1 — The Problem

### Slide 1 — Kubernetes Is Infrastructure, Not a Platform

You provision a Kubernetes cluster.
Now what?

You still don't have:

| Missing piece | Why it matters |
|---------------|---------------|
| Hardened, observable OS on every node | Metrics, compliance, and security start at the host |
| Day-2 ops for stateful applications | Upgrades, backups, credential rotation — all manual |
| Wiring between applications | Every secret passed by hand, every endpoint hardcoded |
| A coherent observability stack | You assemble Prometheus, Grafana, Loki yourself — and rewire them yourself |

Kubernetes gives you the floor.
You have to build the building.

---

### Slide 2 — How Teams Usually Fill the Gap

```
OS hardening       →  Ansible playbook (runs once, then drifts)
Node exporters     →  Another Ansible role (maybe forgotten)
Kubernetes         →  Cloud-managed or kubeadm (manual upgrades)
Application deploy →  Helm (install only, day-2 is yours)
App connections    →  Environment variables + Secrets (managed forever)
Observability      →  Manually wired Prometheus + Grafana
```

Each layer is solved in isolation.
Nothing talks to anything else.
Nobody owns the full picture.

The result: a platform that works until it doesn't,
and when it doesn't, nobody knows which layer broke.

---

## Part 2 — Juju: Operator for Everything Above the OS

### Slide 3 — What Juju Is

Juju is a **model-driven application operator** by Canonical.

It deploys **Charmed Operators** — software that encodes the **complete lifecycle**
of an application, not just its installation:

```
Install  →  Configure  →  Relate  →  Scale  →  Upgrade  →  Backup
```

All of this is expressed as a **model** — a declared state that Juju continuously
reconciles, exactly like Kubernetes reconciles pod specs.

400+ Charmed Operators on [Charmhub](https://charmhub.io):
PostgreSQL, Kafka, OpenSearch, Vault, Grafana, and more.

---

### Slide 4 — The Relation: Juju's Core Differentiator

**The problem with Helm:**
Deploy PostgreSQL + deploy your app + manually create a Secret +
manually inject it as an env var + maintain it through every upgrade.

**With Juju:**

```
juju relate my-app postgresql
```

Juju automatically:
1. Tells PostgreSQL to create a dedicated user and database
2. Passes credentials to `my-app` through the relation interface
3. Keeps this wiring intact through restarts, upgrades, and scaling

No Secrets written by hand. No env vars configured manually.
**The relation is the contract. The charm honours it.**

---

### Slide 5 — Juju vs Helm: The Day-2 Gap

| Capability | Helm | Juju |
|-----------|------|------|
| Install | ✅ | ✅ |
| Upgrade | `helm upgrade` — you own values drift | `juju upgrade-charm` — charm owns it |
| Scale | Edit replicas manually | `juju add-unit` |
| Connect two apps | Manual secrets + env vars | `juju relate app-a app-b` |
| Credential rotation | Edit Secret → redeploy → restart | Charm handles via relation update |
| Run a backup | Write a CronJob yourself | `juju run postgresql/0 create-backup` |
| Status visibility | `kubectl get pods` — infra view | `juju status` — application-aware view |

Helm is a package manager.
Juju is an operator framework.
They are not competing — they operate at different depths.

---

### Slide 6 — COS Lite: The Observability Charm Bundle

Deploy a complete observability stack in one command:

```
juju deploy cos-lite --trust
juju relate cos-lite microk8s     ← auto-wires cluster scraping
```

| Component | Role |
|-----------|------|
| Prometheus | Metrics + alert evaluation |
| Grafana | Dashboards — datasources wired via relations |
| Alertmanager | Alert routing + deduplication |
| Loki | Log aggregation |
| Traefik | Ingress for all components |

**Every component is a charm.**
Upgrades are coordinated. Datasources are wired automatically.
Add a new application, relate it to COS Lite — its dashboards appear.

No manual `prometheus.yml`. No manual Grafana datasource config.

---

### Slide 7 — MicroK8s as a Juju Charm

Juju can also manage **Kubernetes itself** as a charmed operator:

```
juju deploy microk8s --channel 1.28/stable   # control plane node
juju add-unit microk8s                        # add worker
juju add-unit microk8s                        # add another worker
```

The MicroK8s charm handles:
- Multi-node bootstrap and join token management
- High availability control plane
- Channel-based upgrades (`juju config microk8s channel=1.30/stable`)
- Addons (DNS, storage, ingress) as charm config

The entire Kubernetes lifecycle is now a Juju model:
versioned in Git, reproducible on any cloud, upgraded with one command.

---

## Part 3 — The Gap Juju Leaves Behind

### Slide 8 — What Juju Doesn't Own

Juju manages Kubernetes clusters and applications running on them.

It does not manage the **operating system** on the nodes.

Every node underneath your cluster still needs:

```
SSH hardening          →  who can log in, how
Firewall               →  what traffic is allowed at kernel level
Automatic OS updates   →  security patches without human intervention
Prometheus exporters   →  CPU, memory, disk, systemd health on the HOST
Compliance posture     →  CIS benchmarks, GDPR, NIS2
```

If Juju is the top half of the stack, something needs to own the bottom half.

You could write Ansible. You could use cloud-init.
The problem: those tools run **once**. They don't continuously reconcile.
A node that drifts from its desired state stays drifted — silently.

---

## Part 4 — LinuxAid: The Bottom Half of the Stack

### Slide 9 — Continuous OS Reconciliation

LinuxAid is a Puppet-based OS management tool (built on OpenVox) that applies
the **same operator philosophy** Juju uses for applications — to the OS:

> Declare desired state → operator reconciles toward it continuously.

Every 30 minutes, every node pulls its catalog and applies it.
SSH policy drifted? Fixed. Exporter crashed and stopped? Restarted.
New node joined? Bootstrap brings it to policy in one agent run.

**Config lives in Git:**

```
your-linuxaid-config/
├── data/
│   └── global.yaml          # applies to ALL nodes
└── agents/
    ├── node-1.demo.yaml     # role for node-1
    └── node-2.demo.yaml     # role for node-2
```

Push to Git → r10k deploys → nodes converge.
No SSH. No Ansible run. No manual step.

---

### Slide 10 — What `role::basic` Gives You

Assign one role to a node:

```yaml
# agents/node-1.demo.yaml
classes:
  - role::basic
```

That single line automatically installs and maintains:

| What | Why it matters |
|------|---------------|
| SSH hardened to policy | No root login, key auth only, idle timeout |
| Firewall configured | iptables rules declared in hiera, not set manually |
| Unattended security updates | Patches apply without human intervention |
| `node_exporter` on `:9100` | CPU, memory, disk, network, load — host metrics |
| `systemd_exporter` | Every service state — failed units visible in Prometheus |
| `process_exporter` | Per-process CPU and memory |
| `iptables_exporter` | Firewall rule hit counts |

Everything applied by the agent. Everything reconciled every 30 minutes.
Everything traceable to a Git commit.

---

## Part 5 — The Perfect Combination

### Slide 11 — Two Operators, One Philosophy, Zero Gaps

This is where the stack becomes a platform.

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   LinuxAid                        Juju                       │
│   ─────────────────               ────────────────────       │
│   Owns: the OS                    Owns: the applications     │
│   Tool: OpenVox (Puppet)          Tool: Charmed Operators    │
│   Model: Hiera declares state     Model: Juju model declares │
│   Apply: agent every 30 min       Apply: charm continuously  │
│                                                              │
│   ◄─────────── same philosophy ──────────────────────────►  │
│        Declare desired state. Operator converges.            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**They meet at exactly one point: Prometheus.**

LinuxAid puts `node_exporter` on every host.
Juju's COS Lite Prometheus scrapes it.

You get host-to-application observability with no manual scrape config,
no ServiceMonitor to write, no dashboard to build by hand.

---

### Slide 12 — The Full Stack Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  Git: your-linuxaid-config                                         │
│  Declares: node roles, firewall, exporters, compliance             │
└─────────────────────────────┬──────────────────────────────────────┘
                              │ r10k on push
                              ▼
                   ┌─────────────────────┐
                   │   OpenVox Server    │
                   │  compiles catalog   │
                   └──────────┬──────────┘
                              │ agent pull every 30 min
              ┌───────────────┴───────────────┐
              ▼                               ▼
       Linux Node 1                    Linux Node 2
       ─────────────                   ─────────────
       SSH hardened                    SSH hardened
       Firewall managed                Firewall managed
       node_exporter  :9100 ──────────►               ─────────────┐
       systemd_exporter      ──────────►               ─────────────┤
       process_exporter      ──────────►               ─────────────┤
                                                                     │
       Juju agent ──────────────────── Juju agent                   │
       MicroK8s CP                     MicroK8s Worker              │
              │                               │                      │ scrape
              └───────────────┬───────────────┘                      │
                              ▼                                       │
                   ┌──────────────────────────┐                      │
                   │     Juju Controller      │                      │
                   │                          │                      │
                   │  model: k8s              │                      │
                   │  └── MicroK8s            │                      │
                   │                          │                      │
                   │  model: monitoring       │                      │
                   │  ├── Prometheus  ◄───────┼──────────────────────┘
                   │  ├── Grafana             │  host OS metrics
                   │  ├── Alertmanager        │  + cluster metrics
                   │  └── Loki               │  in one stack
                   │                          │
                   │  model: apps             │
                   │  ├── PostgreSQL ◄──┐     │
                   │  └── my-app ───────┘     │  (related — no manual
                   │                          │   secret management)
                   └──────────────────────────┘
```

---

### Slide 13 — Layer Ownership: Clean Boundaries

The reason this works is that neither tool overreaches.

| Concern | Tool | Boundary |
|---------|------|---------|
| SSH policy, firewall, OS packages | LinuxAid | Host OS — Juju doesn't touch this |
| Prometheus exporters on the host | LinuxAid | Must run on bare metal, not in a pod |
| OS compliance (CIS / GDPR / NIS2) | LinuxAid | Applied at OS level, agent reconciles |
| Kubernetes cluster lifecycle | Juju | MicroK8s charm — LinuxAid doesn't touch this |
| Stateful application deployment | Juju | Charm handles install, scale, upgrade |
| Cross-application credentials | Juju | Relations — no Secret written by hand |
| Observability stack | Juju COS Lite | Prometheus + Grafana as charms |
| Host metrics in that Prometheus | LinuxAid → Juju | The integration point |

Every layer has one owner. No layer is unowned. No tool fights the other.

---

### Slide 14 — What Day One Looks Like

A brand new node enrolls in this stack:

**Step 1 — LinuxAid agent runs (first 30 minutes):**
- SSH hardened
- Firewall configured
- Security updates enabled
- `node_exporter`, `systemd_exporter`, `process_exporter` installed and running
- CIS compliance applied

**Step 2 — Juju agent registers the node:**
- Node joins MicroK8s cluster
- COS Lite Prometheus begins scraping `:9100`
- Host metrics appear in Grafana dashboards

**Step 3 — Application gets deployed:**
```
juju deploy postgresql
juju deploy my-app
juju relate my-app postgresql
```
- PostgreSQL running, credentials wired automatically
- `my-app` connected — zero manual config

**Total human input: one YAML file + three Juju commands.**

---

## Part 6 — Key Takeaways

### Slide 15 — What to Remember

1. **Kubernetes is not a platform.**
   It handles container scheduling. Every other layer is still yours to build.

2. **Juju solves the application layer.**
   Relations eliminate manual secret passing.
   Charms eliminate manual day-2 operations.
   COS Lite wires observability automatically.

3. **But Juju stops at the OS boundary.**
   Node exporters, SSH policy, firewall, compliance — these live on the host.
   Something needs to own them continuously, not just at bootstrap.

4. **LinuxAid fills that gap with the same operator philosophy.**
   Declare state in Git. Agent reconciles every 30 minutes.
   No drift. No forgotten nodes. No manual SSH.

5. **They meet at Prometheus — and that is the whole point.**
   When the OS layer and the application layer are both continuously managed
   and both feeding the same observability stack, you have a real platform.
   Not a cluster. Not a deployment. A platform.

---

### Slide 16 — References

| Resource | URL |
|----------|-----|
| Juju | https://juju.is |
| Charmhub | https://charmhub.io |
| MicroK8s Charm | https://charmhub.io/microk8s |
| COS Lite | https://charmhub.io/cos-lite |
| Charmed PostgreSQL | https://charmhub.io/postgresql |
| OpenVox | https://voxpupuli.org/openvox |
| LinuxAid | https://github.com/Obmondo/LinuxAid |
| Demo Repo | https://github.com/MAVRICK-1/juju-linuxaid-demo |

---

## Appendix

### Juju Relation Graph (COS Lite)

```
microk8s ──────────────── cos-lite
                            ├── prometheus
                            │     ├── grafana        (datasource relation)
                            │     └── alertmanager   (receiver relation)
                            └── loki
                                  └── grafana        (datasource relation)
```

### LinuxAid Hiera Hierarchy (Priority Order)

| Priority | Scope | Path |
|----------|-------|------|
| 1 — highest | Node-specific | `agents/<certname>.yaml` |
| 2 | Tag group | `agents/tags/<tag>/*.yaml` |
| 3 | OS/hardware facts | `facts/os.family=<value>.yaml` |
| 4 — lowest | Global default | `data/global.yaml` |

### role::basic Module Tree

```
role::basic
  └── common::init
        ├── common::monitor::exporter::node        :9100
        ├── common::monitor::exporter::systemd
        ├── common::monitor::exporter::process
        ├── common::monitor::exporter::iptables
        ├── common::network::firewall
        ├── common::packages (unattended-upgrades)
        └── common::users
```
