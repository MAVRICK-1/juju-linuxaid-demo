# LinuxAid + Juju + MicroK8s Demo

Full stack demo: LinuxAid manages the OS, Juju deploys Kubernetes on top.

```
EC2 (OpenVox Server)
        ↓  signs certs + delivers config
Hetzner node-1  <--  LinuxAid (OS hardened)  <--  Juju (k8s control plane)
Hetzner node-2  <--  LinuxAid (OS hardened)  <--  Juju (k8s worker)
```

---

## Part 1 — OpenVox Server on EC2

### 1.1 Launch EC2 instance

- AMI: Ubuntu 24.04 LTS
- Instance type: t3.medium (2 vCPU, 4GB RAM)
- Security group — open inbound ports:
  - `8140/tcp` — OpenVox agent communication (puppet catalog + cert signing)
  - `443/tcp` — OpenVox status API check by linuxaid-install
  - `22/tcp` — SSH

### 1.2 Install OpenVox Server

```bash
# SSH into EC2
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>

# Add OpenVox repo (note: no hyphen before ubuntu)
wget https://apt.voxpupuli.org/openvox8-release-ubuntu24.04.deb
sudo dpkg -i openvox8-release-ubuntu24.04.deb
sudo apt update

# Install OpenVox Server
sudo apt install -y openvox-server

# Start and enable (service is named puppetserver, not openvox-server)
sudo systemctl start puppetserver
sudo systemctl enable puppetserver

# Verify
sudo systemctl status puppetserver
```

### 1.3 Deploy LinuxAid code via r10k

**Step 1: Install r10k**

```bash
sudo apt install -y git ruby ruby-dev
sudo gem install r10k
```

**Step 2: Configure r10k**

```bash
sudo mkdir -p /etc/puppetlabs/r10k
sudo tee /etc/puppetlabs/r10k/r10k.yaml <<EOF
cachedir: /var/cache/r10k
sources:
  linuxaid:
    remote: https://github.com/MAVRICK-1/juju-linuxaid-demo.git
    basedir: /etc/puppetlabs/code/environments
EOF
```

**Step 3: Deploy the environment**

```bash
# This pulls the `production` branch and installs all modules in Puppetfile
sudo r10k deploy environment production -v
```

**Step 4: Configure global.yaml for your nodes**

`global.yaml` is the top-level config that controls which LinuxAid modules are enabled across all nodes.

```bash
sudo cat /etc/puppetlabs/code/environments/production/data/global.yaml
```

Minimal working config — edit as needed:

```yaml
# /etc/puppetlabs/code/environments/production/data/global.yaml
---
linuxaid::manage_puppet: true
linuxaid::manage_firewall: true
linuxaid::manage_ssh: true
linuxaid::manage_ntp: true
linuxaid::exporters::node: true
```

**Step 5: Verify modules loaded**

```bash
sudo ls /etc/puppetlabs/code/environments/production/modules/
```

> **GitOps flow:** Edit `data/global.yaml` or node configs in `MAVRICK-1/juju-linuxaid-demo` → push to GitHub → run `sudo r10k deploy environment production -v` on EC2 → changes applied on next puppet run on nodes (default every 30 min).

### 1.4 Configure autosign (demo only)

```bash
# Allow all certs to autosign — demo only, NOT for production
echo '*' | sudo tee /etc/puppetlabs/puppet/autosign.conf

# Restart server
sudo systemctl restart puppetserver
```

---

## Part 2 — Hetzner VMs

### 2.1 Create VMs in Hetzner Cloud console

| Name | Type | OS | Role |
|------|------|----|------|
| node-1 | CX32 | Ubuntu 24.04 | k8s control plane |
| node-2 | CX22 | Ubuntu 24.04 | k8s worker |

- Location: Nuremberg (nbg1)
- Add your SSH public key:
  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/hetzner -N ""
  cat ~/.ssh/hetzner.pub  # paste this in Hetzner UI
  ```
- Enable private network: create `demo-net`

### 2.2 Note the IPs

```
node-1: <NODE1-PUBLIC-IP>
node-2: <NODE2-PUBLIC-IP>
```

---

## Part 3 — Install LinuxAid on each node

Run on **both node-1 and node-2** (change `--certname` for each):

```bash
# SSH into node
ssh -i ~/.ssh/hetzner root@<NODE-IP>

# Install LinuxAid CLI
# jq is required — install it first
sudo apt update && sudo apt install -y jq
curl -sSL https://raw.githubusercontent.com/Obmondo/linuxaid-cli/main/install.sh | bash

# Run linuxaid-install pointing to your EC2 OpenVox server
linuxaid-install \
  --certname node-1.demo \
  --puppet-server <EC2-PUBLIC-IP>

# First puppet run
linuxaid-cli run-openvox --certname node-1.demo
```

> **What LinuxAid does on the node:**
> - Installs and configures openvox-agent
> - Hardens SSH configuration
> - Sets up firewall rules
> - Deploys 24 Prometheus exporters
> - Manages package updates via service windows
> - Enforces CIS/GDPR/NIS2 compliance defaults

### 3.1 Verify registered nodes (on EC2)

```bash
sudo puppetserver ca list --all
# Should show node-1.demo and node-2.demo
```

---

## Part 4 — Install Juju on node-1

```bash
# SSH into node-1
ssh -i ~/.ssh/hetzner root@<NODE1-PUBLIC-IP>

# Install Juju
snap install juju --classic

# Create required directory
mkdir -p ~/.local/share/juju
```

---

## Part 5 — Bootstrap Juju controller

```bash
# Get Hetzner API token from:
# https://console.hetzner.cloud -> Project -> Security -> API Tokens -> Generate

# Add Hetzner credentials
juju add-credential hetzner
# Enter token when prompted

# Bootstrap Juju controller
juju bootstrap hetzner hetzner-controller

# Verify
juju status
```

---

## Part 6 — Deploy MicroK8s (multi-node)

```bash
# Create k8s model
juju add-model k8s-demo

# Deploy control plane
juju deploy microk8s \
  --channel 1.28/stable \
  --base ubuntu@22.04

# Watch it come up
juju status --watch 5s

# Once first unit is active, add worker
juju add-unit microk8s

# Watch worker join
juju status --watch 5s
```

Wait for all units to show `active`.

### 6.1 Access the cluster

```bash
mkdir -p ~/.kube
juju ssh microk8s/0 -- sudo microk8s config > ~/.kube/config

# Verify both nodes
kubectl get nodes
```

Expected:
```
NAME           STATUS   ROLES    AGE
juju-xxxx-0    Ready    <none>   5m
juju-xxxx-1    Ready    <none>   3m
```

---

## Part 7 — Deploy test workload

```bash
kubectl create deployment nginx --image=nginx --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods -o wide
kubectl get svc
```

---

## Full Architecture

```
                    +---------------------+
                    |   EC2 t3.medium     |
                    |   OpenVox Server    |
                    |   port 8140         |
                    +----------+----------+
                               |
               +---------------+---------------+
               |                               |
    +----------+----------+       +------------+--------+
    |   Hetzner node-1    |       |   Hetzner node-2    |
    |   CX32 (4 vCPU 8GB) |       |   CX22 (2 vCPU 4GB) |
    |                     |       |                     |
    |   LinuxAid          |       |   LinuxAid          |
    |   (OS hardening)    |       |   (OS hardening)    |
    |                     |       |                     |
    |   Juju agent        |       |   Juju agent        |
    |   MicroK8s CP       |       |   MicroK8s Worker   |
    +---------------------+       +---------------------+
```

---

## Cost Estimate (7 days)

| Resource | Type | Cost/hr | 7 days |
|----------|------|---------|--------|
| EC2 t3.medium | AWS | ~$0.047/hr | ~$7.90 |
| Hetzner node-1 | CX32 | €0.013/hr | ~€2.18 |
| Hetzner node-2 | CX22 | €0.007/hr | ~€1.18 |
| **Total** | | | **~$12** |

---

## Cleanup

```bash
# Destroy Juju model
juju destroy-model k8s-demo --force --no-wait
# Type: k8s-demo

# Delete Hetzner servers from Hetzner console
# Terminate EC2 instance from AWS console
```

---

## References

- LinuxAid: https://github.com/Obmondo/LinuxAid
- LinuxAid CLI: https://github.com/Obmondo/linuxaid-cli
- LinuxAid Docs: https://linuxaid.io/docs
- OpenVox: https://voxpupuli.org/openvox
- Juju: https://juju.is/docs
- MicroK8s Charm: https://charmhub.io/microk8s
