# Kubernetes Is Not a Platform: Building One with Juju + LinuxAid

**Presentation Overview**
Speaker: Rishi Mondal — SRE at Obmondo | CNCF KubeStellar Maintainer | Docker Captain

---

## Part 1 — The Problem

### Slide 1 — What Kubernetes Actually Gives You

When you provision a Kubernetes cluster, you get a container scheduler.
That is the complete list.

You do **not** get:
- A hardened operating system
- Compliance with GDPR, CIS, or NIS2
- Prometheus exporters on your nodes
- Automated application operators
- Day-2 lifecycle management for stateful services
- Any relation between your applications

Every organisation that ships Kubernetes to production discovers this the same way:
by manually stitching each missing layer together, one incident at a time.

This talk is about not doing that.

---

### Slide 2 — The Platform Tax

Building a real platform on top of Kubernetes requires answering at least four questions:

| Layer | Question | Who normally answers it |
|-------|----------|------------------------|
| OS | How do nodes stay hardened, patched, and observable? | Manual scripts / Ansible / hope |
| Kubernetes | How do clusters get created, upgraded, and operated? | Cloud control planes / KubeAid |
| Applications | How do stateful apps get deployed and connected? | Helm / manual config |
| Observability | How does everything get scraped, dashboarded, and alerted? | Hand-wired Prometheus stacks |

Each of these is a solved problem.
The unsolved problem is that they are not solved **together** — or solved **consistently**.

This talk shows two tools — LinuxAid and Juju — that apply the same operator philosophy
to the OS layer and the application layer respectively, and how they compose naturally.

---

## Part 2 — LinuxAid: The OS Operator

### Slide 3 — What LinuxAid Is

LinuxAid is an **open-source OS management platform** built on OpenVox,
the community fork of Puppet.

It treats the operating system the same way Kubernetes treats workloads:
declare the desired state, and the system continuously reconciles toward it.

**What LinuxAid manages:**
- System hardening — SSH policy, PAM, sysctl, firewall
- Package repositories — GPG-signed, staged rollouts, air-gap support
- User management — centralised users, sudo policies
- SSL certificates — managed and rotated
- 60+ application profiles — Apache, PostgreSQL, GitLab, Mailcow, HAProxy, WireGuard, and more
- Compliance — GDPR, CIS benchmarks, NIS2 built-in
- Observability — 11+ Prometheus exporters deployed automatically

**Scale**: proven architecture supporting **20,000+ nodes**.

---

### Slide 4 — The LinuxAid Architecture

LinuxAid separates **code** (what to do) from **data** (how to do it for each node).

```
┌──────────────────────────────────────────────┐
│            Your Git Config Repo              │
│                                              │
│  agents/node-1.demo.yaml   ← node-specific  │
│  agents/tags/web/*.yaml    ← group config   │
│  data/global.yaml          ← all nodes      │
└────────────────────┬─────────────────────────┘
                     │
                     ▼
        ┌────────────────────────┐
        │    OpenVox Server      │
        │                        │
        │  ENC classifies node   │
        │  Hiera merges data     │
        │  Catalog compiled      │
        └────────────┬───────────┘
                     │  agent pull every 30 min
          ┌──────────┴──────────┐
          ▼                     ▼
     Linux Node 1          Linux Node 2
     (hardened)            (hardened)
```

**Control Plane Components:**

| Component | Role |
|-----------|------|
| OpenVox Server | Compiles catalog from code + hiera data |
| Hiera | Hierarchical data lookup — node → group → global |
| ENC (External Node Classifier) | Assigns parameters to each node by certname |
| Git | All config versioned, change-previewed before applying |

---

### Slide 5 — The Hiera Hierarchy

Hiera is LinuxAid's configuration resolution engine.
It resolves which value applies to a given node by walking a priority chain:

```
Priority 1  →  agents/node-1.demo.yaml        (this node only)
Priority 2  →  agents/tags/web/*.yaml          (nodes tagged 'web')
Priority 3  →  facts/os.family=Debian.yaml     (all Debian nodes)
Priority 4  →  subscriptions/pro.yaml          (subscription tier)
Priority 5  →  data/global.yaml                (all nodes, all time)
```

This means:
- A global default applies everywhere with zero duplication
- A group override applies to tagged nodes without touching others
- A node-specific override is always respected, never silently overridden

Every value is traceable to the exact file and hierarchy level that provided it.
Encrypted secrets (eyaml / PKCS7) are supported at every level.

---

### Slide 6 — The Role-Profile-Common Pattern

Every node in LinuxAid is assigned a **role** — a declaration of what that machine does.

```
Role        →  what the node does (business meaning)
Profile     →  how to deploy the technology
Common      →  foundation shared by all nodes
```

Available roles include:

| Category | Roles |
|----------|-------|
| Web | `role::web::apache`, `role::web::haproxy` |
| Database | `role::db::mysql`, `role::db::postgresql` |
| Mail | `role::mail::mailcow` |
| CI/CD | `role::projectmanagement::gitlab` |
| Monitoring | `role::monitoring` |
| Basic | `role::basic` — SSH + firewall + all exporters |

Assigning `role::basic` to a node automatically enables:
SSH hardening, firewall management, automatic system updates,
package repository client, and all 11+ Prometheus exporters.

---

### Slide 7 — Observability as a First-Class Citizen

LinuxAid does not make you configure monitoring separately.
Every role includes the relevant exporters automatically.

**Exporters deployed by default on every node:**

| Exporter | What it measures |
|----------|-----------------|
| node_exporter | CPU, memory, disk, network, load |
| iptables_exporter | Firewall rule hit counts |
| systemd_exporter | Service states and failed units |
| process_exporter | Per-process CPU and memory |
| dns_exporter | DNS resolution health |
| mtail_exporter | Log-based metrics |

**Additional exporters enabled automatically when a role requires them:**

| Role | Extra exporters added |
|------|-----------------------|
| `role::db::mysql` | mysql_exporter |
| `role::web::haproxy` | haproxy_exporter |
| `role::web::apache` | apache_exporter |
| `role::projectmanagement::gitlab` | gitlab_runner_exporter |

No annotation, no scrape config, no ServiceMonitor.
**The exporter is there because the role is there.**

---

### Slide 8 — GitOps and Change Preview

One of LinuxAid's most important capabilities is that you can calculate the
**exact impact of a code change across all nodes before applying it**.

**The workflow:**
1. Make a change in your config Git repo
2. Run a changeset calculation — LinuxAid compiles the catalog for every node without applying
3. Review the diff — exactly which resources will change, on which nodes
4. 30,000 nodes typically produce 5–7 distinct changeset patterns
5. Approve and merge — agents apply on their next pull

This is the **same guarantee** that `terraform plan` gives for infrastructure,
applied to operating system configuration.

---

## Part 3 — Juju: The Application Operator

### Slide 9 — What Juju Is

Juju is a **model-driven application operator** from Canonical.

It deploys **Charmed Operators** — software packages that encode not just installation,
but the entire operational lifecycle of an application:
configuration, scaling, upgrading, relating to other applications, and running backups.

The key mental model:

> In Kubernetes, you write a Deployment and manage its connections manually.
> In Juju, you declare a model and Juju manages connections for you.

**400+ Charmed Operators** are available on Charmhub — for databases,
message queues, observability stacks, identity providers, and more.

---

### Slide 10 — Why Juju, Not Just Helm

The difference between Juju and Helm is not installation — it is **day-2 operations**.

| Capability | Helm | Juju |
|-----------|------|------|
| Install | ✅ | ✅ |
| Upgrade | `helm upgrade` (manual) | `juju upgrade-charm` (managed) |
| Scale | Set replicas manually | `juju add-unit` |
| Connect two applications | Pass secrets manually | `juju relate app-a app-b` |
| Application config changes | Edit values, re-apply | `juju config app key=val` |
| Run backups | You write it | `juju run pg/0 create-backup` |
| Health status | Check pods | `juju status` gives app-level view |
| Cross-application dependencies | Manual env vars | Relations handle the wiring automatically |

**Juju `relate`** is the core differentiator.
When you run `juju relate postgresql my-app`, Juju:
- Tells PostgreSQL to create a database and user
- Passes the credentials to `my-app` automatically
- Maintains this relationship through restarts and upgrades

You do not write a single environment variable.

---

### Slide 11 — Juju Models and Controllers

Juju introduces two concepts that replace ad-hoc cluster sprawl:

**Controller** — the brain of Juju. One controller manages many models.
Deployed once per cloud or Kubernetes cluster. Persists across all operations.

**Model** — a namespace for a group of applications that work together.
Think of it as a bounded context:
`monitoring`, `production-data`, `staging`, `ci-cd`.

Applications in the same model can be related to each other directly.
Applications in different models can be related via cross-model relations (CMR).

```
Juju Controller
├── model: monitoring
│   ├── prometheus
│   ├── grafana
│   └── alertmanager
├── model: production
│   ├── postgresql (related to my-app)
│   └── my-app
└── model: microk8s
    └── microk8s (Kubernetes cluster)
```

---

### Slide 12 — COS Lite: The Observability Model

**COS Lite** (Canonical Observability Stack Lite) is a single Juju bundle that deploys:

| Component | Purpose |
|-----------|---------|
| Prometheus | Metrics collection and rule evaluation |
| Grafana | Dashboards and visualisation |
| Alertmanager | Alert routing and deduplication |
| Loki | Log aggregation |
| Traefik | Ingress for all components |

**What makes COS Lite different from a manual Prometheus stack:**

1. Every component is a charmed operator — upgrades are coordinated, not manual
2. `juju relate cos-lite microk8s` auto-configures scraping of the MicroK8s cluster
3. Grafana datasources and dashboards are wired automatically via relations
4. Adding a new application and relating it to COS Lite registers its metrics automatically

---

## Part 4 — How They Work Together

### Slide 13 — The Complementary Problem Space

LinuxAid and Juju solve different problems at different layers.
They do not overlap. They compose.

```
LinuxAid owns:    the OS, the machine, the exporter
Juju owns:        the application, the cluster, the relation
```

| Concern | LinuxAid | Juju |
|---------|----------|------|
| Node hardening | ✅ | — |
| Firewall management | ✅ | — |
| Prometheus exporters on nodes | ✅ | — |
| Kubernetes lifecycle | — | ✅ (MicroK8s charm) |
| Application deployment | — | ✅ (Charms) |
| Application relations | — | ✅ |
| Cluster observability | — | ✅ (COS Lite) |
| OS-level observability | ✅ | feeds into → COS Lite |

The final row is where they meet:
LinuxAid installs `node_exporter` on every machine.
Juju's Prometheus in COS Lite scrapes those exporters.
**Full observability from bare metal to application — no manual scrape configuration.**

---

### Slide 14 — The Full Stack Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Git Repo: your-linuxaid-config                                 │
│  Declare: roles, exporters, firewall, compliance, users         │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
                  ┌──────────────────┐
                  │  OpenVox Server  │
                  │  (LinuxAid CP)   │
                  └────────┬─────────┘
                           │  reconciles every 30 min
            ┌──────────────┴──────────────┐
            ▼                             ▼
     Linux Node 1                  Linux Node 2
     ─────────────                  ─────────────
     • SSH hardened                 • SSH hardened
     • Firewall managed             • Firewall managed
     • node_exporter :9100          • node_exporter :9100
     • 10+ more exporters           • 10+ more exporters
     • Compliant (CIS/GDPR)         • Compliant (CIS/GDPR)
            │                             │
            └──────────────┬──────────────┘
                           │  exporters scraped by
                           ▼
          ┌──────────────────────────────────┐
          │         Juju Controller          │
          │                                  │
          │  model: microk8s                 │
          │  ├── MicroK8s (multi-node)        │
          │                                  │
          │  model: monitoring (COS Lite)    │
          │  ├── Prometheus ◄── node metrics │
          │  ├── Grafana    ◄── dashboards   │
          │  ├── Alertmanager                │
          │  └── Loki       ◄── logs         │
          │                                  │
          │  model: production               │
          │  ├── PostgreSQL ←─── related ──┐ │
          │  └── your-app  ────────────────┘ │
          └──────────────────────────────────┘
```

---

### Slide 15 — The Operator Pattern at Every Layer

Both tools apply the same philosophical principle:
**declare what you want, let the operator converge toward it.**

| Layer | Tool | Operator mechanism |
|-------|------|-------------------|
| OS baseline | LinuxAid | OpenVox agent reconciles every 30 minutes |
| OS observability | LinuxAid | Role assigns exporters declaratively |
| Kubernetes cluster | Juju MicroK8s charm | Charm handles join, upgrade, token rotation |
| Application deployment | Juju Charms | Charm handles install, config, scale, upgrade |
| Application relations | Juju | Relations engine passes credentials and config |
| Observability stack | Juju COS Lite | Bundle wires Prometheus + Grafana automatically |

One philosophy. Two tools. Every layer covered.

---

## Part 5 — Production Relevance

### Slide 16 — This Is How Obmondo Runs Production

Obmondo manages infrastructure for customers at scale using this exact stack:

- **LinuxAid** manages every node: OS hardening, exporters, updates, compliance
- **KubeAid** (Obmondo's Kubernetes GitOps platform) manages cluster lifecycle
- **kube-prometheus-stack** collects metrics from LinuxAid node exporters
- Customers get a fully observable, compliant, GitOps-driven platform on day one

The LinuxAid node exporters are not an afterthought.
They are the foundation of the observability model.
Every Prometheus alert, every Grafana dashboard, every SLA is backed by data
that LinuxAid installed and LinuxAid keeps running.

---

### Slide 17 — What You Get From Day One

A new node enrolled in this stack receives automatically:

**From LinuxAid:**
- SSH hardened to policy
- Firewall configured
- GPG-signed package repository client configured
- Automatic security updates enabled
- 11+ Prometheus exporters installed and running
- GDPR / CIS compliance applied

**From Juju + COS Lite:**
- Node metrics visible in Grafana within minutes of scrape config
- Alerts firing based on pre-built Prometheus rules
- Log aggregation in Loki
- No manual dashboard creation, no manual scrape config

**The total operator investment for a new node: one line in a YAML file.**

---

### Slide 18 — Comparing to the Alternative

What does building this without LinuxAid and Juju look like?

| Task | Without | With LinuxAid + Juju |
|------|---------|----------------------|
| Harden 10 new nodes | Ansible playbook per team | One role assignment in Git |
| Add node_exporter to all nodes | Separate Ansible role or manual | Already there with `role::basic` |
| Deploy PostgreSQL with monitoring | Helm + manual secrets + ServiceMonitor | `juju deploy postgresql` + `juju relate` |
| Preview config changes before applying | Not possible | Built-in changeset calculation |
| Scale from 10 to 100 nodes | Re-run all playbooks, update all scrape configs | Git commit, agents converge |
| Compliance audit | Manual checklist | Pre-built GDPR/CIS/NIS2 profiles |

---

## Part 6 — Key Takeaways

### Slide 19 — What to Remember

1. **Kubernetes is not a platform** — it is one layer of a platform.
   Every layer above it must be designed and operated deliberately.

2. **The operator pattern works at every layer** — not just for Kubernetes workloads.
   LinuxAid applies it to the OS. Juju applies it to applications.

3. **Observability must be built in, not bolted on.**
   LinuxAid deploys exporters as part of the role. COS Lite scrapes them automatically.
   The data is there from day one because the design demands it.

4. **GitOps means full auditability** — not just for Kubernetes resources, but for
   OS configuration, user access, firewall rules, and application deployments.

5. **Composability beats monoliths** — LinuxAid and Juju solve different problems.
   They compose cleanly because each has a clear boundary.

---

### Slide 20 — Open Source and Community

| Project | Maintainer | Source |
|---------|-----------|--------|
| LinuxAid | Obmondo | https://github.com/Obmondo/LinuxAid |
| LinuxAid CLI | Obmondo | https://github.com/Obmondo/linuxaid-cli |
| OpenVox | Vox Pupuli | https://voxpupuli.org/openvox |
| Juju | Canonical | https://juju.is |
| MicroK8s Charm | Canonical | https://charmhub.io/microk8s |
| COS Lite | Canonical | https://charmhub.io/cos-lite |

All components are open source.
No vendor lock-in. Your config repo. Your servers. Your data.

---

## Appendix — Architecture Reference

### LinuxAid Module Hierarchy

```
role::*
  └── profile::*
        └── common::*
              ├── common::monitor::exporter::node
              ├── common::monitor::exporter::iptables
              ├── common::monitor::exporter::systemd
              ├── common::monitor::exporter::process
              ├── common::network::firewall
              ├── common::users
              └── common::packages
```

### Juju Relation Graph (COS Lite + MicroK8s)

```
microk8s ──── cos-lite
                ├── prometheus
                │     ├── grafana (datasource)
                │     └── alertmanager (receiver)
                └── loki
                      └── grafana (datasource)
```

### Hiera Hierarchy Levels (Priority Order)

| Priority | Scope | File path |
|----------|-------|-----------|
| 1 | Node-specific | `agents/<certname>.yaml` |
| 2 | Tag group | `agents/tags/<tag>/*.yaml` |
| 3 | OS/hardware facts | `facts/os.family=<family>.yaml` |
| 4 | Subscription tier | `subscriptions/<tier>.yaml` |
| 5 | Global default | `data/global.yaml` |
