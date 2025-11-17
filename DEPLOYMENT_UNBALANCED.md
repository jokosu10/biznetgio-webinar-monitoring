# Deployment Guide: Unbalanced 2 VM Setup (Budget-Friendly)

**Setup Type:** Budget-Friendly / Development
**Total VMs:** 2 (1 new + 1 existing)
**Estimated Cost:** Rp 200,000 - 250,000/bulan
**Setup Time:** 4-6 hours

---

## 📋 Overview

Setup ini menggunakan **2 VM dengan spec yang berbeda** untuk menghemat biaya dengan memanfaatkan VPS yang sudah ada:

```
┌─────────────────────────────────────────────────────────┐
│      VM2: Kubernetes (K3s) - VPS LAMA                   │
│      Spec: 1 vCPU, 2GB RAM, 60GB disk                   │
│      IP: <VM2_PUBLIC> / <VM2_PRIVATE>                   │
│                                                         │
│  ⚠️ RESOURCE-LIMITED MODE                              │
│  ┌────────────────────────────────────────────────┐    │
│  │  K3s (Lightweight Kubernetes)                  │    │
│  │  ├── kube-prometheus-stack (minimal config)    │    │
│  │  ├── Reduced retention: 7 days                 │    │
│  │  ├── Lower resource limits                     │    │
│  │  └── Higher scrape interval (2m)               │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                        │
                        │ Remote Write (HTTP)
                        │ Port 9090
                        ▼
┌─────────────────────────────────────────────────────────┐
│      VM1: Monitoring Server - VPS BARU                  │
│      Spec: 4 vCPU, 4GB RAM, 60GB disk                   │
│      IP: <VM1_PUBLIC> / <VM1_PRIVATE>                   │
│                                                         │
│  ✅ COMFORTABLE RESOURCES                              │
│  ┌────────────────────────────────────────────────┐    │
│  │  Prometheus Server                             │    │
│  │  ├── Central metrics storage                   │    │
│  │  ├── 30 days retention                         │    │
│  │  ├── Port 9090                                 │    │
│  │  └── Handles queries efficiently               │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  Grafana Server                                │    │
│  │  ├── Dashboard visualization                   │    │
│  │  ├── Port 3000                                 │    │
│  │  └── Fast query performance                    │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 🖥️ VM Specifications

### VM1: Monitoring Server (VPS BARU - Order)

```yaml
Hostname: monitoring-server
OS: Ubuntu 22.04 LTS
CPU: 4 vCPU ✅
RAM: 4 GB ✅
Disk: 60 GB minimum (100 GB recommended)
Network:
  - Public IP (untuk akses Grafana)
  - Private IP (untuk komunikasi dengan VM2)
Ports:
  - 22/tcp: SSH
  - 9090/tcp: Prometheus (dari VM2)
  - 3000/tcp: Grafana (public)
```

**Order via Biznet Gio:**
- Flavor: **MS 4.4** (4 vCPU, 4GB RAM) atau equivalent
- Cost: ~Rp 200,000 - 250,000/bulan

---

### VM2: Kubernetes Server (VPS LAMA - Sudah Ada)

```yaml
Hostname: k8s-node
OS: Ubuntu 22.04 LTS
CPU: 1 vCPU ⚠️ (TIGHT!)
RAM: 2 GB ⚠️ (MINIMAL!)
Disk: 60 GB
Network:
  - Public/Private IP
Ports:
  - 22/tcp: SSH
  - 6443/tcp: Kubernetes API (optional)
```

**Status:** Sudah ada (no additional cost!)

**⚠️ IMPORTANT:**
- Resource sangat terbatas
- Hanya untuk monitoring stack (jangan deploy aplikasi lain)
- Akan menggunakan konfigurasi minimal/optimized

---

## 💰 Cost Comparison

| Item | Cost |
|------|------|
| VM1 (4 vCPU, 4GB) - NEW | Rp 200-250k/bulan |
| VM2 (1 vCPU, 2GB) - EXISTING | Rp 0 (sudah ada) |
| **TOTAL** | **Rp 200-250k/bulan** |

**Savings:** 50% dibanding 2 VM baru (2C/4GB each)

---

## 🚀 Deployment Steps

### Phase 1: Provisioning (30 minutes)

#### 1.1 Order VM1 (Monitoring Server)

**Via Biznet Gio Portal:**
```
☐ Order VPS baru
☐ OS: Ubuntu 22.04
☐ Flavor: MS 4.4 (4 vCPU, 4GB RAM) atau similar
☐ Region: SAMA dengan VM2
☐ Network: Same VPC/network dengan VM2
☐ SSH Key: Upload public key
```

#### 1.2 Prepare VM2 (Existing VPS)

**Backup data (jika ada):**
```bash
# SSH to VM2
ssh user@<VM2_IP>

# Backup any important data
tar -czf ~/backup-$(date +%Y%m%d).tar.gz ~/important-data/

# Optional: Fresh install Ubuntu 22.04 (recommended)
# Via Biznet Gio portal: Rebuild/Reinstall OS
```

#### 1.3 Record IP Addresses

```bash
# Create IP reference file
cat > ~/deployment-ips.txt <<EOF
VM1_PUBLIC=<your-vm1-public-ip>
VM1_PRIVATE=<your-vm1-private-ip>
VM2_PUBLIC=<your-vm2-public-ip>
VM2_PRIVATE=<your-vm2-private-ip>
EOF

# Verify network connectivity
# From VM1:
ping <VM2_PRIVATE_IP>

# From VM2:
ping <VM1_PRIVATE_IP>
```

---

### Phase 2: Setup VM1 - Monitoring Server (2-3 hours)

**VM1 memiliki resource yang comfortable, jadi follow standard installation.**

📘 **Follow:** [DEPLOYMENT_2VM.md](DEPLOYMENT_2VM.md) **Phase 2** (halaman VM1 setup)

**Summary commands:**
```bash
# SSH to VM1
ssh ubuntu@<VM1_PUBLIC_IP>

# Basic setup
sudo su
hostnamectl set-hostname monitoring-server
timedatectl set-timezone Asia/Jakarta
apt update -y && apt upgrade -y

# Install Prometheus (follow DEPLOYMENT_2VM.md section 2.3-2.7)
# Install Grafana (follow DEPLOYMENT_2VM.md section 2.8-2.10)
```

**⚠️ No changes needed** - VM1 has sufficient resources untuk standard config.

---

### Phase 3: Setup VM2 - Kubernetes (MODIFIED for Low Resources)

**⚠️ IMPORTANT: VM2 memiliki resource terbatas, jadi kita gunakan konfigurasi OPTIMIZED.**

SSH ke VM2:
```bash
ssh ubuntu@<VM2_PUBLIC_IP>
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

# Allow Kubernetes API (optional)
ufw allow 6443/tcp

# Allow internal K8s networks
ufw allow from 10.42.0.0/16   # K3s pod network
ufw allow from 10.43.0.0/16   # K3s service network

# Enable firewall
ufw enable
ufw status verbose
```

#### 3.3 System Optimizations for Low Memory

```bash
# Disable swap for Kubernetes (recommended)
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Configure kernel parameters for low memory
cat >> /etc/sysctl.conf <<EOF

# Kubernetes optimizations for low memory
vm.swappiness=0
vm.overcommit_memory=1
kernel.panic=10
kernel.panic_on_oops=1
EOF

sysctl -p
```

#### 3.4 Install K3s (Lightweight Mode)

```bash
# Install K3s with minimal components
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
    --disable traefik \
    --disable servicelb \
    --kubelet-arg=max-pods=50 \
    --kube-apiserver-arg=default-not-ready-toleration-seconds=10 \
    --kube-apiserver-arg=default-unreachable-toleration-seconds=10" sh -

# Wait for K3s to be ready
sleep 30
systemctl status k3s

# Setup kubectl alias
echo "alias kubectl='k3s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Verify
kubectl get nodes
```

#### 3.5 Install Helm

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version

# Add Prometheus Community repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

#### 3.6 Prepare Kubernetes Secret

```bash
# Use the same password as Prometheus on VM1
PROM_USER="admin"
PROM_PASSWORD="<YOUR_PROMETHEUS_PASSWORD>"

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

#### 3.7 Create OPTIMIZED Helm Values File

**⚠️ CRITICAL: Gunakan konfigurasi ini untuk VM dengan 1 vCPU, 2GB RAM**

```bash
# Get VM1 private IP
VM1_PRIVATE_IP="<VM1_PRIVATE_IP>"

# Create optimized values.yaml for low-resource VM
cat > /root/kube-prometheus-values-minimal.yaml <<EOF
# OPTIMIZED CONFIGURATION FOR 1 vCPU, 2GB RAM
# Do NOT use this on production!

prometheus:
  prometheusSpec:
    # Reduce scrape interval to save CPU/memory
    scrapeInterval: "2m"
    scrapeTimeout: "30s"
    evaluationInterval: "2m"

    # Shorter retention to save memory/disk
    retention: 7d
    retentionSize: "5GB"

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
        capacity: 500
        maxShards: 10
        minShards: 1
        maxSamplesPerSend: 500
        batchSendDeadline: 10s

    # CRITICAL: Low resource limits for 1 vCPU, 2GB RAM
    resources:
      requests:
        cpu: 300m
        memory: 800Mi
      limits:
        cpu: 800m        # Max 80% of 1 vCPU
        memory: 1400Mi   # Max 70% of 2GB (leave room for K8s)

    # Use emptyDir instead of PVC to save resources
    storageSpec: {}

    # Reduce walCompression
    walCompression: true

# Disable Grafana (we use central Grafana on VM1)
grafana:
  enabled: false

# Disable AlertManager to save resources
alertmanager:
  enabled: false

# Node Exporter (keep enabled)
nodeExporter:
  enabled: true
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 100m
      memory: 128Mi

# Kube State Metrics (keep enabled but limit resources)
kubeStateMetrics:
  enabled: true
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 100m
      memory: 128Mi

# Prometheus Operator (reduce resources)
prometheusOperator:
  resources:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi

# Disable default scrape configs we don't need
defaultRules:
  create: true
  rules:
    alertmanager: false
    etcd: false
    configReloaders: false
    general: false
    k8s: true
    kubeApiserverAvailability: false
    kubeApiserverBurnrate: false
    kubeApiserverHistogram: false
    kubeApiserverSlos: false
    kubelet: true
    kubeProxy: false
    kubePrometheusGeneral: false
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    kubeScheduler: false
    kubeStateMetrics: true
    network: false
    node: true
    nodeExporterAlerting: false
    nodeExporterRecording: true
    prometheus: false
    prometheusOperator: false
EOF
```

#### 3.8 Install kube-prometheus-stack (Minimal Config)

```bash
# Install with optimized values
helm install -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --version 58.2.2 \
    -f /root/kube-prometheus-values-minimal.yaml \
    --create-namespace

# This will take 3-5 minutes
# Monitor the installation
kubectl get pods -n monitoring -w

# Press Ctrl+C when all pods are Running
```

#### 3.9 Verify Installation

```bash
# Check all pods are running
kubectl get pods -n monitoring

# Expected pods:
# - kube-prometheus-stack-prometheus-0
# - kube-prometheus-stack-operator-xxx
# - kube-prometheus-stack-kube-state-metrics-xxx
# - kube-prometheus-stack-prometheus-node-exporter-xxx

# Check resource usage
kubectl top nodes
kubectl top pods -n monitoring

# Check Prometheus logs for remote write
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 | grep remote_write

# Should NOT show errors like "401 Unauthorized" or "connection refused"
```

#### 3.10 Monitor Resource Usage

```bash
# Install htop for monitoring
apt install htop

# Run htop to monitor resources
htop

# Expected usage on 1 vCPU, 2GB RAM:
# - CPU: 60-90% (high but acceptable)
# - Memory: 1.4-1.8 GB (70-90% usage)
```

**⚠️ Warning Signs:**
- If memory > 95%: Pods akan di-evict (OOMKilled)
- If CPU consistently 100%: Slow performance
- If swap usage > 0: System is struggling

---

### Phase 4: Import Dashboards (30 minutes)

Same as standard deployment - follow [DEPLOYMENT_2VM.md Phase 4](DEPLOYMENT_2VM.md#phase-4-import-dashboards-30-minutes)

Import via Grafana UI at: `http://<VM1_PUBLIC_IP>:3000`

**All 6 dashboards:**
- k8s-api-server.json
- k8s-clusters.json
- k8s-namespaces.json
- k8s-nodes.json
- k8s-pods.json
- node-exporter-full.json

---

## ✅ Post-Deployment Verification

### VM1 (Monitoring Server) - 4 vCPU, 4GB RAM

```bash
# SSH to VM1
ssh ubuntu@<VM1_PUBLIC_IP>

# Check services
systemctl status prometheus grafana-server

# Check resource usage (should be comfortable)
htop
# Expected: CPU 20-40%, Memory 2-3 GB

# Verify receiving metrics from K8s
curl -u admin:PASSWORD 'http://localhost:9090/api/v1/query?query=up{job=~"kubernetes.*"}'
```

### VM2 (Kubernetes) - 1 vCPU, 2GB RAM

```bash
# SSH to VM2
ssh ubuntu@<VM2_PUBLIC_IP>

# Check K3s status
systemctl status k3s

# Check pods
kubectl get pods -n monitoring

# Check resource usage (will be HIGH)
kubectl top nodes
kubectl top pods -n monitoring

# Expected on 1 vCPU, 2GB RAM:
# - Node CPU: 60-90%
# - Node Memory: 70-90%
# - This is NORMAL for this low-spec setup

# Check for OOMKilled pods
kubectl get pods -n monitoring | grep OOMKilled
# Should be empty

# Check logs for errors
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 --tail=50
```

---

## ⚠️ Known Limitations & Issues

### 1. High Resource Usage on VM2

**Expected:**
- CPU usage: 60-90% (NORMAL for 1 vCPU)
- Memory usage: 70-90% (NORMAL for 2GB)

**Mitigation:**
- Configured with low resource limits
- Disabled unnecessary components
- Using remote write (data stored on VM1)
- Short retention (7 days only on VM2)

---

### 2. Slow Queries on VM2

**Symptoms:**
- PromQL queries timeout
- Dashboard loading slow when querying VM2 directly

**Solution:**
- ✅ Query dari **VM1 (central Prometheus)** instead
- VM1 has better resources for queries
- Grafana already pointed to VM1

---

### 3. Possible OOMKilled Pods

**If pods get OOMKilled:**

```bash
# Check events
kubectl get events -n monitoring --sort-by='.lastTimestamp'

# If Prometheus pod OOMKilled:
# Reduce memory limits further
helm upgrade -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --set prometheus.prometheusSpec.resources.limits.memory=1200Mi \
    --reuse-values

# Or disable remote storage completely (keep local only)
# Edit values.yaml, set retention: 3d
```

---

### 4. Cannot Deploy Additional Apps on VM2

**Important:**
- VM2 resource sudah hampir penuh dengan monitoring stack
- **JANGAN** deploy aplikasi lain di K8s ini
- Jika perlu aplikasi, order VM3 untuk aplikasi cluster

---

### 5. Disk Space Can Fill Quickly

**Monitor disk usage:**
```bash
# On VM2
df -h

# If /var/lib/rancher filling up:
# Clean up old images
k3s crictl rmi --prune

# Reduce Prometheus retention
kubectl edit prometheus -n monitoring kube-prometheus-stack-prometheus
# Change retention: 3d
```

---

## 🔧 Troubleshooting

### Issue: VM2 High Memory Usage (>95%)

**Solution 1: Reduce Prometheus memory limits**
```bash
helm upgrade -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --set prometheus.prometheusSpec.resources.limits.memory=1000Mi \
    --set prometheus.prometheusSpec.resources.requests.memory=600Mi \
    --reuse-values
```

**Solution 2: Disable local storage completely**
```bash
# Force remote write only (no local storage)
helm upgrade -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --set prometheus.prometheusSpec.retention=1d \
    --reuse-values
```

**Solution 3: Enable swap (last resort)**
```bash
# Create 1GB swap file
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Make permanent
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Note: Swap will slow down performance but prevent OOM
```

---

### Issue: Remote Write Failing

**Check connectivity:**
```bash
# From VM2, test connection to VM1
kubectl run test-curl --rm -it --image=curlimages/curl -- \
    curl -v http://<VM1_PRIVATE_IP>:9090/-/ready

# Should return HTTP 200
```

**Check authentication:**
```bash
# Verify secret
kubectl get secret kubepromsecret -n monitoring -o jsonpath='{.data.password}' | base64 -d

# Should match Prometheus password on VM1
```

**Check Prometheus logs:**
```bash
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 | grep -i "remote.*write"
```

---

### Issue: K3s Service Failing

**Check K3s logs:**
```bash
journalctl -u k3s -n 100 --no-pager
```

**Common causes:**
- Out of memory: Enable swap or reduce resource usage
- Disk full: Clean up old images/data
- Network issues: Check firewall rules

**Restart K3s:**
```bash
systemctl restart k3s
kubectl get nodes  # Should become Ready in 1-2 minutes
```

---

## 💡 Optimization Tips

### 1. Reduce Scrape Interval Further

If CPU/memory still high:
```bash
# Edit values, change scrapeInterval to 5m
helm upgrade -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --set prometheus.prometheusSpec.scrapeInterval=5m \
    --reuse-values
```

### 2. Disable Unnecessary Metrics

```bash
# Disable certain scrape jobs
kubectl edit prometheus -n monitoring kube-prometheus-stack-prometheus

# In the spec, add:
spec:
  scrapeConfigs:
    - job_name: 'kubernetes-service-endpoints'
      enabled: false  # Disable if not needed
```

### 3. Use Node Selector for Prometheus

If you later add more K8s nodes, pin Prometheus to the beefier node:
```yaml
prometheus:
  prometheusSpec:
    nodeSelector:
      node-role: monitoring
```

---

## 📊 Expected Resource Usage

### VM1 (4 vCPU, 4GB RAM) - Comfortable

```
CPU Usage:    20-40% (plenty of headroom)
Memory:       2-3 GB (comfortable)
Disk:         ~50-80 GB for 30 days retention
Network IN:   1-3 Mbps (receiving metrics)
Network OUT:  <1 Mbps

Status: ✅ HEALTHY
```

### VM2 (1 vCPU, 2GB RAM) - Tight but Workable

```
CPU Usage:    60-90% (high but acceptable)
Memory:       1.4-1.8 GB (70-90% usage)
Disk:         ~10-20 GB for 7 days retention
Network OUT:  1-3 Mbps (sending metrics)

Status: ⚠️ TIGHT but functional
```

**Total System:**
- 5 vCPU total (good)
- 6 GB RAM total (adequate for monitoring stack)
- Cost: Rp 200-250k/bulan

---

## 🎯 When to Upgrade VM2

**Consider upgrading VM2 to 2 vCPU, 4GB RAM if:**

- ❌ Frequent OOMKilled errors
- ❌ CPU consistently 100%
- ❌ Remote write dropping samples
- ❌ Pods failing to start
- ✅ You want to deploy applications on K8s
- ✅ Need better monitoring resolution

**Upgrade cost:** +Rp 100-150k/bulan (total ~Rp 300-400k)

---

## 📝 Maintenance

### Daily Monitoring

```bash
# On VM2 - Check resource usage
ssh ubuntu@<VM2_PUBLIC_IP>
htop
kubectl top nodes
kubectl top pods -n monitoring

# Watch for:
- Memory > 95% → Risk of OOM
- CPU > 100% sustained → Performance issues
- Disk usage > 80% → Clean up needed
```

### Weekly Tasks

```bash
# Clean up unused images
k3s crictl rmi --prune

# Check for OOMKilled events
kubectl get events -n monitoring | grep OOMKilled

# Verify remote write is working
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 --tail=100 | grep remote_write
```

---

## 🚀 Next Steps After Deployment

### 1. Monitor VM2 Resource Usage

```bash
# Setup simple monitoring script
cat > /root/check-resources.sh <<'EOF'
#!/bin/bash
echo "=== Resource Check $(date) ==="
echo "Memory:"
free -h | grep Mem
echo ""
echo "CPU:"
top -bn1 | grep "Cpu(s)"
echo ""
echo "Disk:"
df -h | grep -E '/$|/var'
echo ""
echo "K8s Pods:"
k3s kubectl top pods -n monitoring 2>/dev/null || echo "Metrics not ready"
EOF

chmod +x /root/check-resources.sh

# Run every hour via cron
(crontab -l 2>/dev/null; echo "0 * * * * /root/check-resources.sh >> /var/log/resource-check.log") | crontab -
```

### 2. Setup Alerts (on VM1)

Monitor VM2 health from VM1 Prometheus:
```yaml
# Alert if VM2 node memory high
- alert: NodeMemoryHigh
  expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.90
  for: 5m
  annotations:
    summary: "Node {{ $labels.instance }} memory usage > 90%"

# Alert if Prometheus pod restarting
- alert: PrometheusPodRestarting
  expr: rate(kube_pod_container_status_restarts_total{namespace="monitoring",pod=~".*prometheus.*"}[15m]) > 0
  annotations:
    summary: "Prometheus pod restarting frequently"
```

### 3. Plan for Scaling

If you need more capacity later:
- **Option A:** Upgrade VM2 to 2 vCPU, 4GB RAM (+Rp 100-150k)
- **Option B:** Add VM3 for application workloads (keep VM2 monitoring-only)
- **Option C:** Migrate to managed Kubernetes (NEO K8s, GKE, etc.)

---

## 📚 Related Documentation

- **Main Installation Guide:** [README.md](README.md)
- **Standard 2 VM Guide:** [DEPLOYMENT_2VM.md](DEPLOYMENT_2VM.md) (for equal-spec VMs)
- **Requirements:** [REQUIREMENTS.md](REQUIREMENTS.md)
- **Security Review:** [CODE_REVIEW.md](CODE_REVIEW.md)

---

## ✅ Success Checklist

```
Setup Complete:
☐ VM1 (4C/4GB) running Prometheus + Grafana smoothly
☐ VM2 (1C/2GB) running K3s + monitoring stack
☐ Remote write dari VM2 → VM1 working
☐ All 6 dashboards imported and showing data
☐ VM2 memory usage < 90%
☐ VM2 CPU usage < 95%
☐ No OOMKilled pods
☐ Resource monitoring script setup
☐ Alerts configured (optional)

Cost Savings:
☑ Saved 50% vs. 2x (2C/4GB) VMs
☑ Utilized existing VPS instead of letting it idle
☑ Total cost: ~Rp 200-250k/bulan

Limitations Understood:
☑ VM2 is resource-constrained (expected)
☑ Cannot deploy apps on VM2 K8s cluster
☑ May need to upgrade VM2 later if needed
☑ Monitoring resolution reduced (2m vs 1m scrape interval)
```

---

**Document Version:** 1.0
**Last Updated:** 2025-11-05
**For:** Budget-friendly deployment using 1 existing + 1 new VPS
**Total Cost:** ~Rp 200,000 - 250,000/bulan (50% savings!)

---

## 💬 Final Notes

Setup ini adalah **trade-off antara cost dan performance**:

**✅ Pros:**
- Hemat biaya (50% cheaper)
- Memanfaatkan VPS lama yang sudah ada
- VM1 comfortable untuk Prometheus + Grafana
- Tetap bisa monitoring K8s dengan baik

**⚠️ Cons:**
- VM2 resource tight (high CPU/memory usage normal)
- Tidak bisa deploy aplikasi lain di K8s
- Monitoring resolution reduced (2m scrape interval)
- Mungkin perlu upgrade VM2 di masa depan

**Cocok untuk:**
- Development/testing
- Learning Kubernetes monitoring
- Budget-conscious deployment
- Webinar/demo purposes

**TIDAK cocok untuk:**
- Production dengan high traffic
- Monitoring banyak aplikasi
- Require low-latency queries
- Kubernetes cluster untuk aplikasi

Jika nanti butuh more capacity, tinggal upgrade VM2 atau add VM3! 🚀
