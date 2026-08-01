# LinuxAid + Juju + MicroK8s Demo

> **Talk:** "Kubernetes Is Not a Platform: Building One with Juju"
> **Stack:** LinuxAid (OS layer) → Juju (K8s deployment) → MicroK8s → KubeAid (monitoring)

## Architecture

```
                    EC2 (OpenVox Server :8140)
                    ├── puppet code  ← github.com/Obmondo/LinuxAid
                    └── hiera data   ← github.com/MAVRICK-1/juju-linuxaid-demo
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
      Hetzner node-1 (CX32)          Hetzner node-2 (CX22)
      LinuxAid → OS hardened         LinuxAid → OS hardened
      exporters: node, iptables,     exporters: node, iptables,
                 systemd, process               systemd, process
      Juju agent                     Juju agent
      MicroK8s control plane         MicroK8s worker
               │
               ▼
      MicroK8s cluster
      └── KubeAid (ArgoCD)
          └── prometheus-linuxaid
              ├── Prometheus   ← scrapes exporters from all nodes
              ├── Grafana      ← dashboards at grafana-linuxaid.<domain>
              └── AlertManager
```

---

## Part 1 — OpenVox Server on EC2

### 1.1 Launch EC2 instance

- AMI: **Ubuntu 24.04 LTS**
- Instance type: **t3.medium** (2 vCPU, 4 GB RAM)
- Security group inbound rules:

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH |
| 8140 | TCP | OpenVox agent ↔ server (catalog + cert signing) |
| 443 | TCP | linuxaid-install compatibility check |

### 1.2 Install OpenVox Server

```bash
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>

# Add OpenVox apt repo (no hyphen before ubuntu in filename)
wget https://apt.voxpupuli.org/openvox8-release-ubuntu24.04.deb
sudo dpkg -i openvox8-release-ubuntu24.04.deb
sudo apt update && sudo apt install -y openvox-server

# The systemd service is named 'puppetserver', not 'openvox-server'
sudo systemctl start puppetserver
sudo systemctl enable puppetserver
sudo systemctl status puppetserver
```

### 1.3 Deploy LinuxAid puppet code via r10k

LinuxAid splits puppet code and hiera config into **two separate repos**:

| Repo | Role | Contents |
|------|------|----------|
| `Obmondo/LinuxAid` | Puppet code | Modules, site.pp, ENC |
| `MAVRICK-1/juju-linuxaid-demo` | Hiera data | `global.yaml`, `agents/*.yaml` |

```bash
# Install r10k
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

# Fix permissions (puppetserver runs as 'puppet' user)
sudo chown -R puppet:puppet /etc/puppetlabs/code/environments
sudo chmod -R 755 /etc/puppetlabs/code/environments

# Deploy — LinuxAid uses 'master' branch (maps to 'master' environment)
sudo r10k deploy environment master -v

# Verify modules loaded
sudo ls /etc/puppetlabs/code/environments/master/modules/enableit/
```

### 1.4 Set up hiera config repo

```bash
# Clone hiera data repo (your node config lives here)
sudo git clone https://github.com/MAVRICK-1/juju-linuxaid-demo.git \
  /etc/puppetlabs/code/hieradata

# Configure hiera.yaml to read from it
sudo tee /etc/puppetlabs/code/environments/master/hiera.yaml <<EOF
version: 5
hierarchy:
  - name: "Node-specific"
    path: "agents/%{trusted.certname}.yaml"
    datadir: /etc/puppetlabs/code/hieradata
  - name: "Global"
    path: "global.yaml"
    datadir: /etc/puppetlabs/code/hieradata/data
defaults:
  data_hash: yaml_data
EOF

sudo systemctl restart puppetserver
```

### 1.5 Configure autosign (demo only)

```bash
# Allow all node certs to autosign — demo only, NOT for production
echo '*' | sudo tee /etc/puppetlabs/puppet/autosign.conf
sudo systemctl restart puppetserver
```

### 1.6 Note your EC2 internal hostname

Puppet TLS cert is issued to the **internal hostname**, not the public IP:

```bash
sudo openssl x509 -in /etc/puppetlabs/puppet/ssl/ca/ca_crt.pem \
  -text -noout | grep "Subject:"
# Example: CN = Puppet CA: ip-172-31-17-41.us-east-2.compute.internal
```

Save this — nodes need it for TLS verification.

### 1.7 GitOps: update node config

```bash
# On EC2 — pull latest config from GitHub and redeploy
sudo git -C /etc/puppetlabs/code/hieradata pull
sudo r10k deploy environment master -v
```

> **GitOps flow:** Edit `data/global.yaml` or `agents/<node>.yaml` →
> push to GitHub → run the two commands above on EC2 →
> nodes pick up changes on next puppet run (every 30 min).

---

## Part 2 — Hetzner VMs

### 2.1 Create VMs in Hetzner Cloud console

| Name | Type | vCPU | RAM | OS | Role |
|------|------|------|-----|----|------|
| node-1 | CX32 | 4 | 8 GB | Ubuntu 24.04 | K8s control plane |
| node-2 | CX22 | 2 | 4 GB | Ubuntu 24.04 | K8s worker |

- Location: Nuremberg (nbg1)
- SSH key:
  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/hetzner -N ""
  cat ~/.ssh/hetzner.pub  # paste into Hetzner UI
  ```

### 2.2 Fix SSH known_hosts when recreating servers

```bash
ssh-keygen -f ~/.ssh/known_hosts -R '<SERVER-IP>'
ssh -i ~/.ssh/hetzner -o StrictHostKeyChecking=no root@<SERVER-IP>
```

---

## Part 3 — Install LinuxAid on each node

Run on **node-1** and **node-2** (change `--certname` for each).

```bash
ssh -i ~/.ssh/hetzner root@<NODE-IP>

# 1. Install prereqs
sudo apt update && sudo apt install -y jq

# 2. Install LinuxAid CLI
curl -sSL https://raw.githubusercontent.com/Obmondo/linuxaid-cli/main/install.sh | bash

# 3. Install openvox-agent (provides the puppet binary)
wget https://apt.voxpupuli.org/openvox8-release-ubuntu24.04.deb
sudo dpkg -i openvox8-release-ubuntu24.04.deb
sudo apt update && sudo apt install -y openvox-agent

# 4. Map EC2 internal hostname → public IP for TLS verification
echo "<EC2-PUBLIC-IP>  <EC2-INTERNAL-HOSTNAME>" | sudo tee -a /etc/hosts
# e.g: echo "3.143.246.44  ip-172-31-17-41.us-east-2.compute.internal" | sudo tee -a /etc/hosts

# 5. Configure puppet.conf
sudo tee /etc/puppetlabs/puppet/puppet.conf <<EOF
[main]
certname = node-1.demo
server = <EC2-INTERNAL-HOSTNAME>
masterport = 8140
EOF

# 6. Run linuxaid-install (pass IP:port — avoids port 443 default check)
linuxaid-install \
  --certname node-1.demo \
  --puppet-server <EC2-PUBLIC-IP>:8140

# 7. First puppet run (use master environment)
sudo /opt/puppetlabs/bin/puppet agent --test \
  --server <EC2-INTERNAL-HOSTNAME> \
  --masterport 8140 \
  --environment master \
  --no-daemonize
```

> **What LinuxAid deploys on the node (via `role::basic`):**
> - SSH hardening
> - Firewall rules
> - NTP sync
> - `node_exporter` on `:9100` — CPU, RAM, disk, network
> - `iptables_exporter` — firewall metrics
> - `systemd_exporter` — service health metrics
> - `process_exporter` — per-process metrics

### 3.1 Verify node registered (on EC2)

```bash
echo "127.0.0.1  puppet" | sudo tee -a /etc/hosts
sudo /opt/puppetlabs/bin/puppetserver ca list --all
# Should show: node-1.demo  node-2.demo
```

### 3.2 Verify exporters running (on node)

```bash
curl http://localhost:9100/metrics | grep node_cpu
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
# Enter token when prompted

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

juju status --watch 5s

# Once first unit is active, add worker
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

## Part 7 — KubeAid + Prometheus + Grafana

KubeAid deploys `prometheus-linuxaid` which scrapes all node exporters.

```bash
# Install ArgoCD via KubeAid
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/Obmondo/kubeaid/master/argocd-helm-charts/argo-cd/crds.yaml

# Add prometheus-linuxaid app to ArgoCD
# (configure scrape targets pointing to node-1:9100 and node-2:9100)
```

### 7.1 Access Grafana

| Service | URL |
|---------|-----|
| Grafana | `http://grafana-linuxaid.<your-domain>` |
| Prometheus | `http://prometheus-linuxaid.<your-domain>` |
| AlertManager | `http://alertmanager-linuxaid.<your-domain>` |

Default credentials:
- **User:** `root`
- **Password:** `secretroot`

---

## Part 8 — Deploy test workload

```bash
kubectl create deployment nginx --image=nginx --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods -o wide
kubectl get svc
```

---

## Repo Structure (hiera config)

```
MAVRICK-1/juju-linuxaid-demo/
├── data/
│   └── global.yaml          # shared config for ALL nodes
└── agents/
    ├── node-1.demo.yaml      # node-1 specific: classes + overrides
    └── node-2.demo.yaml      # node-2 specific: classes + overrides
```

### data/global.yaml

```yaml
---
common::monitor::exporter::enable: true
common::monitor::exporter::node::enable: true       # CPU/RAM/disk on :9100
common::monitor::exporter::iptables::enable: true
common::monitor::exporter::systemd::enable: true
common::monitor::exporter::process::enable: true
common::system::time::manage_ntp: true
```

### agents/node-1.demo.yaml

```yaml
---
classes:
  - role::basic
common::monitor::exporter::node::enable: true
common::monitor::exporter::iptables::enable: true
```

---

## Cost Estimate (7 days)

| Resource | Type | Cost/hr | 7 days |
|----------|------|---------|--------|
| EC2 t3.medium | AWS | ~$0.047/hr | ~$7.90 |
| Hetzner CX32 (node-1) | Hetzner | €0.013/hr | ~€2.18 |
| Hetzner CX22 (node-2) | Hetzner | €0.007/hr | ~€1.18 |
| **Total** | | | **~$12** |

---

## Cleanup

```bash
# Juju
juju destroy-model k8s-demo --force --no-wait

# Delete Hetzner servers from Hetzner console
# Terminate EC2 instance from AWS console
```

---

## Known Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| `openvox-server.service not found` | Service is named `puppetserver` | Use `systemctl start puppetserver` |
| `linuxaid-install` hits port 443 | Hardcoded in CLI | Pass `IP:8140` as `--puppet-server` |
| `hostname does not match server cert` | Cert issued to internal hostname | Add `/etc/hosts` entry mapping IP → internal hostname |
| `r10k deploy production` fails | LinuxAid has no `production` branch | Deploy `master` branch instead |
| `Permission denied on environments/` | Wrong ownership | `chown -R puppet:puppet /etc/puppetlabs/code/environments` |
| Catalog applied in 0.01s (empty) | No hiera data / wrong environment | Use `--environment master`, set up hiera.yaml |

---

## References

| Resource | URL |
|----------|-----|
| LinuxAid | https://github.com/Obmondo/LinuxAid |
| LinuxAid CLI | https://github.com/Obmondo/linuxaid-cli |
| LinuxAid Docs | https://linuxaid.io/docs |
| OpenVox | https://voxpupuli.org/openvox |
| Juju | https://juju.is/docs |
| MicroK8s Charm | https://charmhub.io/microk8s |
| KubeAid | https://github.com/Obmondo/kubeaid |
| prometheus-linuxaid | https://github.com/Obmondo/kubeaid/tree/master/argocd-helm-charts/prometheus-linuxaid |
