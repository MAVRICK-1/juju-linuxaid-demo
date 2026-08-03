# Kubernetes Is Not a Platform: Building One with Juju

**Presentation Overview**
Speaker: Rishi Mondal — SRE at Obmondo | CNCF KubeStellar Maintainer | Docker Captain

---

## Part 1 — The Problem

### Slide 1 — What Kubernetes Actually Gives You

Kubernetes gives you a container scheduler.
That is the complete list.

What you still need to build yourself:
- A managed, observable OS on every node
- Day-2 lifecycle for stateful applications (upgrades, scaling, backups)
- Wiring between applications — credentials, endpoints, config
- A coherent observability stack that is wired up automatically

Every team discovers these gaps the same way: in production, under pressure.

---

### Slide 2 — The Platform Gap

| Layer | What you need | What Kubernetes gives you |
|-------|--------------|--------------------------|
| OS | Hardened, patched, observable nodes | Nothing |
| Kubernetes | Lifecycle, upgrades, multi-node | The cluster itself |
| Applications | Deploy, connect, scale, back up stateful services | Nothing |
| Observability | Metrics from OS to app, wired automatically | Nothing |

The middle two columns are where platforms are built — or where they fall apart.

---

## Part 2 — Juju: The Application Operator

### Slide 3 — What Juju Is

Juju is a **model-driven application operator** by Canonical.

It deploys **Charmed Operators** — packages that encode not just installation,
but the entire operational lifecycle of an application:

- Install
- Configure
- Scale
- Upgrade
- **Relate to other applications automatically**
- Run operational actions (backup, restore, flush)

400+ Charmed Operators available on [Charmhub](https://charmhub.io).

---

### Slide 4 — The Key Concept: Relations

This is what separates Juju from every other deployment tool.

**With Helm / plain Kubernetes:**
- Deploy PostgreSQL → manually create a Secret with DB credentials
- Deploy your app → manually pass that Secret as an env var
- Upgrade PostgreSQL → manually rotate credentials → manually restart app
- Add a second app → repeat all of the above

**With Juju:**
```
juju relate my-app postgresql
```

Juju automatically:
1. Tells PostgreSQL to create a dedicated user and database
2. Passes credentials to `my-app` via the relation interface
3. Maintains this wiring through restarts, upgrades, and scaling events

You write zero environment variables. You manage zero secrets manually.

---

### Slide 5 — Juju Models and Controllers

Juju introduces two concepts that bring order to cluster sprawl:

**Controller** — the brain. One controller manages many models.
Deployed once per cloud or cluster. Persists indefinitely.

**Model** — a bounded namespace for applications that work together.

```
Juju Controller
├── model: monitoring          # Prometheus + Grafana + Loki
├── model: data                # PostgreSQL + Kafka (related)
├── model: ci                  # GitLab + runners
└── model: k8s                 # MicroK8s cluster itself
```

Applications in the same model relate directly.
Applications across models use cross-model relations (CMR).

---

### Slide 6 — Juju vs Helm: Day-2 Reality

Most teams adopt Helm for day-1 installs and discover its limits on day 2.

| Capability | Helm | Juju |
|-----------|------|------|
| Install | `helm install` | `juju deploy` |
| Upgrade | `helm upgrade` — you manage values | `juju upgrade-charm` — charm manages it |
| Scale | Edit replicas manually | `juju add-unit` |
| Connect two apps | Manually pass secrets / env vars | `juju relate app-a app-b` |
| Rotate credentials | Edit Secret, redeploy, restart | Handled by the charm relation |
| Run a backup | You write a CronJob | `juju run postgresql/0 create-backup` |
| Status at app level | `kubectl get pods` | `juju status` — app-aware view |
| Cross-app dependencies | You manage them forever | Relations encode and maintain them |

Helm is a package manager. Juju is an application operator.
They solve different problems — one of them at a much deeper layer.

---

### Slide 7 — COS Lite: Observability as a Charm Bundle

**COS Lite** (Canonical Observability Stack Lite) is a single Juju bundle that deploys
and wires together a complete observability stack:

| Component | Role |
|-----------|------|
| Prometheus | Metrics collection and alert evaluation |
| Grafana | Dashboards and visualisation |
| Alertmanager | Alert routing and deduplication |
| Loki | Log aggregation |
| Traefik | Ingress for all components |

**What makes it different from a manual Prometheus stack:**

1. All components are charmed operators — upgrades are coordinated, not manual
2. `juju relate cos-lite microk8s` auto-configures scraping of the cluster
3. Grafana datasources are wired via relations — no manual config
4. Any new application related to COS Lite registers its dashboards automatically

---

### Slide 8 — MicroK8s as a Juju Charm

Juju does not just deploy applications **on** Kubernetes.
It can also deploy **Kubernetes itself** as a charmed operator.

The MicroK8s charm handles:
- Multi-node cluster bootstrap
- Node join and token management
- Channel-based upgrades (`juju config microk8s channel=1.30/stable`)
- High availability control plane
- Addons (DNS, storage, ingress) managed via charm config

```
juju deploy microk8s --channel 1.28/stable   # control plane
juju add-unit microk8s                        # add worker
juju add-unit microk8s                        # add another worker
```

The cluster lifecycle is now expressed as a Juju model — versioned, auditable,
reproducible on any cloud.

---

## Part 3 — The OS Layer: Where LinuxAid Fits

### Slide 9 — The Layer Juju Does Not Own

Juju manages applications and Kubernetes clusters.
It does not manage the operating system underneath them.

Every node still needs:
- SSH hardening
- Firewall rules
- Automatic security updates
- Prometheus exporters for OS-level metrics
- Compliance posture (CIS, GDPR, NIS2)

You could do this with Ansible, shell scripts, or cloud-init.
The problem is those tools don't **continuously reconcile** — they run once and drift.

---

### Slide 10 — LinuxAid: Continuous OS Reconciliation

LinuxAid is a Puppet-based OS management tool (built on OpenVox) that applies
**the same operator philosophy to the OS** that Juju applies to applications:

Declare desired state → system continuously converges toward it.

**What it manages:**

| Concern | How |
|---------|-----|
| SSH policy | Declared in Hiera, applied every 30 min |
| Firewall | iptables rules generated from config |
| System updates | Unattended upgrades, staged rollouts |
| Prometheus exporters | Deployed automatically with the role |
| Compliance | CIS / GDPR / NIS2 profiles built in |

The config lives in Git. A push to the repo propagates to every node
on the next agent run — no SSH, no Ansible playbook, no manual step.

---

### Slide 11 — Why This Matters for Prometheus

The critical integration point is **node-level metrics**.

Juju's COS Lite runs Prometheus inside the cluster.
But the most important metrics — CPU steal, disk saturation, OOM events,
network errors, systemd unit failures — live **outside** the cluster, on the host OS.

LinuxAid deploys these exporters on every node automatically:

| Exporter | What it exposes |
|----------|----------------|
| node_exporter `:9100` | CPU, memory, disk, network, load |
| systemd_exporter | Failed/degraded systemd units |
| process_exporter | Per-process CPU and memory usage |
| iptables_exporter | Firewall rule hit counts |

COS Lite Prometheus scrapes them via static config.
Result: **host-to-application observability in one stack, no manual wiring.**

---

### Slide 12 — The Boundary Between the Two Tools

```
┌─────────────────────────────────────────────────┐
│  LinuxAid owns: OS state, machine config        │
│  Juju owns:     applications, cluster, relations │
├─────────────────────────────────────────────────┤
│  They meet at: Prometheus scrape targets        │
│  LinuxAid puts the exporter there.              │
│  Juju's Prometheus scrapes it.                  │
└─────────────────────────────────────────────────┘
```

| Concern | Tool | Why |
|---------|------|-----|
| SSH hardening | LinuxAid | OS-level, outside Juju's scope |
| Firewall | LinuxAid | Kernel netfilter, not a K8s resource |
| node_exporter | LinuxAid | Must run on the host, not in a container |
| MicroK8s cluster | Juju | Cluster lifecycle = charm's job |
| PostgreSQL | Juju | Stateful app lifecycle = charm's job |
| Prometheus (cluster) | Juju COS Lite | Application-layer observability |
| Cross-app credentials | Juju | Relations handle this |

Neither tool tries to own the other's layer.
That clean boundary is what makes them composable.

---

## Part 4 — Full Architecture

### Slide 13 — The Complete Picture

```
┌──────────────────────────────────────────────────────────────────┐
│  Git: your-linuxaid-config                                       │
│  agents/node-1.yaml → role::basic                                │
│  data/global.yaml   → exporters, firewall, updates              │
└────────────────────────┬─────────────────────────────────────────┘
                         │ r10k pulls on push
                         ▼
              ┌──────────────────────┐
              │    OpenVox Server    │
              │  compiles catalog    │
              │  puppet agent pulls  │
              │  every 30 minutes    │
              └──────────┬───────────┘
                         │
           ┌─────────────┴─────────────┐
           ▼                           ▼
    Linux Node 1                Linux Node 2
    ────────────                ────────────
    SSH hardened                SSH hardened
    Firewall managed            Firewall managed
    node_exporter :9100         node_exporter :9100
    systemd_exporter            systemd_exporter
    process_exporter            process_exporter
    Juju agent running          Juju agent running
    MicroK8s control plane      MicroK8s worker
           │                           │
           └─────────────┬─────────────┘
                         │
              ┌──────────▼───────────┐
              │   Juju Controller    │
              │                      │
              │  model: k8s          │
              │  └── MicroK8s        │
              │                      │
              │  model: monitoring   │
              │  ├── Prometheus ─────┼──► scrapes :9100 on nodes
              │  ├── Grafana         │   (host OS metrics)
              │  ├── Alertmanager    │
              │  └── Loki            │
              │                      │
              │  model: apps         │
              │  ├── PostgreSQL ◄──┐ │
              │  └── my-app ───────┘ │ (related — no manual secrets)
              └──────────────────────┘
```

---

## Part 5 — Key Takeaways

### Slide 14 — What to Remember

1. **Kubernetes is infrastructure, not a platform.**
   The application layer, the OS layer, and the observability layer are still yours to build.

2. **Juju solves the application layer** with operators that handle day-2 automatically.
   Relations eliminate the manual secret-passing problem permanently.

3. **The operator pattern is not just for Kubernetes workloads.**
   The same continuous reconciliation model works at the OS level too.

4. **Observability must be structural, not added later.**
   If the OS layer deploys exporters automatically, the metrics are always there.
   If Prometheus is a charm, scraping is always configured.

5. **Clean layer boundaries make tools composable.**
   Each tool owns what it is designed to own. Neither one overreaches.

---

### Slide 15 — References

| Resource | URL |
|----------|-----|
| Juju | https://juju.is |
| Charmhub | https://charmhub.io |
| MicroK8s Charm | https://charmhub.io/microk8s |
| COS Lite | https://charmhub.io/cos-lite |
| Charmed PostgreSQL | https://charmhub.io/postgresql |
| OpenVox | https://voxpupuli.org/openvox |
| LinuxAid | https://github.com/Obmondo/LinuxAid |

---

## Appendix — Architecture Reference

### LinuxAid Module Hierarchy

```
role::basic
  └── common::init
        ├── common::monitor::exporter::node
        ├── common::monitor::exporter::iptables
        ├── common::monitor::exporter::systemd
        ├── common::monitor::exporter::process
        ├── common::network::firewall
        ├── common::users
        └── common::packages
```

### Hiera Hierarchy (Priority Order)

| Priority | Scope | Path |
|----------|-------|------|
| 1 | Node-specific | `agents/<certname>.yaml` |
| 2 | Tag group | `agents/tags/<tag>/*.yaml` |
| 3 | OS/hardware facts | `facts/os.family=<value>.yaml` |
| 4 | Global default | `data/global.yaml` |

### Juju Relation Graph (COS Lite)

```
microk8s ──── cos-lite
                ├── prometheus ──── grafana   (datasource relation)
                ├── alertmanager ── prometheus (receiver relation)
                └── loki ────────── grafana   (datasource relation)
```
