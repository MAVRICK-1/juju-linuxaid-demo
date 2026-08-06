# Kubernetes Is Not a Platform: Building One with Juju

---

## Part 1 — The Problem

### Slide 1 — Congrats, You Have a Cluster. Now What?

You provisioned Kubernetes. You feel powerful.

You still don't have:

| Missing piece | Why it hurts |
|---|---|
| Hardened OS on every node | Security starts at the host, not the YAML |
| Day-2 ops for stateful apps | Upgrades/backups/rotation = manual labor forever |
| App-to-app wiring | Every secret typed by hand, every endpoint hardcoded |
| Observability | You're now a full-time Prometheus plumber |

Kubernetes gave you the floor. Enjoy building the rest of the house yourself.

---

### Slide 2 — How Teams "Solve" This

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

## Part 2 — Juju: An Operator for Everything Above the OS

### Slide 3 — What Juju Actually Is

Juju = Canonical's model-driven app operator.

It deploys **Charmed Operators** — apps that know their *entire* lifecycle, not just "install and pray":

```
Install → Configure → Relate → Scale → Upgrade → Backup
```

400+ charms on Charmhub: PostgreSQL, Kafka, OpenSearch, Vault, Grafana...

---

### Slide 4 — The Relation: Juju's Party Trick

**Helm way:** deploy Postgres, deploy app, hand-craft a Secret, wire it as env vars, re-do it every upgrade.

**Juju way:**

```
juju relate my-app postgresql
```

That's it. Juju creates the user, hands over credentials, and keeps the wiring alive through restarts and upgrades.

No Secrets copy-pasted by a tired human at 11 PM.

---

### Slide 5 — Juju vs Helm, Head to Head

| Capability | Helm | Juju |
|---|---|---|
| Install | ✅ | ✅ |
| Upgrade | You babysit values drift | Charm owns it |
| Scale | Edit replicas by hand | `juju add-unit` |
| Connect two apps | Manual secrets + env vars | `juju relate` |
| Rotate credentials | Edit → redeploy → restart | Charm handles it |
| Backup | Write your own CronJob | `juju run postgresql/0 create-backup` |
| Status | `kubectl get pods` (infra view) | `juju status` (app-aware) |

Helm is a package manager. Juju is an operator. Different sports.

---

### Slide 6 — COS Lite: Observability in One Line

```
juju deploy cos-lite --trust
juju relate cos-lite microk8s
```

| Piece | Job |
|---|---|
| Prometheus | Metrics + alerts |
| Grafana | Dashboards, auto-wired datasources |
| Alertmanager | Routing + dedup |
| Loki | Logs |
| Traefik | Ingress |

No hand-written `prometheus.yml`. No manual Grafana datasource clicking.

---

### Slide 7 — Juju Can Even Charm Kubernetes Itself

```
juju deploy microk8s --channel 1.28/stable   # control plane
juju add-unit microk8s                        # worker
juju add-unit microk8s                        # another worker
```

Bootstrap, HA, channel upgrades, addons — all charm config.
Your entire cluster lifecycle now lives in Git.

---

## Part 3 — The Gap Juju Politely Ignores

### Slide 8 — What Juju Won't Touch

Juju rules the cluster and the apps on it. The **OS underneath** is not its problem.

Every node still needs:

```
SSH hardening        → who logs in, and how
Firewall             → kernel-level traffic rules
Auto security updates → patches without a human awake
Prometheus exporters → host-level CPU/mem/disk/systemd
Compliance           → CIS, GDPR, NIS2
```

Ansible/cloud-init can do this — once. Then the node drifts silently and nobody notices until the audit.

---

## Part 4 — LinuxAid: The Bottom Half Nobody Wanted to Own

### Slide 9 — Continuous OS Reconciliation

LinuxAid (Puppet, built on OpenVox) applies the same "declare it, reconcile it forever" philosophy — to the OS.

Every 30 min, every node pulls its catalog and self-corrects.
SSH drifted? Fixed. Exporter died? Restarted. New node? Converges in one run.

```
your-linuxaid-config/
├── data/global.yaml        # everyone
└── agents/
    ├── node-1.demo.yaml
    └── node-2.demo.yaml
```

Push to Git → r10k deploys → nodes fall in line. No SSH heroics required.

---

### Slide 10 — One Role, Zero Effort: `role::basic`

```yaml
# agents/node-1.demo.yaml
classes:
  - role::basic
```

One line buys you:

| What | Why |
|---|---|
| SSH hardened | No root login, key-only, idle timeout |
| Firewall | Declared in hiera, not fat-fingered |
| Unattended security updates | Patches apply while you sleep |
| `node_exporter` :9100 | Host metrics |
| `systemd_exporter` | Failed units, visible instantly |
| `process_exporter` | Per-process CPU/mem |
| `iptables_exporter` | Firewall hit counts |

Reconciled every 30 minutes. Traceable to a Git commit. No mystery config.

---

## Part 5 — Where This Stops Being a Cluster and Starts Being a Platform

### Slide 11 — Two Operators, One Philosophy

```
LinuxAid                        Juju
Owns: the OS                    Owns: the apps
Tool: OpenVox (Puppet)          Tool: Charmed Operators
Apply: agent, every 30 min      Apply: charm, continuously
       ◄──── same philosophy ────►
   Declare desired state. Operator converges. You sleep.
```

They meet at exactly one point: **Prometheus.**
LinuxAid puts `node_exporter` everywhere → Juju's COS Lite scrapes it.
Host-to-app observability, zero manual ServiceMonitors.

---

### Slide 12 — The Full Stack, One Diagram

```
Git (LinuxAid config) → OpenVox catalog → agent pulls every 30 min
      → nodes get: SSH hardened, firewall, node/systemd/process exporters

Juju agents on same nodes → MicroK8s cluster
      → Juju Controller runs 3 models:
          - k8s: MicroK8s
          - monitoring: Prometheus ← scrapes host + cluster metrics
                        Grafana, Alertmanager, Loki
          - apps: PostgreSQL ↔ my-app (related, no manual secrets)
```

Two Git repos. Zero manual wiring. One Grafana to rule them all.

---

### Slide 13 — Who Owns What (So Nobody Fights)

| Concern | Owner | Boundary |
|---|---|---|
| SSH, firewall, OS packages | LinuxAid | Juju doesn't go near it |
| Host exporters | LinuxAid | Bare metal, not a pod |
| OS compliance | LinuxAid | Agent-reconciled |
| K8s cluster lifecycle | Juju | MicroK8s charm |
| Stateful apps | Juju | Install/scale/upgrade via charm |
| Cross-app credentials | Juju | Relations, no hand-written Secrets |
| Observability stack | Juju (COS Lite) | Prometheus/Grafana as charms |
| Host metrics *in* that Prometheus | LinuxAid → Juju | The handshake point |

Every layer, one owner. No turf wars.

---

### Slide 14 — Day One of a New Node

**Minute 0–30 (LinuxAid):** SSH hardened, firewall up, updates on, 3 exporters running, CIS applied.

**Minute 30 (Juju):** Node joins MicroK8s, COS Lite starts scraping `:9100`, metrics show up in Grafana.

**Then:**
```
juju deploy postgresql
juju deploy my-app
juju relate my-app postgresql
```
Credentials wired automatically. App connected.

**Total human effort: one YAML line + three Juju commands.**

---

## Part 6 — Before vs After (a.k.a. Receipts)

### Slide 15 — Onboarding a Node

**Old way:**
```
Day 1   Run Ansible for SSH
Day 1   Run another Ansible for exporters
Day 1   Hand-edit prometheus.yml, reload, hope
Day 2   Realize compliance got skipped, run yet another playbook
Day 7   Node drifts. Silently. Forever.
Day 30  Audit asks for evidence — you grep bash history like a caveman
```

**New way:**
```
Day 1   One line in agents/node-1.yaml
        SSH + firewall + 4 exporters + CIS, done, metrics live
Day 30  Audit: git log. Every change, every author, every timestamp.
```

---

### Slide 16 — Wiring PostgreSQL to Your App

**Without Juju:** 7 manual steps involving copy-pasting a password out of a Secret, editing a Deployment, and doing it all over again for staging. You know this pain personally.

**With Juju:**
```
juju deploy postgresql
juju deploy my-app
juju relate my-app postgresql
```
Done. Upgrades rotate the password for you. Staging looks identical to prod, because it *is* the same relation.

---

### Slide 17 — Debugging at 3 AM

**Without LinuxAid:** Alert fires, you SSH in, systemd looks "fine," `node_exporter` has been dead for 6 hours and nobody noticed, your alert was firing on stale data the whole time. Root cause: unknown. Sleep: also unknown.

**With LinuxAid:** Alert fires, Grafana already shows `nginx.service` failed at 02:47, `node_exporter` auto-restarted itself at 02:47. Root cause visible without touching SSH.

The exporter didn't vanish quietly — LinuxAid caught it before you did.

---

### Slide 18 — Scaling 5 Nodes to 50

**Old way:** 45 new VMs, 45× Ansible hardening, 45× exporter installs, hand-edit `prometheus.yml`, reload, pray nothing was missed.

**New way:**
```
Provision 45 VMs → linuxaid-install (or one cloud-init line)
Add 45 lines to agents/ in Git → push → r10k → converged
juju add-unit microk8s × N
Prometheus auto-discovers the rest
```

Git is truth. Nodes converge to it. Scale is a commit, not a campaign.

---

## Part 7 — Key Takeaways

### Slide 19 — What to Actually Remember

1. **Kubernetes schedules containers. That's it.** Everything else is on you.
2. **Juju owns the app layer.** Relations kill manual secrets. Charms kill manual day-2.
3. **Juju stops at the OS.** SSH, firewall, exporters, compliance — nobody's covering that by default.
4. **LinuxAid covers it — same philosophy, different layer.** Git → reconcile → repeat, forever.
5. **They meet at Prometheus.** OS + app, both self-healing, both feeding one dashboard. That's not a cluster. That's a platform.

---

### Slide 20 — References

| Resource | URL |
|---|---|
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
microk8s ── cos-lite
              ├── prometheus
              │     ├── grafana (datasource)
              │     └── alertmanager (receiver)
              └── loki
                    └── grafana (datasource)
```

### LinuxAid Hiera Priority

| Priority | Scope | Path |
|---|---|---|
| 1 (highest) | Node-specific | `agents/<certname>.yaml` |
| 2 | Tag group | `agents/tags/<tag>/*.yaml` |
| 3 | OS/hardware facts | `facts/os.family=<value>.yaml` |
| 4 (lowest) | Global default | `data/global.yaml` |

### `role::basic` Module Tree

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
