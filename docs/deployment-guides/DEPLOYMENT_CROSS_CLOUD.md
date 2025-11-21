# Deployment Guide: Cross-Cloud Setup (VM2 Without Public IP)

**Setup Type:** Cross-Cloud / VM2 No Public IP
**Total VMs:** 2 (different cloud providers)
**Estimated Cost:** Varies by provider
**Setup Time:** 4-6 hours

---

## 📋 Scenario Overview

Setup ini untuk kasus di mana:
- ✅ VM1 dan VM2 di cloud provider yang **BERBEDA**
- ✅ VM1 memiliki **Public IPv4**
- ❌ VM2 **TIDAK** memiliki Public IPv4
- ✅ VM2 bisa melakukan **outbound connections**

```
┌─────────────────────────────────────────────────────────┐
│         VM2: Kubernetes (Cloud Provider B)              │
│         NO PUBLIC IPv4 ❌                               │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  K3s + kube-prometheus-stack                   │    │
│  │  - Scrapes local metrics                       │    │
│  │  - Sends to VM1 via OUTBOUND connection        │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                        │
                        │ Remote Write (OUTBOUND)
                        │ http://103.x.x.x:9090/api/v1/write
                        │ (via Internet - Basic Auth)
                        ▼
┌─────────────────────────────────────────────────────────┐
│       VM1: Monitoring Server (Cloud Provider A)         │
│       Public IPv4: 103.x.x.x ✅                         │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  Prometheus Server                             │    │
│  │  - Port 9090 (accepts remote write)            │    │
│  │  - Basic authentication enabled                │    │
│  │  - Firewall: Allow port 9090                   │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  Grafana Server                                │    │
│  │  - Port 3000 (web UI)                          │    │
│  │  - Accessible via public IP                    │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                        ▲
                        │ HTTPS (Browser)
                        │ http://103.x.x.x:3000
                        │
                 ┌──────────────┐
                 │ Admin/Users  │
                 └──────────────┘
```

---

## 🔑 Key Concepts

### Why This Works Without VM2 Public IP

**VM2 hanya perlu OUTBOUND connection:**
```
Analogy: Handphone browsing internet
- Handphone tidak punya IP publik
- Tapi bisa akses website (outbound connection)
- Website respond balik ke handphone

Same concept:
- VM2 tidak punya IP publik
- Tapi bisa kirim data ke VM1 (outbound connection)
- VM1 respond balik (ack packets)
```

**VM2 TIDAK perlu:**
- ❌ Public IP
- ❌ Inbound firewall rules
- ❌ Port forwarding
- ❌ VPN/tunnel setup

**VM2 HANYA perlu:**
- ✅ Internet access (outbound)
- ✅ Bisa resolve DNS
- ✅ Bisa reach VM1 public IP port 9090

---

## 🖥️ VM Specifications

### VM1: Monitoring Server (Cloud Provider A)

```yaml
Cloud: Provider A (Biznet Gio, AWS, etc.)
Public IPv4: ✅ REQUIRED
Private IP: Optional
OS: Ubuntu 22.04 LTS
CPU: 4 vCPU (or 2 vCPU minimum)
RAM: 4 GB
Disk: 60 GB minimum

Ports (Public Access):
  - 22/tcp: SSH
  - 9090/tcp: Prometheus ⚠️ EXPOSED TO INTERNET
  - 3000/tcp: Grafana
```

**⚠️ Security Note:**
Port 9090 akan exposed ke internet. Gunakan:
- ✅ Basic authentication (required)
- ✅ Strong password
- ✅ Firewall rules (optional: restrict to VM2's cloud IP range)
- 🔒 TLS (recommended untuk production)

---

### VM2: Kubernetes Server (Cloud Provider B)

```yaml
Cloud: Provider B (Different from VM1)
Public IPv4: ❌ TIDAK PERLU
Private IP: Yes (for internal cluster networking)
OS: Ubuntu 22.04 LTS
CPU: 1-4 vCPU (depending on workload)
RAM: 2-4 GB
Disk: 60 GB minimum

Ports (Outbound Only):
  - Internet access REQUIRED
  - No inbound ports needed
```

**✅ This VM can be:**
- Cloud instance without public IP
- Private subnet instance with NAT gateway
- Container instance with outbound-only networking
- Any compute with internet egress

---

## 🚀 Deployment Steps

### Phase 1: Setup VM1 (Monitoring Server)

#### 1.1 Provision VM1

**Requirements:**
```
☐ Ubuntu 22.04
☐ Public IPv4 address
☐ SSH access configured
☐ Note down Public IP: _______________
```

#### 1.2 Basic Configuration

```bash
# SSH to VM1
ssh ubuntu@<VM1_PUBLIC_IP>

# Become root
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

#### 1.3 Configure Firewall - INTERNET EXPOSED

**⚠️ CRITICAL: VM1 port 9090 akan exposed ke internet!**

```bash
# Default policies
ufw default deny incoming
ufw default allow outgoing

# Allow SSH
ufw allow 22/tcp

# Allow Prometheus - DARI INTERNET!
ufw allow 9090/tcp
# ⚠️ This allows from anywhere! See security options below

# Allow Grafana - untuk admin access
ufw allow 3000/tcp
# Or restrict: ufw allow from <YOUR_OFFICE_IP> to any port 3000

# Enable firewall
ufw enable
ufw status verbose
```

**🔒 Security Options (Choose One):**

**Option A: Allow from anywhere** (simplest, requires strong auth)
```bash
ufw allow 9090/tcp
# ⚠️ Basic auth MUST be strong!
```

**Option B: Restrict to VM2's cloud IP range** (better)
```bash
# Find VM2's egress IP range
# Example: AWS region us-east-1 IP ranges
# Or: Check VM2's actual egress IP (curl ifconfig.me dari VM2)

# Allow only from specific IP or CIDR
ufw allow from <VM2_EGRESS_IP_OR_CIDR> to any port 9090 proto tcp

# Example:
# ufw allow from 54.80.0.0/16 to any port 9090 proto tcp
```

**Option C: Use Cloudflare Tunnel** (most secure - see separate guide)

---

#### 1.4 Install Prometheus

```bash
# Create user
useradd -M -U prometheus
mkdir -p /var/lib/prometheus_data/
chown prometheus:prometheus -R /var/lib/prometheus_data/

# Download Prometheus
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.53.4/prometheus-2.53.4.linux-amd64.tar.gz

# Extract
tar -zxvf prometheus-2.53.4.linux-amd64.tar.gz
mv prometheus-2.53.4.linux-amd64 /opt/prometheus
chown prometheus:prometheus -R /opt/prometheus/
```

#### 1.5 Configure Prometheus Authentication

**⚠️ EXTRA IMPORTANT: Strong password required karena exposed ke internet!**

```bash
# Generate VERY STRONG password (32+ chars)
PROM_PASSWORD=$(openssl rand -base64 48)
echo "Prometheus Password: $PROM_PASSWORD"
echo "⚠️ SAVE THIS IN PASSWORD MANAGER!"
echo "$PROM_PASSWORD" > /root/prometheus-password.txt
chmod 600 /root/prometheus-password.txt

# Generate bcrypt hash (use online tool)
# Visit: https://bcrypt.online/
# Input: Your password
# Cost Factor: 14 (higher for better security)
# Copy hash

# Create auth file
cat > /opt/prometheus/auth.yml <<'EOF'
basic_auth_users:
  admin: "$2y$14$PASTE_YOUR_BCRYPT_HASH_HERE"
EOF

# Verify
/opt/prometheus/promtool check web-config /opt/prometheus/auth.yml
```

#### 1.6 Configure Prometheus

```bash
cat > /opt/prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    basic_auth:
      username: 'admin'
      password: 'YOUR_PLAIN_PASSWORD_HERE'
    static_configs:
      - targets: ['localhost:9090']
EOF

# Verify
/opt/prometheus/promtool check config /opt/prometheus/prometheus.yml

# Set permissions
chown prometheus:prometheus /opt/prometheus/prometheus.yml
chown prometheus:prometheus /opt/prometheus/auth.yml
```

#### 1.7 Create Systemd Service

```bash
cat > /etc/systemd/system/prometheus.service <<'EOF'
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Restart=on-failure
RestartSec=5s

ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus_data/ \
  --storage.tsdb.retention.time=30d \
  --web.enable-remote-write-receiver \
  --web.config.file=/opt/prometheus/auth.yml \
  --web.listen-address=0.0.0.0:9090

[Install]
WantedBy=multi-user.target
EOF

# ⚠️ Note: --web.listen-address=0.0.0.0:9090
# This listens on ALL interfaces (including public IP)

# Reload and start
systemctl daemon-reload
systemctl enable prometheus.service
systemctl start prometheus.service
systemctl status prometheus.service
```

#### 1.8 Verify Prometheus Accessibility

**From VM1 (local):**
```bash
curl -u admin:YOUR_PASSWORD http://localhost:9090/api/v1/status/runtimeinfo
# Should return JSON
```

**From your local machine (test public access):**
```bash
# Replace with your VM1 public IP and password
curl -u admin:YOUR_PASSWORD http://103.x.x.x:9090/api/v1/status/runtimeinfo

# Should return JSON
# If fails: Check firewall, check service status
```

**⚠️ Security Check:**
```bash
# Try without authentication (should FAIL)
curl http://103.x.x.x:9090/api/v1/status/runtimeinfo
# Should return: 401 Unauthorized ✅
```

#### 1.9 Install Grafana

```bash
# Prerequisites
apt-get install -y apt-transport-https software-properties-common

# Add GPG key
mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Add repository
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | tee /etc/apt/sources.list.d/grafana.list

# Install
apt-get update
apt-get install grafana

# Start
systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server
```

#### 1.10 Access Grafana & Configure

**Access via browser:**
```
http://<VM1_PUBLIC_IP>:3000

Default credentials:
Username: admin
Password: admin

⚠️ Change password immediately!
```

**Add Prometheus data source:**
1. Go to: Configuration → Data Sources
2. Add data source → Prometheus
3. Configure:
   ```
   Name: Prometheus
   URL: http://localhost:9090

   Auth: ☑ Basic auth

   Basic Auth Details:
   User: admin
   Password: <YOUR_PROMETHEUS_PASSWORD>
   ```
4. Save & Test → Should show "Data source is working"

---

### Phase 2: Setup VM2 (Kubernetes Without Public IP)

#### 2.1 Provision VM2

**Requirements:**
```
☐ Ubuntu 22.04
☐ NO public IP needed ✅
☐ Internet access (outbound) REQUIRED
☐ SSH access configured (via private IP, bastion, or console)
```

**How to access VM2 without public IP:**
- Via bastion host / jump server
- Via cloud console (web SSH)
- Via VPN to cloud private network
- Via private network from another instance

```bash
# Example: SSH via bastion
ssh -J bastion-host ubuntu@<VM2_PRIVATE_IP>

# Or via cloud console (varies by provider)
```

#### 2.2 Basic Configuration

```bash
# Become root
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

#### 2.3 Verify Internet Connectivity

**⚠️ CRITICAL: VM2 must have outbound internet access**

```bash
# Test DNS resolution
nslookup google.com
# Should resolve

# Test internet connectivity
ping -c 3 8.8.8.8
# Should succeed

# Test HTTPS connectivity
curl -I https://google.com
# Should return HTTP 200 or 301

# Test connectivity to VM1 Prometheus
curl -u admin:PASSWORD http://<VM1_PUBLIC_IP>:9090/-/ready
# Should return: Prometheus is Ready.
```

**If connectivity fails:**
- Check NAT gateway configuration
- Check cloud routing tables
- Check security groups allow outbound
- Check network ACLs

#### 2.4 Install K3s

```bash
# Install K3s (choose based on VM2 spec)

# For 4 vCPU, 4GB RAM (standard):
curl -sfL https://get.k3s.io | sh -

# For 1 vCPU, 2GB RAM (minimal):
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
    --disable traefik \
    --disable servicelb \
    --kubelet-arg=max-pods=50" sh -

# Wait and verify
sleep 30
systemctl status k3s

# Setup kubectl
echo "alias kubectl='k3s kubectl'" >> ~/.bashrc
source ~/.bashrc

kubectl get nodes
# Should show 1 node Ready
```

#### 2.5 Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

#### 2.6 Create Kubernetes Secret

```bash
# Use VM1 Prometheus credentials
PROM_USER="admin"
PROM_PASSWORD="<YOUR_VM1_PROMETHEUS_PASSWORD>"

kubectl create namespace monitoring

kubectl create secret generic kubepromsecret \
    --from-literal=username=$PROM_USER \
    --from-literal=password=$PROM_PASSWORD \
    -n monitoring

kubectl get secret kubepromsecret -n monitoring
```

#### 2.7 Create Helm Values (CROSS-CLOUD CONFIG)

**⚠️ KEY DIFFERENCE: Use VM1 PUBLIC IP for remote write!**

```bash
# Get VM1 PUBLIC IP
VM1_PUBLIC_IP="103.x.x.x"  # Replace with actual

# Choose config based on VM2 spec:

# === FOR VM2: 4 vCPU, 4GB RAM (Standard) ===
cat > /root/kube-prometheus-values.yaml <<EOF
prometheus:
  prometheusSpec:
    scrapeInterval: "1m"
    retention: 15d

    # CROSS-CLOUD: Use PUBLIC IP!
    remoteWrite:
    - url: http://${VM1_PUBLIC_IP}:9090/api/v1/write
      basicAuth:
        username:
          name: kubepromsecret
          key: username
        password:
          name: kubepromsecret
          key: password
      queueConfig:
        capacity: 2500
        maxSamplesPerSend: 1000
        maxShards: 200
        batchSendDeadline: 5s
        minBackoff: 30ms
        maxBackoff: 5s

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
              storage: 50Gi

grafana:
  enabled: false

alertmanager:
  enabled: false
EOF

# === FOR VM2: 1 vCPU, 2GB RAM (Minimal) ===
# Use /root/kube-prometheus-values-minimal.yaml from DEPLOYMENT_UNBALANCED.md
# But change remoteWrite URL to use PUBLIC IP!
```

**🔒 Security Note:**
- Remote write URL uses **HTTP** (not HTTPS)
- Traffic goes over **internet**
- Protected by **basic authentication**
- Consider TLS for production (see advanced section)

#### 2.8 Install kube-prometheus-stack

```bash
helm install -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --version 58.2.2 \
    -f /root/kube-prometheus-values.yaml \
    --create-namespace

# Wait for pods (3-5 minutes)
kubectl get pods -n monitoring -w
# Ctrl+C when all Running
```

#### 2.9 Verify Remote Write

**Check Prometheus logs:**
```bash
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 | grep remote_write

# Good logs:
# level=info msg="Remote write sent" ...

# Bad logs (troubleshoot):
# level=error msg="remote write: HTTP status 401" → Auth problem
# level=error msg="connection refused" → Network problem
# level=error msg="timeout" → Firewall or network issue
```

**Test from VM2 manually:**
```bash
# Install curl in a pod
kubectl run test-curl --rm -it --image=curlimages/curl -- sh

# Inside pod:
curl -u admin:PASSWORD http://<VM1_PUBLIC_IP>:9090/-/ready
# Should return: Prometheus is Ready.

# Exit pod
exit
```

**Verify metrics on VM1:**
```bash
# SSH to VM1
curl -u admin:PASSWORD 'http://localhost:9090/api/v1/query?query=up{job=~"kubernetes.*"}'

# Should return metrics from VM2 K8s cluster
```

---

### Phase 3: Import Dashboards

Same as standard deployment:
1. Access Grafana: `http://<VM1_PUBLIC_IP>:3000`
2. Import all 6 dashboards via UI
3. Verify data is showing

---

## ✅ Verification Checklist

```
VM1 (Monitoring):
☐ Prometheus accessible from internet (with auth)
☐ Grafana accessible from browser
☐ Prometheus receiving metrics from VM2
☐ No 401/403 errors in logs

VM2 (Kubernetes):
☐ K3s running
☐ All pods Running in monitoring namespace
☐ No remote write errors in Prometheus pod logs
☐ Can curl VM1 public IP port 9090

Dashboards:
☐ All 6 dashboards imported
☐ Metrics displaying correctly
☐ No "No Data" errors
```

---

## 🔒 Security Considerations

### Current Setup Security Level: ⚠️ MEDIUM

**What's Protected:**
- ✅ Basic authentication (username/password)
- ✅ Firewall enabled
- ✅ Strong password required

**What's NOT Protected:**
- ❌ Traffic not encrypted (HTTP, not HTTPS)
- ❌ Port 9090 exposed to entire internet
- ❌ Susceptible to brute force (rate limiting not configured)

---

### Security Improvements (Optional)

#### 1. Add TLS/HTTPS to Prometheus

**Generate TLS certificate:**
```bash
# On VM1
# Option A: Let's Encrypt (requires domain)
# Option B: Self-signed (development)

# Self-signed example:
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/prometheus/server.key \
  -out /opt/prometheus/server.crt \
  -subj "/CN=monitoring.example.com"

chown prometheus:prometheus /opt/prometheus/server.{key,crt}
chmod 600 /opt/prometheus/server.key
```

**Update auth.yml:**
```yaml
tls_server_config:
  cert_file: /opt/prometheus/server.crt
  key_file: /opt/prometheus/server.key

basic_auth_users:
  admin: "$2y$14$..."
```

**Update remote write URL (VM2):**
```yaml
remoteWrite:
- url: https://<VM1_PUBLIC_IP>:9090/api/v1/write  # HTTPS!
  tlsConfig:
    insecureSkipVerify: true  # For self-signed cert
```

#### 2. Use Cloudflare Tunnel (Advanced)

See separate guide: `DEPLOYMENT_CLOUDFLARE_TUNNEL.md` (coming soon)

Benefits:
- ✅ Zero ports exposed to internet
- ✅ Automatic HTTPS
- ✅ DDoS protection
- ✅ Zero Trust security

#### 3. Restrict Firewall to VM2's IP Range

```bash
# Find VM2's egress IP
# On VM2:
curl ifconfig.me
# Returns: 54.x.x.x

# On VM1: Update firewall
ufw delete allow 9090/tcp
ufw allow from 54.x.x.x to any port 9090 proto tcp

# Or allow entire CIDR if VM2 IP changes
ufw allow from 54.0.0.0/8 to any port 9090 proto tcp
```

#### 4. Add Fail2Ban for Brute Force Protection

```bash
# On VM1
apt install fail2ban

# Configure Prometheus jail (custom rule)
cat > /etc/fail2ban/filter.d/prometheus.conf <<'EOF'
[Definition]
failregex = ^.*"remote_addr":"<HOST>".*"status":401.*$
ignoreregex =
EOF

cat > /etc/fail2ban/jail.d/prometheus.conf <<'EOF'
[prometheus]
enabled = true
port = 9090
filter = prometheus
logpath = /var/log/prometheus/access.log
maxretry = 5
bantime = 3600
EOF

systemctl restart fail2ban
```

---

## 🐛 Troubleshooting

### Issue: Remote Write Failing (401 Unauthorized)

**Symptoms:**
```
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 | grep 401
# level=error msg="remote write: HTTP status 401"
```

**Solutions:**
```bash
# 1. Verify password is correct
kubectl get secret kubepromsecret -n monitoring -o jsonpath='{.data.password}' | base64 -d
# Should match VM1 Prometheus password EXACTLY

# 2. Test auth manually from VM2
kubectl run test-curl --rm -it --image=curlimages/curl -- \
  curl -u admin:PASSWORD -v http://<VM1_PUBLIC_IP>:9090/-/ready

# 3. Check Prometheus auth.yml on VM1
cat /opt/prometheus/auth.yml
# Verify bcrypt hash is correct

# 4. Re-create secret with correct password
kubectl delete secret kubepromsecret -n monitoring
kubectl create secret generic kubepromsecret \
  --from-literal=username=admin \
  --from-literal=password=<CORRECT_PASSWORD> \
  -n monitoring

# 5. Restart Prometheus pod
kubectl rollout restart statefulset kube-prometheus-stack-prometheus -n monitoring
```

---

### Issue: Connection Timeout or Refused

**Symptoms:**
```
level=error msg="remote write: dial tcp 103.x.x.x:9090: connect: connection refused"
# Or: i/o timeout
```

**Solutions:**
```bash
# 1. Check VM2 can reach VM1 public IP
# On VM2:
ping <VM1_PUBLIC_IP>
# Should work

# 2. Check VM2 can reach port 9090
# On VM2:
nc -zv <VM1_PUBLIC_IP> 9090
# Should show: Connection to 103.x.x.x 9090 port [tcp/*] succeeded!

# If fails: Check VM1 firewall
# On VM1:
ufw status | grep 9090
# Should show: 9090/tcp ALLOW Anywhere

# 3. Check Prometheus is listening on all interfaces
# On VM1:
netstat -tlnp | grep 9090
# Should show: 0.0.0.0:9090 (not 127.0.0.1:9090)

# If showing 127.0.0.1 only:
# Edit /etc/systemd/system/prometheus.service
# Add: --web.listen-address=0.0.0.0:9090

# 4. Check VM2 security group allows outbound to internet
# Via cloud console, verify:
# - Outbound rules allow HTTP (port 9090)
# - NAT gateway configured for internet access

# 5. Check cloud network ACLs
# Some clouds have network ACLs separate from security groups
```

---

### Issue: VM2 No Internet Access

**Symptoms:**
```
# On VM2:
curl google.com
# curl: (6) Could not resolve host: google.com
# Or: curl: (7) Failed to connect
```

**Solutions:**
```bash
# Check DNS resolution
cat /etc/resolv.conf
# Should have nameserver (e.g., 8.8.8.8)

# If missing, add:
echo "nameserver 8.8.8.8" >> /etc/resolv.conf

# Check routing
ip route
# Should have default route via gateway

# Check NAT gateway (cloud-specific)
# AWS: Check NAT Gateway in route table
# GCP: Check Cloud NAT configuration
# Azure: Check NAT Gateway or Azure Firewall

# Test with IP (bypass DNS)
ping 8.8.8.8
# If this works but DNS doesn't: DNS issue
# If this fails: routing/NAT issue
```

---

## 📊 Performance Considerations

### Network Latency

**Cross-cloud communication adds latency:**
```
Same cloud (private network):  < 1ms
Cross-cloud (same region):     10-30ms
Cross-cloud (different region): 50-200ms
Cross-cloud (different continent): 200-500ms
```

**Impact:**
- Remote write may be slower
- Increase queue capacity to buffer
- Increase batch send deadline

**Optimization (in values.yaml):**
```yaml
remoteWrite:
- url: http://<VM1_PUBLIC_IP>:9090/api/v1/write
  queueConfig:
    capacity: 5000           # Increase buffer
    maxSamplesPerSend: 2000  # Larger batches
    batchSendDeadline: 10s   # More time to send
    maxShards: 50            # More parallel senders
```

### Bandwidth Usage

**Estimated bandwidth:**
```
Metrics/sec: 10,000 samples/sec
Sample size: ~100 bytes (with labels)
Bandwidth: ~1 MB/sec = ~8 Mbps
Data transfer: ~2.5 TB/month
```

**Cost implications:**
- Most clouds charge for data egress
- VM2 → Internet: Usually charged
- Calculate based on cloud provider pricing

**Optimization:**
- Reduce scrape interval (2m instead of 1m)
- Filter unnecessary metrics
- Use recording rules to pre-aggregate

---

## 💰 Cost Considerations

### Compute Costs
```
VM1: Standard pricing (varies by cloud)
VM2: Standard pricing (varies by cloud)
```

### Data Transfer Costs (Important!)

**Typical egress pricing:**
```
AWS:   $0.09/GB (first 10TB/month)
GCP:   $0.12/GB (to internet)
Azure: $0.087/GB (Zone 1)

Estimated monthly egress (VM2 → VM1):
~2-5 GB/month for monitoring stack only
Cost: ~$0.20 - $0.50/month
```

**⚠️ Can increase significantly if:**
- High cardinality metrics
- Many applications monitored
- Low scrape interval

### Ways to Reduce Costs

1. **Use same cloud provider** (free private network)
2. **Co-locate in same region** (lower latency + cost)
3. **Use cloud peering** (if available between providers)
4. **Optimize metrics** (reduce cardinality)
5. **Increase scrape interval** (less data transferred)

---

## 🎯 When to Use This Setup

### ✅ Good For:

- VM1 and VM2 must be in different clouds
- VM2 doesn't have public IP (cost saving)
- Development/testing cross-cloud setup
- Multi-cloud monitoring strategy
- Temporary setup before migration

### ❌ NOT Recommended For:

- Production with strict latency requirements
- High-volume metrics (bandwidth costs)
- Compliance requiring private networking
- Same cloud provider (use private network instead)

### Better Alternatives:

**If same cloud:**
→ Use private networking (VPC peering, private link)

**If cross-cloud production:**
→ Use VPN or cloud interconnect for private connection

**If high security requirement:**
→ Use Cloudflare Tunnel or similar zero-trust solution

---

## 📚 Related Documentation

- **Standard 2 VM (Same Cloud):** [DEPLOYMENT_2VM.md](DEPLOYMENT_2VM.md)
- **Budget Setup:** [DEPLOYMENT_UNBALANCED.md](DEPLOYMENT_UNBALANCED.md)
- **Requirements:** [REQUIREMENTS.md](REQUIREMENTS.md)
- **Security Review:** [CODE_REVIEW.md](CODE_REVIEW.md)

---

## ✅ Success Criteria

```
Setup Complete:
☐ VM1 accessible from internet (port 9090 + 3000)
☐ VM2 has outbound internet access
☐ Remote write from VM2 → VM1 working
☐ No authentication errors
☐ Metrics appearing in VM1 Prometheus
☐ Grafana dashboards showing data
☐ Latency acceptable for your use case

Security:
☐ Strong password configured (32+ chars)
☐ Basic authentication working
☐ Unauthorized access returns 401
☐ Firewall rules configured
☐ (Optional) TLS configured

Performance:
☐ Remote write queue not backing up
☐ No dropped samples
☐ Acceptable query latency in Grafana
```

---

**Document Version:** 1.0
**Last Updated:** 2025-11-05
**For:** Cross-cloud deployment with VM2 without public IP

---

## 💬 Summary

Setup ini cocok untuk situasi:
- ✅ VM1 dan VM2 di cloud berbeda
- ✅ VM2 tidak punya public IP (cost saving)
- ✅ VM2 punya internet access (outbound)

**Trade-offs:**
- ⚠️ Port 9090 exposed ke internet (secured with basic auth)
- ⚠️ Traffic over internet (higher latency, bandwidth costs)
- ⚠️ HTTP not HTTPS (can be improved with TLS)

**When stable:**
Consider upgrade to:
- 🔒 TLS/HTTPS for encryption
- 🔒 Cloudflare Tunnel for zero-trust
- 🔒 VPN/interconnect for private connection

Good luck with deployment! 🚀
