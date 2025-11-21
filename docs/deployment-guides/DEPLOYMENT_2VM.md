# Deployment Guide: 2 VM Setup

**Setup Type:** Production-Lite / Development
**Total VMs:** 2
**Estimated Cost:** Rp 300,000 - 400,000/bulan (Biznet Gio)
**Setup Time:** 4-6 hours

---

## 📋 Overview

Deployment ini menggunakan **2 VM** untuk memisahkan monitoring server dan Kubernetes cluster:

```
┌─────────────────────────────────────────────────────────┐
│           VM2: Kubernetes Cluster (K3s)                 │
│           IP: 10.x.x.2 (private) / Public IP            │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  K3s (Lightweight Kubernetes)                  │    │
│  │  ├── kube-prometheus-stack                     │    │
│  │  ├── Node Exporter                             │    │
│  │  └── Your Applications                         │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  Scrapes metrics from:                                  │
│  • Kubernetes nodes, pods, services                     │
│  • Container metrics                                    │
│  • Application metrics                                  │
└─────────────────────────────────────────────────────────┘
                        │
                        │ Remote Write (HTTP)
                        │ Port 9090
                        ▼
┌─────────────────────────────────────────────────────────┐
│        VM1: Monitoring Server (Ubuntu 22.04)            │
│        IP: 10.x.x.1 (private) / Public IP               │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  Prometheus Server                             │    │
│  │  ├── Receives metrics via remote write         │    │
│  │  ├── Stores time-series data (30 days)         │    │
│  │  ├── Port 9090 (from K8s)                      │    │
│  │  └── Basic authentication enabled              │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  Grafana Server                                │    │
│  │  ├── Visualization dashboards                  │    │
│  │  ├── Port 3000 (public access)                 │    │
│  │  └── 6 pre-configured dashboards               │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                        ▲
                        │ HTTPS/HTTP
                        │ Port 3000
                        │
                 ┌──────────────┐
                 │ Admin/Users  │
                 │ Web Browser  │
                 └──────────────┘
```

---

## 🖥️ VM Specifications

### VM1: Monitoring Server

```yaml
Hostname: monitoring-server
OS: Ubuntu 22.04 LTS
CPU: 2 vCPU (recommended: 4 vCPU untuk production)
RAM: 4 GB (recommended: 8 GB untuk production)
Disk: 60 GB minimum (recommended: 200 GB SSD)
Network:
  - Public IP (untuk akses Grafana)
  - Private IP (untuk komunikasi dengan K8s)
Ports:
  - 22/tcp: SSH
  - 9090/tcp: Prometheus (dari VM2 saja)
  - 3000/tcp: Grafana (dari internet/office IP)
```

**Estimated Cost (Biznet Gio):**
- Flavor MS 4.2: ~Rp 150,000 - 200,000/bulan

---

### VM2: Kubernetes Server (K3s)

```yaml
Hostname: k8s-node
OS: Ubuntu 22.04 LTS
CPU: 2 vCPU (recommended: 4 vCPU)
RAM: 4 GB minimum (recommended: 8 GB)
Disk: 60 GB minimum (recommended: 100 GB)
Network:
  - Public IP (optional, untuk kubectl remote access)
  - Private IP (untuk komunikasi dengan monitoring server)
Ports:
  - 22/tcp: SSH
  - 6443/tcp: Kubernetes API (optional, untuk remote kubectl)
  - 80/tcp: Ingress (optional)
  - 443/tcp: Ingress HTTPS (optional)
```

**Estimated Cost (Biznet Gio):**
- Flavor MS 4.2: ~Rp 150,000 - 200,000/bulan

---

**Total Estimated Cost:** Rp 300,000 - 400,000/bulan

---

## 🚀 Deployment Steps

### Phase 1: Provisioning (30 minutes)

#### 1.1 Order VMs

**Via Biznet Gio Portal:**
```
VM1 & VM2:
  ☐ OS: Ubuntu 22.04
  ☐ Flavor: MS 4.2 (atau lebih tinggi)
  ☐ Region: Pilih yang sama untuk latency rendah
  ☐ Network: Pastikan bisa komunikasi antar VM (same VPC/network)
  ☐ SSH Key: Upload public key Anda
```

#### 1.2 Verify Network Connectivity

```bash
# Dari VM1 ke VM2
ping <VM2_PRIVATE_IP>

# Dari VM2 ke VM1
ping <VM1_PRIVATE_IP>

# Test port (setelah Prometheus installed)
nc -zv <VM1_PRIVATE_IP> 9090
```

#### 1.3 Record IP Addresses

```bash
# VM1 (Monitoring Server)
PUBLIC_IP_VM1=<your-vm1-public-ip>
PRIVATE_IP_VM1=<your-vm1-private-ip>

# VM2 (K8s Server)
PUBLIC_IP_VM2=<your-vm2-public-ip>
PRIVATE_IP_VM2=<your-vm2-private-ip>

# Save these untuk digunakan di configuration
echo "VM1_PUBLIC=$PUBLIC_IP_VM1" > ~/deployment-ips.txt
echo "VM1_PRIVATE=$PRIVATE_IP_VM1" >> ~/deployment-ips.txt
echo "VM2_PUBLIC=$PUBLIC_IP_VM2" >> ~/deployment-ips.txt
echo "VM2_PRIVATE=$PRIVATE_IP_VM2" >> ~/deployment-ips.txt
```

---

### Phase 2: Setup VM1 - Monitoring Server (2-3 hours)

SSH ke VM1:
```bash
ssh -i ~/.ssh/your-key.pem ubuntu@<VM1_PUBLIC_IP>
```

#### 2.1 Basic System Configuration

```bash
# Switch to root
sudo su

# Set hostname
hostnamectl set-hostname monitoring-server

# Set timezone
timedatectl set-timezone Asia/Jakarta

# Update system
apt update -y && apt upgrade -y

# Install prerequisites
apt install -y curl wget tar vim htop ufw
```

#### 2.2 Configure Firewall

```bash
# Default policies
ufw default deny incoming
ufw default allow outgoing

# Allow SSH
ufw allow 22/tcp

# Allow Prometheus from VM2 only
ufw allow from <VM2_PRIVATE_IP> to any port 9090 proto tcp

# Allow Grafana (restrict to your office IP untuk production)
ufw allow 3000/tcp
# Atau restrict: ufw allow from <YOUR_OFFICE_IP> to any port 3000 proto tcp

# Enable firewall
ufw enable
ufw status verbose
```

#### 2.3 Install Prometheus

```bash
# Create user and directories
useradd -M -U prometheus
mkdir -p /var/lib/prometheus_data/
chown prometheus:prometheus -R /var/lib/prometheus_data/

# Download Prometheus
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.53.4/prometheus-2.53.4.linux-amd64.tar.gz

# Verify checksum (optional but recommended)
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.53.4/sha256sums.txt
sha256sum -c sha256sums.txt 2>&1 | grep prometheus-2.53.4.linux-amd64.tar.gz

# Extract and install
tar -zxvf prometheus-2.53.4.linux-amd64.tar.gz
mv prometheus-2.53.4.linux-amd64 /opt/prometheus
chown prometheus:prometheus -R /opt/prometheus/
```

#### 2.4 Configure Prometheus Authentication

```bash
# Generate strong password
PROM_PASSWORD=$(openssl rand -base64 32)
echo "Prometheus Password: $PROM_PASSWORD"
echo "SAVE THIS PASSWORD IN YOUR PASSWORD MANAGER!"
echo "$PROM_PASSWORD" > /root/prometheus-password.txt
chmod 600 /root/prometheus-password.txt

# Generate bcrypt hash (you need to do this via online tool or htpasswd)
# Visit: https://bcrypt.online/
# Input your password, Cost Factor: 12
# Copy the hash

# Create auth file
cat > /opt/prometheus/auth.yml <<EOF
basic_auth_users:
  admin: "\$2y\$12\$PASTE_YOUR_BCRYPT_HASH_HERE"
EOF

# Verify auth file
/opt/prometheus/promtool check web-config /opt/prometheus/auth.yml
```

#### 2.5 Configure Prometheus

```bash
# Backup original config
cp /opt/prometheus/prometheus.yml /opt/prometheus/prometheus.yml.bak

# Create new config
cat > /opt/prometheus/prometheus.yml <<EOF
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    basic_auth:
      username: 'admin'
      password: 'PASTE_YOUR_PASSWORD_HERE'  # Plain text for self-scraping
    static_configs:
      - targets: ['localhost:9090']
EOF

# Verify config
/opt/prometheus/promtool check config /opt/prometheus/prometheus.yml

# Set permissions
chown prometheus:prometheus /opt/prometheus/prometheus.yml
chown prometheus:prometheus /opt/prometheus/auth.yml
```

#### 2.6 Create Prometheus Systemd Service

```bash
cat > /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Restart=on-failure
RestartSec=5s

ExecStart=/opt/prometheus/prometheus \\
  --config.file=/opt/prometheus/prometheus.yml \\
  --storage.tsdb.path=/var/lib/prometheus_data/ \\
  --storage.tsdb.retention.time=30d \\
  --web.enable-remote-write-receiver \\
  --web.config.file=/opt/prometheus/auth.yml

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd
systemctl daemon-reload

# Start Prometheus
systemctl enable prometheus.service
systemctl start prometheus.service

# Check status
systemctl status prometheus.service

# Check logs
journalctl -u prometheus.service -f
# Press Ctrl+C to exit
```

#### 2.7 Verify Prometheus

```bash
# Test locally (should prompt for auth)
curl -u admin:YOUR_PASSWORD http://localhost:9090/api/v1/status/runtimeinfo

# Check remote write is enabled
curl -u admin:YOUR_PASSWORD http://localhost:9090/api/v1/status/flags | grep remote-write
```

#### 2.8 Install Grafana

```bash
# Install prerequisites
apt-get install -y apt-transport-https software-properties-common

# Add GPG key
mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Add repository
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | tee /etc/apt/sources.list.d/grafana.list

# Update and install
apt-get update
apt-get install grafana

# Start Grafana
systemctl enable grafana-server
systemctl start grafana-server

# Check status
systemctl status grafana-server
```

#### 2.9 Access Grafana

```bash
# Grafana should now be accessible at:
echo "Grafana URL: http://$PUBLIC_IP_VM1:3000"
echo "Default credentials: admin / admin"
echo "YOU WILL BE PROMPTED TO CHANGE PASSWORD ON FIRST LOGIN"
```

**Via Web Browser:**
1. Navigate to: `http://<VM1_PUBLIC_IP>:3000`
2. Login: `admin` / `admin`
3. Change password when prompted
4. Save new password in password manager!

#### 2.10 Configure Grafana Data Source

**Via Web UI:**
1. Go to: Configuration → Data Sources → Add data source
2. Select: **Prometheus**
3. Configure:
   ```
   Name: Prometheus
   URL: http://localhost:9090

   Auth:
   ☑ Basic auth

   Basic Auth Details:
   User: admin
   Password: <YOUR_PROMETHEUS_PASSWORD>
   ```
4. Click: **Save & Test**
5. Should show: "Data source is working"

---

### Phase 3: Setup VM2 - Kubernetes with K3s (2-3 hours)

SSH ke VM2:
```bash
ssh -i ~/.ssh/your-key.pem ubuntu@<VM2_PUBLIC_IP>
```

#### 3.1 Basic System Configuration

```bash
# Switch to root
sudo su

# Set hostname
hostnamectl set-hostname k8s-node

# Set timezone
timedatectl set-timezone Asia/Jakarta

# Update system
apt update -y && apt upgrade -y

# Install prerequisites
apt install -y curl wget
```

#### 3.2 Configure Firewall

```bash
apt install -y ufw

# Default policies
ufw default deny incoming
ufw default allow outgoing

# Allow SSH
ufw allow 22/tcp

# Allow Kubernetes API (optional, jika mau remote kubectl access)
ufw allow 6443/tcp

# Allow dari semua range internal untuk K8s networking
ufw allow from 10.42.0.0/16   # K3s pod network
ufw allow from 10.43.0.0/16   # K3s service network

# Enable firewall
ufw enable
ufw status verbose
```

#### 3.3 Install K3s

```bash
# Install K3s (lightweight Kubernetes)
curl -sfL https://get.k3s.io | sh -

# Wait for K3s to be ready
systemctl status k3s

# Verify installation
k3s kubectl get nodes

# Setup kubectl alias
echo "alias kubectl='k3s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Verify
kubectl get nodes
kubectl get pods -A
```

#### 3.4 Install Helm

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version

# Add Prometheus Community repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

#### 3.5 Prepare Kubernetes Secret for Remote Write

```bash
# Use the same password as Prometheus
PROM_USER="admin"
PROM_PASSWORD="<YOUR_PROMETHEUS_PASSWORD>"  # Same as VM1

# Create monitoring namespace
kubectl create namespace monitoring

# Create secret
kubectl create secret generic kubepromsecret \
    --from-literal=username=$PROM_USER \
    --from-literal=password=$PROM_PASSWORD \
    -n monitoring

# Verify
kubectl get secret kubepromsecret -n monitoring
```

#### 3.6 Create Helm Values File

```bash
# Get VM1 private IP
VM1_PRIVATE_IP="<VM1_PRIVATE_IP>"

# Create values.yaml
cat > /root/kube-prometheus-values.yaml <<EOF
prometheus:
  prometheusSpec:
    # Scrape interval
    scrapeInterval: "1m"
    evaluationInterval: "1m"

    # Data retention
    retention: 15d

    # Remote write to central Prometheus
    remoteWrite:
    - url: http://${VM1_PRIVATE_IP}:9090/api/v1/write
      basicAuth:
        username:
          name: kubepromsecret
          key: username
        password:
          name: kubepromsecret
          key: password
      queueConfig:
        maxSamplesPerSend: 1000
        maxShards: 200
        capacity: 2500

    # Resource limits
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 4Gi

    # Storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

# Disable Grafana (we use central Grafana on VM1)
grafana:
  enabled: false

# Disable AlertManager (optional, enable jika perlu)
alertmanager:
  enabled: false

# Node Exporter (metrics dari node)
nodeExporter:
  enabled: true

# Kube State Metrics (K8s object metrics)
kubeStateMetrics:
  enabled: true
EOF
```

#### 3.7 Install kube-prometheus-stack

```bash
# Install
helm install -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --version 58.2.2 \
    -f /root/kube-prometheus-values.yaml \
    --create-namespace

# Wait for pods to be ready (bisa 2-5 menit)
kubectl get pods -n monitoring -w
# Press Ctrl+C ketika semua pods STATUS = Running

# Verify installation
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

#### 3.8 Verify Remote Write

```bash
# Check Prometheus logs for remote write errors
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 | grep remote_write

# Should NOT show authentication errors
# Successful logs look like:
# level=info msg="Remote write sent" duration=...

# Check metrics on VM1
# SSH to VM1 and query:
curl -u admin:PASSWORD 'http://localhost:9090/api/v1/query?query=up{job="kubernetes-nodes"}'
```

---

### Phase 4: Import Dashboards (30 minutes)

#### 4.1 Download Dashboard Files

**On your local machine:**
```bash
# Clone repository
git clone https://github.com/jokosu10/biznetgio-webinar-monitoring.git
cd biznetgio-webinar-monitoring/dashboard/

# You should have:
ls -lh
# k8s-api-server.json
# k8s-clusters.json
# k8s-namespaces.json
# k8s-nodes.json
# k8s-pods.json
# node-exporter-full.json
```

#### 4.2 Import via Grafana UI

**For Each Dashboard:**

1. Open Grafana: `http://<VM1_PUBLIC_IP>:3000`
2. Login with your credentials
3. Go to: **Dashboards** → **Import**
4. Click: **Upload JSON file**
5. Select dashboard file (e.g., `k8s-clusters.json`)
6. Configure:
   ```
   Folder: General (or create new folder "Kubernetes Monitoring")
   Prometheus: Select "Prometheus" data source
   ```
7. Click: **Import**

**Repeat for all 6 dashboards:**
- ✅ k8s-api-server.json
- ✅ k8s-clusters.json
- ✅ k8s-namespaces.json
- ✅ k8s-nodes.json
- ✅ k8s-pods.json
- ✅ node-exporter-full.json

#### 4.3 Verify Dashboards

Check each dashboard menampilkan data:
- **K8s Clusters**: Overall cluster health
- **K8s Nodes**: Node-level metrics (CPU, Memory, Disk)
- **K8s Pods**: Pod-level metrics
- **Node Exporter Full**: Detailed server metrics

**Troubleshooting jika "No Data":**
```bash
# 1. Check Prometheus has data
curl -u admin:PASS 'http://<VM1_PRIVATE_IP>:9090/api/v1/query?query=up'

# 2. Check time range di Grafana (top right)
# Set to "Last 5 minutes" atau "Last 1 hour"

# 3. Check remote write dari K8s
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 | grep -i error
```

---

## ✅ Post-Deployment Verification

### Checklist

**VM1 (Monitoring Server):**
```bash
# SSH to VM1
ssh ubuntu@<VM1_PUBLIC_IP>

# ✓ Prometheus running
sudo systemctl status prometheus.service

# ✓ Grafana running
sudo systemctl status grafana-server

# ✓ Prometheus has metrics dari K8s
curl -u admin:PASS 'http://localhost:9090/api/v1/query?query=up{job=~"kubernetes.*"}'

# ✓ Disk space adequate
df -h
# Check /var/lib/prometheus_data/ has space

# ✓ Firewall configured
sudo ufw status
```

**VM2 (Kubernetes):**
```bash
# SSH to VM2
ssh ubuntu@<VM2_PUBLIC_IP>

# ✓ K3s running
sudo systemctl status k3s

# ✓ All monitoring pods running
kubectl get pods -n monitoring

# ✓ Prometheus scraping targets
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9091:9090 &
curl http://localhost:9091/api/v1/targets
# Should show UP for all targets

# ✓ Remote write working (no errors)
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 | grep -i "remote.*write"
```

**From Browser:**
```
# ✓ Grafana accessible
http://<VM1_PUBLIC_IP>:3000

# ✓ All dashboards showing data
- Navigate to each dashboard
- Verify metrics are displayed
- Check time range is appropriate
```

---

## 🔒 Security Hardening (Recommended)

### 1. Restrict Grafana Access

```bash
# SSH to VM1
sudo ufw delete allow 3000/tcp
sudo ufw allow from <YOUR_OFFICE_IP> to any port 3000 proto tcp
sudo ufw reload
```

### 2. Setup TLS for Grafana (Production)

```bash
# Install certbot
sudo apt install certbot

# Get certificate (requires domain pointing to VM1)
sudo certbot certonly --standalone -d grafana.yourdomain.com

# Configure Grafana for HTTPS
sudo vim /etc/grafana/grafana.ini

# Edit:
[server]
protocol = https
cert_file = /etc/letsencrypt/live/grafana.yourdomain.com/fullchain.pem
cert_key = /etc/letsencrypt/live/grafana.yourdomain.com/privkey.pem

# Restart Grafana
sudo systemctl restart grafana-server

# Update firewall
sudo ufw allow 443/tcp
```

### 3. Regular Security Updates

```bash
# Setup auto-updates (Ubuntu)
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 4. Monitoring Server Health

```bash
# Add self-monitoring dashboard
# Import dashboard ID: 1860 (Node Exporter Full)
# Point to Prometheus data source

# Monitor:
- CPU usage < 80%
- Memory usage < 85%
- Disk space > 20% free
- Network connectivity stable
```

---

## 🔧 Maintenance

### Daily Tasks
```bash
# Check service status
systemctl status prometheus grafana-server k3s

# Check disk space
df -h

# Check logs for errors
journalctl -u prometheus.service --since "1 hour ago" | grep -i error
```

### Weekly Tasks
```bash
# Verify backups (if configured)
ls -lh /backup/prometheus/
ls -lh /backup/grafana/

# Check metrics retention
curl -u admin:PASS http://localhost:9090/api/v1/status/tsdb | jq

# Review Grafana dashboards
# Ensure all showing data correctly
```

### Monthly Tasks
```bash
# Update system packages
apt update && apt upgrade -y

# Review resource usage trends
# Check if need to upgrade VM specs

# Test backup restore procedure
# Ensure backups are usable
```

---

## 📊 Resource Monitoring

### Expected Resource Usage

**VM1 (Monitoring Server):**
```
CPU: 20-40% average (2 vCPU)
RAM: 2-3 GB used (of 4 GB)
Disk: ~10-15 GB for 30 days retention
Network: 1-5 Mbps (incoming metrics)
```

**VM2 (K8s with K3s):**
```
CPU: 30-50% average (2 vCPU)
RAM: 2.5-3.5 GB used (of 4 GB)
Disk: ~20-30 GB (K3s + monitoring stack + apps)
Network: 1-5 Mbps (outgoing metrics)
```

### When to Upgrade

**Upgrade VM1 to 4 vCPU / 8 GB if:**
- Prometheus CPU > 70% consistently
- Memory usage > 85%
- Query latency increasing
- Adding more K8s clusters to monitor

**Upgrade VM2 to 4 vCPU / 8 GB if:**
- K8s pods experiencing OOMKilled
- CPU throttling observed
- Running multiple applications
- Need more worker capacity

---

## 🐛 Troubleshooting

### Issue: No Metrics in Grafana

**Symptoms:**
- Dashboards show "No Data"
- Panels display "N/A"

**Solutions:**
```bash
# 1. Check Prometheus has data from K8s
curl -u admin:PASS 'http://<VM1_IP>:9090/api/v1/query?query=up{job=~"kubernetes.*"}'

# 2. Check remote write on K8s
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 | grep remote_write

# 3. Verify network connectivity
# From VM2:
nc -zv <VM1_PRIVATE_IP> 9090

# 4. Check authentication
kubectl get secret kubepromsecret -n monitoring -o jsonpath='{.data.password}' | base64 -d
# Should match Prometheus password on VM1

# 5. Check time range in Grafana
# Set to "Last 5 minutes" or "Last 1 hour"
```

### Issue: Remote Write Failing

**Symptoms:**
- Errors in K8s Prometheus logs: "context deadline exceeded", "401 Unauthorized"

**Solutions:**
```bash
# Check network connectivity
kubectl run test-curl --rm -it --image=curlimages/curl -- curl -v http://<VM1_PRIVATE_IP>:9090/-/ready

# Check authentication
# Re-create secret with correct password
kubectl delete secret kubepromsecret -n monitoring
kubectl create secret generic kubepromsecret \
    --from-literal=username=admin \
    --from-literal=password=<CORRECT_PASSWORD> \
    -n monitoring

# Restart Prometheus pod
kubectl rollout restart statefulset kube-prometheus-stack-prometheus -n monitoring

# Check logs again
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 -f
```

### Issue: Prometheus High Memory

**Symptoms:**
- Prometheus service restarting
- OOM kills in logs

**Solutions:**
```bash
# 1. Reduce retention
vim /etc/systemd/system/prometheus.service
# Change: --storage.tsdb.retention.time=15d

systemctl daemon-reload
systemctl restart prometheus.service

# 2. Increase scrape interval (on K8s)
# Edit values.yaml, change scrapeInterval: "2m"
helm upgrade -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    -f /root/kube-prometheus-values.yaml

# 3. Clean old data (if needed)
systemctl stop prometheus.service
rm -rf /var/lib/prometheus_data/wal/*
systemctl start prometheus.service
```

### Issue: K3s Pods Not Starting

**Symptoms:**
- Pods stuck in Pending or CrashLoopBackOff

**Solutions:**
```bash
# Check pod status
kubectl describe pod <pod-name> -n monitoring

# Common issues:

# 1. Insufficient resources
kubectl top nodes
# If CPU/Memory > 90%, upgrade VM or reduce limits

# 2. PVC not binding
kubectl get pvc -n monitoring
# If Pending, check storage class:
kubectl get storageclass

# 3. Image pull issues
# Check internet connectivity:
curl -I https://registry.k8s.io

# 4. Check logs
kubectl logs <pod-name> -n monitoring
```

---

## 💰 Cost Optimization

### Tips to Reduce Costs

**1. Use Smaller VMs for Development:**
```
VM1: MS 2.2 (1 vCPU, 2 GB RAM) - Rp 80k/bulan
VM2: MS 2.2 (1 vCPU, 2 GB RAM) - Rp 80k/bulan
Total: Rp 160k/bulan (50% cheaper)

Limitations:
- Lower retention (7-15 days)
- Fewer concurrent users
- Slower query performance
```

**2. Reduce Retention Period:**
```bash
# Prometheus: 15d instead of 30d (save ~50% disk)
--storage.tsdb.retention.time=15d

# K8s Prometheus: 7d instead of 15d
retention: 7d
```

**3. Increase Scrape Interval:**
```yaml
# Change from 1m to 2m (save ~50% storage)
scrapeInterval: "2m"
```

**4. Disable Unused Components:**
```yaml
# If not using AlertManager
alertmanager:
  enabled: false

# If not scraping external nodes
nodeExporter:
  enabled: false  # Only if not needed
```

---

## 🎯 Next Steps

### After Successful Deployment

**1. Add Alerting (Recommended):**
- Enable AlertManager
- Configure notification channels (Slack, Email, PagerDuty)
- Add alert rules for critical metrics
- See: `CODE_REVIEW.md` Section 9 for alert examples

**2. Setup Backup Strategy:**
- Configure automated backups for Prometheus data
- Backup Grafana configuration
- See: `CODE_REVIEW.md` Section 10 for backup scripts

**3. Add More Monitoring:**
- Deploy sample applications to K8s
- Add custom metrics from your apps
- Create custom dashboards

**4. High Availability (For Production):**
- Add more K8s nodes (multi-node cluster)
- Configure Prometheus federation or Thanos
- Setup Grafana HA with PostgreSQL backend

**5. Security Enhancements:**
- Setup VPN access untuk Grafana
- Implement TLS/SSL everywhere
- Regular security audits
- Enable audit logging

---

## 📚 Reference Documentation

- **Main Installation Guide:** `README.md`
- **Security Review:** `CODE_REVIEW.md`
- **Complete Requirements:** `REQUIREMENTS.md`
- **This Guide:** `DEPLOYMENT_2VM.md`

**External Links:**
- K3s Documentation: https://docs.k3s.io/
- Prometheus: https://prometheus.io/docs/
- Grafana: https://grafana.com/docs/
- kube-prometheus-stack: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

---

## ✅ Quick Command Reference

### VM1 Commands
```bash
# Service management
systemctl status prometheus grafana-server
systemctl restart prometheus grafana-server
journalctl -u prometheus.service -f

# Check metrics
curl -u admin:PASS http://localhost:9090/api/v1/label/__name__/values

# Disk usage
du -sh /var/lib/prometheus_data/
```

### VM2 Commands
```bash
# K3s status
systemctl status k3s
kubectl get nodes
kubectl get pods -A

# Monitoring namespace
kubectl get pods -n monitoring
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 -f

# Helm management
helm list -n monitoring
helm upgrade -n monitoring kube-prometheus-stack prometheus-community/kube-prometheus-stack -f values.yaml
```

---

**Deployment Guide Version:** 1.0
**Last Updated:** 2025-11-05
**Tested On:** Ubuntu 22.04 LTS, K3s v1.28+, kube-prometheus-stack v58.2.2
