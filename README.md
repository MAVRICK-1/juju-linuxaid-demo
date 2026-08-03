# LinuxAid + Juju Demo

> **Talk:** "Kubernetes Is Not a Platform: Building One with Juju"
> **Stack:** LinuxAid (OS layer) → Juju (K8s + apps + monitoring)

## Why Two Tools?

| Tool | Responsibility |
|------|---------------|
| **LinuxAid** | OS hardening, SSH, firewall, NTP, Prometheus exporters on every node |
| **Juju** | Deploy MicroK8s, PostgreSQL, Kafka, COS Lite (Prometheus+Grafana) |

## Architecture

```
                  Hetzner CX22 (OpenVox Server)
                  ├── puppet code  ← github.com/Obmondo/LinuxAid
                  └── hiera data   ← github.com/MAVRICK-1/juju-linuxaid-demo
                             │  port 8140
             ┌───────────────┴───────────────┐
             ▼                               ▼
    Hetzner node-1 (CX32)          Hetzner node-2 (CX22)
    LinuxAid → OS hardened         LinuxAid → OS hardened
    node_exporter :9100            node_exporter :9100
    Juju agent                     Juju agent
    MicroK8s control plane         MicroK8s worker
             │
             ▼
    Juju controller
    ├── MicroK8s (multi-node K8s)
    ├── COS Lite
    │   ├── Prometheus  ← scrapes node_exporter from all nodes
    │   ├── Grafana     ← dashboards showing LinuxAid node metrics
    │   └── AlertManager
    └── Apps (day-2 managed by Juju)
        ├── PostgreSQL with replication
        ├── Kafka
        └── OpenSearch
```

---

## Part 1 — Hetzner Setup

### 1.1 Create servers in Hetzner Cloud Console

Go to https://console.hetzner.cloud → New Project → Add Servers:

| Name | Type | OS | Role |
|------|------|----|------|
| openvox | CX22 | Ubuntu 24.04 | OpenVox puppet server |
| node-1 | CX32 | Ubuntu 24.04 | K8s control plane |
| node-2 | CX22 | Ubuntu 24.04 | K8s worker |

- Location: **Nuremberg (nbg1)**
- Add your SSH public key in Hetzner UI
- Note all three public IPs

### 1.2 Generate SSH key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/hetzner -N ""
cat ~/.ssh/hetzner.pub  # paste into Hetzner UI
```

### 1.3 Fix SSH known_hosts when recreating servers

```bash
ssh-keygen -f ~/.ssh/known_hosts -R '<SERVER-IP>'
ssh -i ~/.ssh/hetzner -o StrictHostKeyChecking=no root@<SERVER-IP>
```

---

## Part 2 — OpenVox Server on Hetzner

### 2.1 Install OpenVox Server

```bash
ssh -i ~/.ssh/hetzner root@<OPENVOX-IP>

# Add OpenVox apt repo (no hyphen before ubuntu in filename)
wget https://apt.voxpupuli.org/openvox8-release-ubuntu24.04.deb
sudo dpkg -i openvox8-release-ubuntu24.04.deb
sudo apt update && sudo apt install -y openvox-server

# Service is named 'puppetserver', not 'openvox-server'
sudo systemctl start puppetserver
sudo systemctl enable puppetserver
sudo systemctl status puppetserver
```

### 2.2 Note the internal hostname (needed for TLS)

```bash
sudo openssl x509 -in /etc/puppetlabs/puppet/ssl/ca/ca_crt.pem \
  -text -noout | grep "Subject:"
# Example: CN = Puppet CA: openvox.nbg1.hetzner.com
```

### 2.3 Configure autosign (demo only)

```bash
echo '*' | sudo tee /etc/puppetlabs/puppet/autosign.conf
sudo systemctl restart puppetserver
```

### 2.4 Install r10k and deploy LinuxAid puppet code

```bash
sudo apt install -y git ruby ruby-dev
sudo gem install r10k

# Configure r10k to pull puppet code from LinuxAid
sudo mkdir -p /etc/puppetlabs/r10k
sudo tee /etc/puppetlabs/r10k/r10k.yaml <<EOF
cachedir: /var/cache/r10k
sources:
  code:
    remote: https://github.com/Obmondo/LinuxAid.git
    basedir: /etc/puppetlabs/code/environments
EOF

# Fix permissions
sudo chown -R puppet:puppet /etc/puppetlabs/code/environments
sudo chmod -R 755 /etc/puppetlabs/code/environments

# Deploy LinuxAid master branch
sudo r10k deploy environment master -v

# Verify
sudo ls /etc/puppetlabs/code/environments/master/modules/enableit/
```

### 2.5 Set up hiera config repo

```bash
# Clone our hiera data repo
sudo mkdir -p /etc/puppetlabs/code/hiera-data/demo
sudo git clone https://github.com/MAVRICK-1/juju-linuxaid-demo.git \
  /etc/puppetlabs/code/hiera-data/demo/main

# Override hiera.yaml — use plain yaml (no eyaml needed for demo)
sudo tee /etc/puppetlabs/code/environments/master/hiera.yaml <<EOF
version: 5
defaults:
  datadir: /etc/puppetlabs/code/hiera-data/demo/main
  data_hash: yaml_data
hierarchy:
  - name: "Node-specific"
    path: "agents/%{trusted.certname}.yaml"
  - name: "Global"
    path: "data/global.yaml"
EOF
```

### 2.6 Set up simplified ENC

The ENC injects required top-scope variables that LinuxAid site.pp needs:

```bash
sudo tee /etc/puppetlabs/code/environments/master/enc.rb <<'EOF'
#!/opt/puppetlabs/puppet/bin/ruby
require 'yaml'
certname = ARGV[0]
customer_id = certname.split('.', 2)[1] || 'demo'
puts YAML.dump({
  'parameters' => {
    'obmondo_monitor' => false,
    'subscription'    => nil,
    'hiera_datapath'  => customer_id + '/main',
    'obmondo_tags'    => [],
    'obmondo_tag_0' => '', 'obmondo_tag_1' => '', 'obmondo_tag_2' => '',
    'obmondo_tag_3' => '', 'obmondo_tag_4' => '', 'obmondo_tag_5' => '',
    'obmondo_tag_6' => '', 'obmondo_tag_7' => '', 'obmondo_tag_8' => '',
    'obmondo_tag_9' => '',
  },
  'environment' => 'master'
})
EOF

sudo chmod +x /etc/puppetlabs/code/environments/master/enc.rb

# Enable ENC in puppet.conf
sudo tee -a /etc/puppetlabs/puppet/puppet.conf <<EOF

[server]
node_terminus = exec
external_nodes = /etc/puppetlabs/code/environments/master/enc.rb
EOF

sudo systemctl restart puppetserver
sudo systemctl status puppetserver --no-pager | head -5
```

### 2.7 GitOps: update node config

```bash
# Pull latest config and redeploy
sudo git -C /etc/puppetlabs/code/hiera-data/demo/main pull
sudo r10k deploy environment master -v
sudo systemctl reload puppetserver
```

> **GitOps flow:** Edit `agents/<node>.yaml` or `data/global.yaml` in this repo →
> push to GitHub → run the two commands above on OpenVox server →
> nodes pick up changes on next puppet run (every 30 min).

---

## Part 3 — Install LinuxAid on each node

Run on **node-1** and **node-2** (change `--certname` for each).

```bash
ssh -i ~/.ssh/hetzner root@<NODE-IP>

# 1. Install prereqs
sudo apt update && sudo apt install -y jq

# 2. Install LinuxAid CLI
curl -sSL https://raw.githubusercontent.com/Obmondo/linuxaid-cli/main/install.sh | bash

# 3. Install openvox-agent (provides puppet binary)
wget https://apt.voxpupuli.org/openvox8-release-ubuntu24.04.deb
sudo dpkg -i openvox8-release-ubuntu24.04.deb
sudo apt update && sudo apt install -y openvox-agent

# 4. Map OpenVox hostname → IP for TLS verification
echo "<OPENVOX-IP>  <OPENVOX-INTERNAL-HOSTNAME>" | sudo tee -a /etc/hosts
# e.g: echo "10.0.0.1  openvox.nbg1.hetzner.com" | sudo tee -a /etc/hosts

# 5. Run linuxaid-install (IP:port bypasses hardcoded port 443 check)
linuxaid-install \
  --certname node-1.demo \
  --puppet-server <OPENVOX-IP>:8140

# 6. Fix puppet.conf — linuxaid-install hardcodes masterport=443
sudo tee /etc/puppetlabs/puppet/puppet.conf <<EOF
[main]
server = <OPENVOX-INTERNAL-HOSTNAME>
certname = node-1.demo
masterport = 8140
stringify_facts = false

[agent]
report = true
pluginsync = true
noop = false
environment = master
EOF

# 7. First puppet run
sudo /opt/puppetlabs/bin/puppet agent --test \
  --server <OPENVOX-INTERNAL-HOSTNAME> \
  --masterport 8140 \
  --environment master \
  --no-daemonize
```

> **What LinuxAid deploys via `role::basic`:**
> - SSH hardening
> - Firewall (iptables)
> - NTP sync
> - `node_exporter` :9100 — CPU, RAM, disk, network metrics
> - `iptables_exporter` — firewall metrics
> - `systemd_exporter` — service health

### 3.1 Verify exporters on node

```bash
curl http://localhost:9100/metrics | grep node_cpu_seconds
```

### 3.2 Verify all nodes registered (on OpenVox server)

```bash
echo "127.0.0.1  puppet" | sudo tee -a /etc/hosts
sudo /opt/puppetlabs/bin/puppetserver ca list --all
# Should show: node-1.demo  node-2.demo
```

---

## Part 4 — Install Juju on node-1

```bash
ssh -i ~/.ssh/hetzner root@<NODE1-IP>

snap install juju --classic
mkdir -p ~/.local/share/juju
```

---

## Part 5 — Bootstrap Juju (Hetzner provider)

```bash
# Get Hetzner API token:
# https://console.hetzner.cloud → Project → Security → API Tokens → Generate

juju add-credential hetzner
# Enter API token when prompted

juju bootstrap hetzner hetzner-controller
juju status
```

---

## Part 6 — Deploy MicroK8s (multi-node)

```bash
juju add-model k8s-demo

# Deploy control plane
juju deploy microk8s \
  --channel 1.28/stable \
  --base ubuntu@22.04

# Watch it come up
juju status --watch 5s

# Once active, add worker node
juju add-unit microk8s
juju status --watch 5s
```

### 6.1 Access the cluster

```bash
mkdir -p ~/.kube
juju ssh microk8s/0 -- sudo microk8s config > ~/.kube/config
kubectl get nodes
```

Expected:
```
NAME          STATUS   ROLES   AGE
juju-xxxx-0   Ready    <none>  5m
juju-xxxx-1   Ready    <none>  3m
```

---

## Part 7 — Deploy COS Lite (Prometheus + Grafana)

COS Lite = Canonical Observability Stack — deployed and managed entirely by Juju.

```bash
juju add-model cos

# Deploy full observability stack in one command
juju deploy cos-lite --trust
juju status --watch 5s

# Relate to MicroK8s — auto-wires scrape config
juju relate cos-lite microk8s
```

### 7.1 Add LinuxAid node exporters as scrape targets

```bash
# Create scrape config for LinuxAid nodes
cat > linuxaid-scrape.yaml <<EOF
- job_name: linuxaid-nodes
  static_configs:
    - targets:
        - '<NODE1-IP>:9100'
        - '<NODE2-IP>:9100'
      labels:
        managed_by: linuxaid
EOF

juju config prometheus scrape-jobs="$(cat linuxaid-scrape.yaml)"
```

### 7.2 Access Grafana

```bash
# Get Grafana URL
juju status cos --format=yaml | grep url

# Default credentials
# User: admin
# Password: get with: juju run grafana/0 get-admin-password
juju run grafana/0 get-admin-password
```

---

## Part 8 — Deploy apps via Juju (day-2 ops demo)

```bash
# Switch to k8s-demo model
juju switch k8s-demo

# Deploy PostgreSQL with replication
juju deploy postgresql --channel 14/stable

# Deploy your app and connect to PostgreSQL automatically
juju deploy my-app
juju relate my-app postgresql

# Scale PostgreSQL
juju add-unit postgresql

# Upgrade PostgreSQL charm
juju upgrade-charm postgresql
```

---

## Repo Structure (hiera config)

```
MAVRICK-1/juju-linuxaid-demo/
├── data/
│   └── global.yaml        # shared config for ALL nodes
└── agents/
    ├── node-1.demo.yaml   # node-1: classes + overrides
    └── node-2.demo.yaml   # node-2: classes + overrides
```

### data/global.yaml

```yaml
---
monitor::enable: true
monitor::service::enable: true
common::monitor::exporter::enable: true
common::monitor::exporter::node::enable: true
common::monitor::exporter::iptables::enable: true
common::monitor::exporter::systemd::enable: true
common::system::time::manage_ntp: true
```

### agents/node-1.demo.yaml

```yaml
---
classes:
  - role::basic
```

---

## Cost Estimate (7 days)

| Resource | Type | Cost/hr | 7 days |
|----------|------|---------|--------|
| Hetzner CX22 (openvox) | Hetzner | €0.007/hr | ~€1.18 |
| Hetzner CX32 (node-1) | Hetzner | €0.013/hr | ~€2.18 |
| Hetzner CX22 (node-2) | Hetzner | €0.007/hr | ~€1.18 |
| **Total** | | | **~€4.50** |

---

## Cleanup

```bash
# Destroy Juju models
juju destroy-model cos --force --no-wait
juju destroy-model k8s-demo --force --no-wait

# Delete all Hetzner servers from Hetzner console
```

---

## Known Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| `openvox-server.service not found` | Service is named `puppetserver` | `systemctl start puppetserver` |
| `linuxaid-install` hits port 443 | Hardcoded in CLI | Pass `IP:8140` as `--puppet-server` |
| `masterport = 443` after install | CLI hardcodes it in puppet.conf | Overwrite puppet.conf manually |
| Cert hostname mismatch | Cert issued to internal hostname | Add `/etc/hosts` mapping IP → hostname |
| `r10k deploy production` fails | LinuxAid has no `production` branch | Deploy `master` branch instead |
| `Permission denied on environments/` | Wrong ownership | `chown -R puppet:puppet /etc/puppetlabs/code/environments` |
| `Role: []` empty catalog | eyaml not installed / hiera.yaml wrong | Override hiera.yaml with plain `yaml_data` |
| `Unknown variable 'subscription'` | ENC missing variable | Add `subscription: null` to ENC |

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
