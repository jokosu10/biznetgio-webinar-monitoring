# Requirements untuk Rebuild Monitoring Stack

**Document Version:** 1.1
**Last Updated:** 2025-11-05
**Purpose:** Checklist lengkap untuk rebuild biznetgio-webinar-monitoring dari awal

---

## 🚀 Deployment Scenarios

### Recommended: 2 VM Setup (Production-Lite)
**Best for:** Production-lite, Development, Webinar Demo

✅ **See detailed guide:** [`DEPLOYMENT_2VM.md`](DEPLOYMENT_2VM.md)

```
VM1: Monitoring Server (Prometheus + Grafana)
VM2: Kubernetes Server (K3s + kube-prometheus-stack)

Cost: ~Rp 300,000 - 400,000/bulan
Setup Time: 4-6 hours
```

**Advantages:**
- ✅ Separation of concerns (monitoring vs workload)
- ✅ Better resource management
- ✅ Easier to scale independently
- ✅ Production-ready architecture

---

### Alternative Scenarios

**Option 1: Single VM All-in-One** (Testing/Demo Only)
```
1 VM (4 vCPU, 8 GB RAM, 100 GB disk)
- K3s + Prometheus + Grafana
Cost: ~Rp 200,000/bulan
⚠️ Not recommended for production (single point of failure)
```

**Option 2: Existing Kubernetes Cluster**
```
1 VM for Monitoring Server only
- Connect to your existing K8s cluster
Cost: ~Rp 150,000 - 200,000/bulan
✅ Ideal if you already have K8s infrastructure
```

**Option 3: Full Production Setup** (5+ VMs)
```
1 VM: Monitoring Server (HA)
3-4 VMs: Kubernetes Cluster (multi-node HA)
Cost: ~Rp 800,000 - 1,500,000/bulan
✅ Enterprise-grade with high availability
```

---

## 📋 Quick Checklist (2 VM Setup)

```
Infrastructure:
☐ VM1: Ubuntu 22.04 Server (Monitoring Server)
   - 2 vCPU, 4 GB RAM, 60 GB disk minimum
   - Public IP untuk akses Grafana
   - Private IP untuk komunikasi dengan K8s
☐ VM2: Ubuntu 22.04 Server (Kubernetes Server)
   - 2 vCPU, 4 GB RAM, 60 GB disk minimum
   - Public/Private IP
☐ Network connectivity: VM2 → VM1 port 9090

Akses:
☐ Root/sudo access ke kedua VM
☐ SSH key configured
☐ Password manager untuk store credentials

Software (VM1):
☐ Prometheus v2.53.4
☐ Grafana latest
☐ curl, wget, tar, vim

Software (VM2):
☐ K3s (installed via script)
☐ Helm 3.x
☐ kubectl (via k3s)

Persiapan:
☐ Strong passwords generated (Prometheus, Grafana)
☐ Bcrypt hash created untuk Prometheus auth
☐ Domain/DNS (optional, untuk TLS)
☐ Firewall rules planned

Time Required:
☐ Phase 1: Provisioning (30 min)
☐ Phase 2: VM1 Setup (2-3 hours)
☐ Phase 3: VM2 Setup (2-3 hours)
☐ Phase 4: Dashboard Import (30 min)
──────────────────────────────
Total: 4-6 hours
```

---

## 🖥️ 1. Infrastructure Requirements

### A. Monitoring Server (Prometheus + Grafana)

**Minimum Specs:**
```yaml
OS: Ubuntu 22.04 LTS (64-bit)
CPU: 2 vCPU
RAM: 4 GB
Disk: 60 GB
Network: Public IP dengan bandwidth stabil
```

**Recommended Specs (Production):**
```yaml
OS: Ubuntu 22.04 LTS (64-bit)
CPU: 4 vCPU
RAM: 8 GB
Disk: 200 GB SSD (untuk TSDB performance)
Network: Public IP + Private network
Backup: Additional volume untuk backup storage
```

**Disk Breakdown:**
```
/                    : 20 GB  (OS + applications)
/var/lib/prometheus_data : 150 GB (Time-series data)
/backup             : 50 GB  (Backup storage)
```

**Storage Calculation:**
```
Retention: 30 days
Samples/sec: 10,000
Sample size: ~2 bytes
Estimated: (10000 * 2 * 86400 * 30) / 1024^3 ≈ 52 GB
With overhead: ~80-100 GB recommended
```

---

### B. Kubernetes Cluster

**Minimum Cluster Specs:**
```yaml
Kubernetes Version: 1.24+
Nodes: 1+ worker nodes
Total CPU: 4 cores (2 untuk monitoring stack)
Total RAM: 8 GB (4 GB untuk monitoring stack)
Storage: 100 GB untuk Prometheus PVC
```

**kube-prometheus-stack Resource Usage:**
```yaml
Prometheus:
  CPU: 500m - 2 cores
  Memory: 2-4 GB
  Storage: 50-100 GB

Kube-state-metrics:
  CPU: 100m
  Memory: 128 MB

Node-exporter (per node):
  CPU: 100m
  Memory: 128 MB
```

**StorageClass Requirements:**
```yaml
# Harus ada StorageClass yang support ReadWriteOnce
kubectl get storageclass

# Example:
NAME                 PROVISIONER
standard (default)   kubernetes.io/gce-pd
fast-ssd            kubernetes.io/gce-pd
```

---

### C. External Nodes (Optional)

Jika ingin monitor external servers (di luar K8s):

```yaml
Per Node:
  OS: Linux (Ubuntu, CentOS, RHEL, etc.)
  CPU: +100m (untuk node-exporter)
  RAM: +128 MB (untuk node-exporter)
  Port 9100: Harus accessible dari Kubernetes cluster
```

---

## 🌐 2. Network Requirements

### Port Requirements

**Monitoring Server:**
```
Inbound:
  22/tcp   : SSH access
  9090/tcp : Prometheus API (dari K8s cluster)
  3000/tcp : Grafana Web UI (dari admin workstation)

Outbound:
  80/tcp   : Package downloads
  443/tcp  : HTTPS downloads
```

**Kubernetes Cluster:**
```
Inbound:
  Tidak perlu port terbuka ke internet

Outbound:
  9090/tcp : Ke monitoring server (remote write)
  80/tcp   : Helm chart downloads
  443/tcp  : Container image pulls
```

**External Nodes (jika ada):**
```
Inbound:
  9100/tcp : Node exporter metrics (dari K8s cluster)
  22/tcp   : SSH untuk maintenance
```

---

### Network Connectivity Matrix

```
┌─────────────────┬──────────────┬───────────────┬──────────────┐
│ From/To         │ Mon. Server  │ K8s Cluster   │ Ext. Nodes   │
├─────────────────┼──────────────┼───────────────┼──────────────┤
│ Admin Workstation│ :22, :3000  │ :6443         │ :22          │
│ K8s Cluster     │ :9090        │ -             │ :9100        │
│ Mon. Server     │ -            │ NOT REQUIRED  │ NOT REQUIRED │
└─────────────────┴──────────────┴───────────────┴──────────────┘
```

**Connectivity Tests:**
```bash
# Dari K8s node ke monitoring server
telnet <monitoring-server-ip> 9090
# atau
nc -zv <monitoring-server-ip> 9090

# Dari K8s node ke external node
telnet <external-node-ip> 9100

# Dari workstation ke Grafana
curl -I http://<monitoring-server-ip>:3000
```

---

### DNS Requirements (Optional tapi Recommended)

Untuk TLS/HTTPS setup:
```
monitoring.example.com  → Monitoring server IP
grafana.example.com     → Monitoring server IP (or same)
```

---

### Firewall Rules

**UFW (Ubuntu):**
```bash
# Monitoring Server
ufw allow 22/tcp                    # SSH
ufw allow from <k8s-cidr> to any port 9090  # Prometheus
ufw allow from <office-ip> to any port 3000 # Grafana
ufw enable

# External Nodes
ufw allow 22/tcp
ufw allow from <k8s-cidr> to any port 9100
ufw enable
```

**Cloud Provider Security Groups:**
```yaml
# Monitoring Server Security Group
Inbound:
  - Port 22: Source = Your IP
  - Port 9090: Source = K8s Node IPs or CIDR
  - Port 3000: Source = Office IP range

# Kubernetes Node Security Group
Outbound:
  - Port 9090: Destination = Monitoring Server IP
  - Port 443: Destination = 0.0.0.0/0 (internet)
```

---

## 💻 3. Software Prerequisites

### A. Monitoring Server

**OS Packages:**
```bash
# Required
apt-transport-https
software-properties-common
wget
curl
tar
gpg

# Recommended
vim / nano (text editor)
htop (monitoring)
ncdu (disk usage)
net-tools (networking tools)
```

**Installation Commands:**
```bash
sudo apt update
sudo apt install -y \
    apt-transport-https \
    software-properties-common \
    wget \
    curl \
    tar \
    gpg \
    vim \
    htop \
    ncdu \
    net-tools
```

---

### B. Kubernetes Cluster Setup

**kubectl:**
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client

# Configure (copy kubeconfig)
mkdir -p ~/.kube
# Copy kubeconfig dari cluster ke ~/.kube/config

# Test connection
kubectl get nodes
kubectl get pods -A
```

**Helm 3:**
```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version

# Add repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Verify repos
helm repo list
```

**Required Helm Repos:**
```
NAME                    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
grafana                 https://grafana.github.io/helm-charts (optional, untuk Loki)
```

---

### C. Admin Workstation

**Untuk administrator yang setup:**
```bash
# Local machine tools
- SSH client
- kubectl (configured)
- helm (optional, untuk debugging)
- Web browser (untuk akses Grafana)
- Git (untuk clone repository)
```

---

## 🔐 4. Credentials & Security

### A. Credentials yang Perlu Disiapkan

**1. Prometheus Basic Auth:**
```bash
# Generate strong password
PASSWORD=$(openssl rand -base64 32)
echo "Prometheus Password: $PASSWORD"
# Simpan di password manager!

# Generate bcrypt hash (gunakan https://bcrypt.online/ atau htpasswd)
# Rounds: 12 atau lebih
htpasswd -nBC 12 admin
# Atau gunakan online tool dengan Cost Factor = 12
```

**2. Grafana Admin:**
```bash
# Initial: admin/admin
# Ganti immediately setelah first login
# Gunakan password kuat minimal 16 karakter
NEW_GRAFANA_PASS=$(openssl rand -base64 24)
echo "Grafana Password: $NEW_GRAFANA_PASS"
```

**3. SSH Keys:**
```bash
# Generate SSH key (jika belum ada)
ssh-keygen -t ed25519 -C "monitoring-server-access"

# Copy ke monitoring server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@<monitoring-server-ip>
```

**4. Kubernetes Secret (untuk remote write):**
```bash
# Buat secret untuk basic auth
PROM_USER="admin"
PROM_PASS="<password-yang-sama-dengan-prometheus>"

kubectl create ns monitoring
kubectl create secret generic kubepromsecret \
    --from-literal=username=$PROM_USER \
    --from-literal=password=$PROM_PASS \
    -n monitoring

# Verify
kubectl get secret kubepromsecret -n monitoring -o yaml
```

---

### B. TLS Certificates (Recommended)

**Option 1: Let's Encrypt (Gratis, Recommended):**
```bash
# Prerequisites:
- Domain name pointing to server
- Port 80 accessible dari internet
- Certbot installed

# Install certbot
sudo apt install certbot python3-certbot-nginx

# Generate certificate
sudo certbot certonly --standalone -d monitoring.example.com

# Certificates location:
# /etc/letsencrypt/live/monitoring.example.com/fullchain.pem
# /etc/letsencrypt/live/monitoring.example.com/privkey.pem
```

**Option 2: Self-Signed (Development):**
```bash
# Generate self-signed cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /opt/prometheus/server.key \
    -out /opt/prometheus/server.crt \
    -subj "/CN=monitoring.example.com"

# Set permissions
chown prometheus:prometheus /opt/prometheus/server.{key,crt}
chmod 600 /opt/prometheus/server.key
```

**Option 3: Corporate CA:**
```
Request certificate dari internal CA team
- CN: monitoring.example.com
- SAN: grafana.example.com (optional)
- Validity: 1-2 years
```

---

## 📦 5. Binary Files & Versions

### A. Version Compatibility Matrix

```yaml
Prometheus Server: v2.53.4
Grafana: v10.4.1 (or latest stable)
kube-prometheus-stack: v58.2.2
node-exporter: v1.8.2
kubectl: v1.28+ (match with cluster version)
helm: v3.12+
```

### B. Download Links

**Prometheus:**
```bash
# x86_64
https://github.com/prometheus/prometheus/releases/download/v2.53.4/prometheus-2.53.4.linux-amd64.tar.gz

# ARM64 (jika perlu)
https://github.com/prometheus/prometheus/releases/download/v2.53.4/prometheus-2.53.4.linux-arm64.tar.gz

# Checksum verification
wget https://github.com/prometheus/prometheus/releases/download/v2.53.4/sha256sums.txt
sha256sum -c sha256sums.txt 2>&1 | grep prometheus-2.53.4.linux-amd64.tar.gz
```

**Node Exporter:**
```bash
https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
```

**Grafana:**
```bash
# Via APT repository (recommended)
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Direct download (alternative)
https://dl.grafana.com/oss/release/grafana_10.4.1_amd64.deb
```

---

## 📁 6. Configuration Files Needed

### A. Files yang Harus Disiapkan

**1. Prometheus Configuration:**
```yaml
# /opt/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    basic_auth:
      username: 'admin'
      password: '<YOUR_PASSWORD>'
    static_configs:
      - targets: ['localhost:9090']
```

**2. Prometheus Auth File:**
```yaml
# /opt/prometheus/auth.yml
basic_auth_users:
  admin: <BCRYPT_HASH_OF_PASSWORD>
```

**3. Prometheus Systemd Service:**
```ini
# /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus_data/ \
  --storage.tsdb.retention.time=30d \
  --web.enable-remote-write-receiver \
  --web.config.file=/opt/prometheus/auth.yml

[Install]
WantedBy=multi-user.target
```

**4. Helm Values (kube-prometheus-stack):**
```yaml
# values.yaml
prometheus:
  prometheusSpec:
    scrapeInterval: "1m"
    retention: 15d
    remoteWrite:
    - url: http://<MONITORING_SERVER_IP>:9090/api/v1/write
      basicAuth:
        username:
          name: kubepromsecret
          key: username
        password:
          name: kubepromsecret
          key: password
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 4Gi
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi

grafana:
  enabled: false

alertmanager:
  enabled: false
```

**5. External Node Configuration (Optional):**
```yaml
# external-node.yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: "external-node-exporter"
        static_configs:
          - targets:
            - "10.10.10.5:9100"
            - "10.10.10.6:9100"
            labels:
              env: "production"
```

---

### B. Grafana Dashboards

**Required Dashboard Files:**
```
dashboard/
├── k8s-api-server.json         # Import via Grafana UI
├── k8s-clusters.json           # atau provisioning
├── k8s-namespaces.json
├── k8s-nodes.json
├── k8s-pods.json
└── node-exporter-full.json
```

**Import Methods:**
```bash
# Method 1: Manual Import via UI
# Grafana → Dashboards → Import → Upload JSON file

# Method 2: Provisioning (Automated)
# Copy to: /etc/grafana/provisioning/dashboards/
sudo mkdir -p /etc/grafana/provisioning/dashboards/
sudo cp dashboard/*.json /etc/grafana/provisioning/dashboards/
sudo chown -R grafana:grafana /etc/grafana/provisioning/
sudo systemctl restart grafana-server
```

---

## 🛠️ 7. Tools & Utilities

### A. Monitoring Tools (untuk troubleshooting)

```bash
# System monitoring
htop          # CPU/Memory real-time
iotop         # Disk I/O
nethogs       # Network bandwidth per process
ncdu          # Disk usage analyzer

# Network tools
netstat       # Network connections
ss            # Socket statistics
tcpdump       # Packet capture
curl/wget     # HTTP testing

# Install semua
sudo apt install -y htop iotop nethogs ncdu net-tools tcpdump
```

### B. Prometheus Tools

```bash
# promtool (included dengan Prometheus)
/opt/prometheus/promtool check config /opt/prometheus/prometheus.yml
/opt/prometheus/promtool check web-config /opt/prometheus/auth.yml
/opt/prometheus/promtool tsdb analyze /var/lib/prometheus_data/
```

### C. Kubernetes Tools (Optional)

```bash
# k9s - Terminal UI untuk Kubernetes
curl -sS https://webinstall.dev/k9s | bash

# kubectx/kubens - Context switching
sudo apt install kubectx

# stern - Multi-pod log tailing
wget https://github.com/stern/stern/releases/download/v1.28.0/stern_1.28.0_linux_amd64.tar.gz
tar xzf stern_1.28.0_linux_amd64.tar.gz
sudo mv stern /usr/local/bin/

# Usage
stern prometheus -n monitoring
```

---

## 📝 8. Documentation & References

### A. Required Reading

**Before Starting:**
- [ ] Prometheus documentation: https://prometheus.io/docs/introduction/overview/
- [ ] Grafana getting started: https://grafana.com/docs/grafana/latest/getting-started/
- [ ] kube-prometheus-stack: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

**Architecture Understanding:**
- [ ] Remote write: https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write
- [ ] Storage: https://prometheus.io/docs/prometheus/latest/storage/
- [ ] Best practices: https://prometheus.io/docs/practices/

---

### B. Repository Files

**Clone Repository:**
```bash
git clone https://github.com/jokosu10/biznetgio-webinar-monitoring.git
cd biznetgio-webinar-monitoring
```

**Files to Review:**
```
README.md           # Main installation guide
CODE_REVIEW.md      # Security & best practices review
REQUIREMENTS.md     # This file - prerequisites
dashboard/*.json    # Dashboard definitions
```

---

## ⏱️ 9. Time Estimation

### Setup Timeline

**Day 1: Infrastructure Setup (2-4 hours)**
```
☐ Provision monitoring server (1 hour)
☐ Basic OS configuration (1 hour)
☐ Install Prometheus (1 hour)
☐ Install Grafana (30 min)
☐ Basic testing (30 min)
```

**Day 2: Kubernetes Integration (2-3 hours)**
```
☐ Install Helm (15 min)
☐ Deploy kube-prometheus-stack (30 min)
☐ Configure remote write (30 min)
☐ Verify metrics flow (1 hour)
☐ Import dashboards (30 min)
```

**Day 3: Hardening & Optimization (3-4 hours)**
```
☐ Configure firewall (30 min)
☐ Setup TLS (1 hour)
☐ Configure alerting (1 hour)
☐ Setup backup strategy (1 hour)
☐ Documentation (30 min)
```

**Total Time:** 7-11 hours untuk complete setup

**Quick Setup (Minimal):** 3-4 hours (tanpa TLS, alerting, backup)

---

## ✅ 10. Pre-Deployment Checklist

### Final Checks Before Starting

**Infrastructure:**
```
☐ Server provisioned dan accessible via SSH
☐ Server timezone set ke Asia/Jakarta (atau sesuai lokasi)
☐ System packages updated (apt update && apt upgrade)
☐ Hostname set (hostnamectl set-hostname monitoring)
☐ Disk space verified (df -h)
☐ Network connectivity tested (ping 8.8.8.8)
```

**Access & Permissions:**
```
☐ Root/sudo access confirmed
☐ SSH key-based authentication setup
☐ Kubectl access to Kubernetes cluster verified
☐ Helm installed dan tested
```

**Security Preparation:**
```
☐ Strong passwords generated dan stored securely
☐ Bcrypt hashes created untuk Prometheus auth
☐ TLS certificates obtained (jika applicable)
☐ Firewall rules planned
```

**Configuration Files:**
```
☐ prometheus.yml prepared
☐ auth.yml prepared
☐ values.yaml prepared
☐ external-node.yaml prepared (jika ada external nodes)
```

**Kubernetes Ready:**
```
☐ Namespace 'monitoring' can be created
☐ StorageClass available
☐ Sufficient cluster resources (4 CPU, 8GB RAM available)
☐ Network policies tidak block monitoring
```

---

## 🚨 11. Common Pitfalls to Avoid

### A. Before You Start

**❌ JANGAN:**
1. ❌ Gunakan password default "admin"/"password"
2. ❌ Skip firewall configuration
3. ❌ Install tanpa TLS di production
4. ❌ Lupa backup config files
5. ❌ Ignore resource limits di Kubernetes
6. ❌ Commit credentials ke Git

**✅ LAKUKAN:**
1. ✅ Generate strong passwords (>16 chars)
2. ✅ Configure firewall rules properly
3. ✅ Setup TLS untuk production
4. ✅ Document semua passwords di password manager
5. ✅ Set resource requests/limits
6. ✅ Use secrets untuk credentials

---

### B. During Installation

**Perhatikan:**
```
⚠️ Disk space: Monitor /var/lib/prometheus_data/
⚠️ Permissions: Prometheus user harus owner data directory
⚠️ Time sync: NTP harus configured (timedatectl)
⚠️ Firewall: Test connectivity sebelum troubleshoot lebih jauh
⚠️ Retention: Sesuaikan dengan disk space available
```

---

## 📞 12. Support Resources

### Official Documentation
- Prometheus: https://prometheus.io/docs/
- Grafana: https://grafana.com/docs/
- Kubernetes: https://kubernetes.io/docs/

### Community Resources
- Prometheus Community Forums: https://prometheus.io/community/
- Grafana Community: https://community.grafana.com/
- CNCF Slack: https://slack.cncf.io/ (#prometheus, #grafana)

### Troubleshooting Resources
- Prometheus Troubleshooting: https://prometheus.io/docs/prometheus/latest/troubleshooting/
- Grafana Troubleshooting: https://grafana.com/docs/grafana/latest/troubleshooting/

---

## 🎯 Quick Start Command Reference

```bash
# === MONITORING SERVER SETUP ===

# 1. Update system
sudo apt update && sudo apt upgrade -y

# 2. Create prometheus user
sudo useradd -M -U prometheus
sudo mkdir -p /var/lib/prometheus_data/
sudo chown prometheus:prometheus -R /var/lib/prometheus_data/

# 3. Download & install Prometheus
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.53.4/prometheus-2.53.4.linux-amd64.tar.gz
tar -zxvf prometheus-2.53.4.linux-amd64.tar.gz
sudo mv prometheus-2.53.4.linux-amd64 /opt/prometheus
sudo chown prometheus:prometheus -R /opt/prometheus/

# 4. Configure auth (prepare auth.yml first)
/opt/prometheus/promtool check web-config /opt/prometheus/auth.yml

# 5. Create & start service
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus.service

# 6. Install Grafana
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana
sudo systemctl enable --now grafana-server

# === KUBERNETES SETUP ===

# 1. Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 2. Create namespace & secret
kubectl create ns monitoring
kubectl create secret generic kubepromsecret \
    --from-literal=username=admin \
    --from-literal=password=<YOUR_PASSWORD> \
    -n monitoring

# 3. Install kube-prometheus-stack
helm install -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --version 58.2.2 \
    -f values.yaml

# 4. Verify
kubectl get pods -n monitoring
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0
```

---

## 📋 Summary

**Must Have:**
- ✅ Ubuntu 22.04 Server (2 CPU, 4 GB RAM, 60 GB disk)
- ✅ Kubernetes cluster (1.24+) dengan kubectl access
- ✅ Helm 3.x installed
- ✅ Network connectivity (K8s → Monitoring server port 9090)
- ✅ Strong credentials prepared

**Nice to Have:**
- 💡 Domain name (untuk TLS)
- 💡 External nodes untuk monitoring
- 💡 Backup storage
- 💡 CI/CD pipeline (untuk automated deployment)

**Total Setup Time:** 7-11 hours (complete), 3-4 hours (minimal)

**Estimated Cost (Biznet Gio Neo Lite):**
- MS 4.2 VPS: ~Rp 150,000 - 200,000/month
- Kubernetes cluster: Varies (managed K8s or self-hosted)

---

**Document Prepared By:** Claude AI
**Last Review:** 2025-11-05
**Next Review:** Quarterly or when major version changes
