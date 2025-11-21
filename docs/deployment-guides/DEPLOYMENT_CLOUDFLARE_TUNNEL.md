# Deployment Guide: With Cloudflare Tunnel (Zero-Trust Security)

**Setup Type:** Cross-Cloud with Cloudflare Tunnel
**Security Level:** 🔒🔒🔒 HIGH (Zero-Trust)
**Complexity:** Advanced
**Setup Time:** 5-7 hours

---

## 📋 Overview

Setup ini menggunakan **Cloudflare Tunnel** untuk secure access ke Prometheus, sehingga:
- ✅ **ZERO ports** exposed ke internet di VM1
- ✅ Automatic **HTTPS** encryption
- ✅ **Zero-trust** security model
- ✅ **DDoS** protection by Cloudflare
- ✅ **Access control** via Cloudflare Access

```
┌─────────────────────────────────────────────────────────┐
│         VM2: Kubernetes (Cloud Provider B)              │
│         NO PUBLIC IPv4 ❌                               │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  K3s + kube-prometheus-stack                   │    │
│  │  Remote write to:                              │    │
│  │  https://prometheus.yourdomain.com/api/v1/write│    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                        │
                        │ HTTPS (via Internet)
                        ▼
               ┌─────────────────┐
               │ Cloudflare Edge │
               │ - DDoS protect  │
               │ - HTTPS term    │
               │ - Access control│
               └─────────────────┘
                        │
                        │ Cloudflare Tunnel
                        │ (encrypted)
                        ▼
┌─────────────────────────────────────────────────────────┐
│       VM1: Monitoring Server (Cloud Provider A)         │
│       Public IPv4: Optional (not exposed!) ✅           │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  cloudflared daemon                            │    │
│  │  - Maintains tunnel to Cloudflare              │    │
│  │  - Proxies traffic to local services           │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  Prometheus Server                             │    │
│  │  - Port 9090 (localhost ONLY)                  │    │
│  │  - NOT exposed to internet                     │    │
│  │  - Accessed via tunnel                         │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  Grafana Server                                │    │
│  │  - Port 3000 (localhost ONLY)                  │    │
│  │  - Accessed via tunnel                         │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                        ▲
                        │ HTTPS (via Cloudflare)
                        │ https://grafana.yourdomain.com
                        │
                 ┌──────────────┐
                 │ Admin/Users  │
                 └──────────────┘
```

---

## 🔑 Key Benefits

### Security

✅ **Zero ports exposed**
- Firewall blocks ALL inbound traffic
- Only outbound connection (tunnel)
- No attack surface

✅ **End-to-end encryption**
- HTTPS from VM2 → Cloudflare
- Encrypted tunnel Cloudflare → VM1
- No plaintext traffic

✅ **Zero-trust access**
- Cloudflare Access can require authentication
- Support for SSO (Google, GitHub, etc.)
- IP restrictions, country blocks, etc.

✅ **DDoS protection**
- Cloudflare absorbs DDoS attacks
- Rate limiting
- WAF (Web Application Firewall)

### Operational

✅ **Automatic HTTPS**
- Cloudflare manages SSL certificates
- Auto-renewal
- No Let's Encrypt setup needed

✅ **Domain-based access**
- Use friendly domains (prometheus.yourdomain.com)
- Instead of IP addresses

✅ **Easy to revoke**
- Disable tunnel = instant block
- No need to change firewall rules

---

## 📋 Prerequisites

### Required

1. **Cloudflare account** (Free tier OK)
   - Sign up: https://dash.cloudflare.com/sign-up

2. **Domain name registered**
   - Can be registered anywhere
   - Will point to Cloudflare nameservers

3. **Domain added to Cloudflare**
   - Add site in Cloudflare dashboard
   - Update nameservers at registrar

4. **Both VMs provisioned**
   - VM1: Ubuntu 22.04 (for Prometheus + Grafana)
   - VM2: Ubuntu 22.04 (for Kubernetes)

---

## 🚀 Deployment Steps

### Phase 1: Setup Cloudflare Tunnel

#### 1.1 Add Domain to Cloudflare

**Via Cloudflare Dashboard:**
```
1. Login: https://dash.cloudflare.com/
2. Click: "Add a Site"
3. Enter your domain: example.com
4. Select plan: Free (or paid if you want)
5. Cloudflare will scan DNS records
6. Click: "Continue"
7. Note the nameservers (e.g., elena.ns.cloudflare.com)
```

**Update Nameservers at Registrar:**
```
Go to your domain registrar (Namecheap, GoDaddy, etc.)
Find: DNS Management or Nameservers
Change to Cloudflare nameservers

Wait 5-60 minutes for propagation
```

**Verify:**
```bash
# Check nameservers
dig NS example.com +short
# Should show Cloudflare nameservers
```

#### 1.2 Create Cloudflare Tunnel

**Via Cloudflare Dashboard:**
```
1. Go to: https://one.dash.cloudflare.com/
2. Select your account
3. Navigate: Networks → Tunnels
4. Click: "Create a tunnel"
5. Choose: "Cloudflared"
6. Tunnel name: "monitoring-tunnel" (or any name)
7. Click: "Save tunnel"

⚠️ IMPORTANT: Copy the tunnel token!
Token looks like:
eyJhIjoiODd... (very long string)

Save this - you'll need it on VM1!
```

#### 1.3 Install cloudflared on VM1

**SSH to VM1:**
```bash
ssh ubuntu@<VM1_IP>
sudo su
```

**Install cloudflared:**
```bash
# Add Cloudflare GPG key
mkdir -p /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | tee /usr/share/keyrings/cloudflare-archive-keyring.gpg >/dev/null

# Add repository
echo "deb [signed-by=/usr/share/keyrings/cloudflare-archive-keyring.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/cloudflared.list

# Install
apt update
apt install cloudflared

# Verify
cloudflared --version
```

#### 1.4 Configure Tunnel on VM1

**Authenticate (using tunnel token from step 1.2):**
```bash
cloudflared service install <YOUR_TUNNEL_TOKEN>

# This creates:
# - /etc/cloudflared/config.yml
# - Systemd service
```

**Verify tunnel is running:**
```bash
systemctl status cloudflared

# Should show: active (running)

# Check logs
journalctl -u cloudflared -f
# Should show: "Connection established" or "Registered tunnel"
```

#### 1.5 Create DNS Records & Routes

**Back in Cloudflare Dashboard:**
```
1. Go back to your tunnel page
2. Click: "Add a public hostname"

3. Configure Prometheus:
   Subdomain: prometheus
   Domain: example.com
   Path: (leave empty)
   Type: HTTP
   URL: localhost:9090

   Click: Save hostname

4. Add another public hostname for Grafana:
   Subdomain: grafana
   Domain: example.com
   Path: (leave empty)
   Type: HTTP
   URL: localhost:3000

   Click: Save hostname
```

**This creates:**
- `prometheus.example.com` → VM1 localhost:9090
- `grafana.example.com` → VM1 localhost:3000

**DNS records are automatically created in Cloudflare!**

#### 1.6 Verify Tunnel Works

**Test from your local machine:**
```bash
# Test Prometheus (without auth first)
curl -I https://prometheus.example.com/-/healthy

# Should return: HTTP 200 OK
# If returns 403: Cloudflare blocking, continue to next step
```

---

### Phase 2: Setup VM1 Services (Prometheus + Grafana)

#### 2.1 Basic VM1 Configuration

```bash
# SSH to VM1
ssh ubuntu@<VM1_IP>
sudo su

# Hostname, timezone, updates
hostnamectl set-hostname monitoring-server
timedatectl set-timezone Asia/Jakarta
apt update -y && apt upgrade -y
apt install -y curl wget tar vim htop
```

#### 2.2 Configure Firewall - BLOCK ALL INBOUND!

**⚠️ CRITICAL: Block ALL inbound except SSH**

```bash
apt install -y ufw

# Default policies
ufw default deny incoming
ufw default allow outgoing

# ONLY allow SSH
ufw allow 22/tcp

# Do NOT open ports 9090 or 3000!
# Traffic comes via Cloudflare tunnel

# Enable
ufw enable
ufw status

# Should ONLY show:
# 22/tcp ALLOW Anywhere
```

#### 2.3 Install Prometheus (Localhost Only)

```bash
# Create user
useradd -M -U prometheus
mkdir -p /var/lib/prometheus_data/
chown prometheus:prometheus -R /var/lib/prometheus_data/

# Download
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.53.4/prometheus-2.53.4.linux-amd64.tar.gz
tar -zxvf prometheus-2.53.4.linux-amd64.tar.gz
mv prometheus-2.53.4.linux-amd64 /opt/prometheus
chown prometheus:prometheus -R /opt/prometheus/
```

#### 2.4 Configure Prometheus (With Auth)

**Generate password:**
```bash
PROM_PASSWORD=$(openssl rand -base64 32)
echo "Prometheus Password: $PROM_PASSWORD"
echo "$PROM_PASSWORD" > /root/prometheus-password.txt
chmod 600 /root/prometheus-password.txt

# Generate bcrypt hash at: https://bcrypt.online/
# Cost: 12
```

**Create auth.yml:**
```bash
cat > /opt/prometheus/auth.yml <<'EOF'
basic_auth_users:
  admin: "$2y$12$YOUR_BCRYPT_HASH_HERE"
EOF

/opt/prometheus/promtool check web-config /opt/prometheus/auth.yml
```

**Create prometheus.yml:**
```bash
cat > /opt/prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    basic_auth:
      username: 'admin'
      password: 'YOUR_PLAIN_PASSWORD_HERE'
    static_configs:
      - targets: ['localhost:9090']
EOF

/opt/prometheus/promtool check config /opt/prometheus/prometheus.yml
chown prometheus:prometheus /opt/prometheus/*.yml
```

#### 2.5 Create Prometheus Service (Localhost Only!)

```bash
cat > /etc/systemd/system/prometheus.service <<'EOF'
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/
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
  --web.config.file=/opt/prometheus/auth.yml \
  --web.listen-address=127.0.0.1:9090

[Install]
WantedBy=multi-user.target
EOF

# ⚠️ NOTE: --web.listen-address=127.0.0.1:9090
# Prometheus ONLY listens on localhost!
# NOT accessible from internet directly

systemctl daemon-reload
systemctl enable prometheus.service
systemctl start prometheus.service
systemctl status prometheus.service
```

#### 2.6 Test Prometheus via Tunnel

**From your local machine:**
```bash
# Test health
curl https://prometheus.example.com/-/healthy
# Should return: Prometheus is Healthy.

# Test with auth
curl -u admin:YOUR_PASSWORD https://prometheus.example.com/api/v1/status/runtimeinfo
# Should return JSON

# Test WITHOUT auth (should fail)
curl https://prometheus.example.com/api/v1/status/runtimeinfo
# Should return: 401 Unauthorized ✅
```

#### 2.7 Install Grafana (Localhost Only)

```bash
# Install
apt-get install -y apt-transport-https software-properties-common
mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | tee /etc/apt/sources.list.d/grafana.list
apt-get update
apt-get install grafana

# Configure to listen on localhost only
vim /etc/grafana/grafana.ini

# Find and edit:
[server]
http_addr = 127.0.0.1
http_port = 3000

# Start
systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server
```

#### 2.8 Access Grafana via Tunnel

**From browser:**
```
https://grafana.example.com

Login: admin / admin
Change password when prompted
```

**Configure Prometheus data source:**
```
Configuration → Data Sources → Add Prometheus

Name: Prometheus
URL: http://localhost:9090

Auth: ☑ Basic auth
User: admin
Password: <YOUR_PROMETHEUS_PASSWORD>

Save & Test → Should work!
```

---

### Phase 3: Setup VM2 (Kubernetes)

#### 3.1 Basic VM2 Setup

```bash
# Access VM2 (via console, bastion, or VPN)
sudo su

hostnamectl set-hostname k8s-node
timedatectl set-timezone Asia/Jakarta
apt update -y && apt upgrade -y
```

#### 3.2 Verify Internet Connectivity

```bash
# Test DNS
nslookup google.com

# Test HTTPS connectivity
curl -I https://google.com

# Test tunnel access to Prometheus
curl -u admin:PASSWORD https://prometheus.example.com/-/ready
# Should return: Prometheus is Ready.
```

#### 3.3 Install K3s

```bash
# Standard or minimal (choose based on VM2 spec)
curl -sfL https://get.k3s.io | sh -

# Wait and verify
sleep 30
systemctl status k3s

kubectl get nodes
```

#### 3.4 Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

#### 3.5 Create Kubernetes Secret

```bash
PROM_USER="admin"
PROM_PASSWORD="<YOUR_PROMETHEUS_PASSWORD>"

kubectl create namespace monitoring

kubectl create secret generic kubepromsecret \
    --from-literal=username=$PROM_USER \
    --from-literal=password=$PROM_PASSWORD \
    -n monitoring
```

#### 3.6 Create Helm Values (HTTPS + Domain!)

**⚠️ KEY: Use HTTPS and domain name!**

```bash
cat > /root/kube-prometheus-values-cloudflare.yaml <<'EOF'
prometheus:
  prometheusSpec:
    scrapeInterval: "1m"
    retention: 15d

    # CLOUDFLARE TUNNEL: Use HTTPS + domain!
    remoteWrite:
    - url: https://prometheus.example.com/api/v1/write
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

# For 1 vCPU, 2GB VM: use minimal config from DEPLOYMENT_UNBALANCED.md
# But change URL to: https://prometheus.example.com/api/v1/write
```

#### 3.7 Install kube-prometheus-stack

```bash
helm install -n monitoring kube-prometheus-stack \
    prometheus-community/kube-prometheus-stack \
    --version 58.2.2 \
    -f /root/kube-prometheus-values-cloudflare.yaml \
    --create-namespace

kubectl get pods -n monitoring -w
```

#### 3.8 Verify Remote Write

```bash
# Check logs
kubectl logs -n monitoring kube-prometheus-stack-prometheus-0 | grep remote_write

# Should show successful sends, no errors

# Verify metrics on VM1 (via browser)
# Go to: https://prometheus.example.com
# Query: up{job=~"kubernetes.*"}
# Should return metrics!
```

---

### Phase 4: Import Dashboards

**Via browser:**
```
https://grafana.example.com

Import all 6 dashboards:
- k8s-api-server.json
- k8s-clusters.json
- k8s-namespaces.json
- k8s-nodes.json
- k8s-pods.json
- node-exporter-full.json
```

---

## 🔒 Advanced Security: Cloudflare Access

**Add authentication layer untuk Prometheus:**

#### Enable Cloudflare Access

**Via Cloudflare Dashboard:**
```
1. Go to: Access → Applications
2. Click: "Add an application"
3. Select: "Self-hosted"

4. Configure application:
   Name: Prometheus
   Subdomain: prometheus
   Domain: example.com

5. Add policy:
   Policy name: Admin Access
   Action: Allow

   Configure rules:
   - Include: Emails ending in @yourcompany.com
   OR
   - Include: Email: your@email.com

6. Save application
```

**Now:**
- Accessing `https://prometheus.example.com` requires login!
- Can use Google, GitHub, or email auth
- Extra layer on top of basic auth

**Same for Grafana:**
- Create separate application for `grafana.example.com`
- Different access policies if needed

---

## ✅ Verification Checklist

```
Cloudflare Tunnel:
☐ Tunnel created and active
☐ cloudflared service running on VM1
☐ DNS records pointing to tunnel
☐ HTTPS working (auto certificate)

VM1 (Monitoring):
☐ Prometheus listening on 127.0.0.1:9090 ONLY
☐ Grafana listening on 127.0.0.1:3000 ONLY
☐ Firewall blocking all ports except SSH
☐ Services accessible via tunnel URLs
☐ Basic auth working

VM2 (Kubernetes):
☐ Can reach prometheus.example.com via HTTPS
☐ Remote write working (no errors)
☐ Metrics appearing in VM1

Dashboards:
☐ All 6 dashboards imported
☐ Metrics displaying correctly
☐ Accessible via grafana.example.com

Security:
☐ No ports exposed to internet on VM1
☐ All traffic encrypted (HTTPS)
☐ Basic auth + Cloudflare Access working
☐ Unauthorized access blocked
```

---

## 💰 Cost Considerations

### Cloudflare Costs

**Free Tier includes:**
- ✅ Unlimited tunnels
- ✅ Unlimited bandwidth
- ✅ Basic DDoS protection
- ✅ SSL certificates

**Cloudflare Access:**
- Free: Up to 50 users
- Paid: $3/user/month (for more users)

### Compute Costs

Same as standard deployment:
- VM1: Compute costs
- VM2: Compute costs

### Data Transfer

**✅ Advantage:**
- Cloudflare provides free bandwidth
- No egress charges through tunnel!
- Can save $$ on data transfer

---

## 🐛 Troubleshooting

### Issue: Tunnel Status "Down"

```bash
# Check cloudflared service
systemctl status cloudflared

# Check logs
journalctl -u cloudflared -n 100

# Common issues:
# - Invalid tunnel token
# - Network connectivity
# - Firewall blocking outbound

# Restart tunnel
systemctl restart cloudflared
```

### Issue: 502 Bad Gateway

**Symptoms:**
```
https://prometheus.example.com → 502 Bad Gateway
```

**Solutions:**
```bash
# Check Prometheus is running
systemctl status prometheus

# Check Prometheus is on localhost:9090
netstat -tlnp | grep 9090
# Should show: 127.0.0.1:9090

# Check tunnel config
cat /etc/cloudflared/config.yml

# Should have correct service URL
# url: http://localhost:9090
```

### Issue: Remote Write Authentication Failed

```bash
# Verify URL uses HTTPS
kubectl get prometheus -n monitoring kube-prometheus-stack-prometheus -o yaml | grep url

# Should be: https://prometheus.example.com (NOT http://)

# Test auth manually
curl -u admin:PASSWORD https://prometheus.example.com/-/ready
```

---

## 📊 Performance vs Standard Setup

| Aspect | Standard (Public IP) | Cloudflare Tunnel |
|--------|---------------------|-------------------|
| Latency | Direct | +5-20ms (tunnel overhead) |
| Security | Medium | High (zero-trust) |
| Setup Complexity | Simple | Medium |
| Port Exposure | Ports exposed | Zero exposure |
| DDoS Protection | None | Cloudflare protected |
| Cost | No extra | Free tier sufficient |

---

## 🎯 When to Use This Setup

### ✅ Perfect For:

- Production deployments requiring high security
- Compliance requirements (zero-trust, no exposed ports)
- Cross-cloud with security concerns
- Want DDoS protection
- Want automatic HTTPS
- Multi-tenant with different access levels

### ⚠️ Consider Alternatives If:

- Latency critical (direct connection faster)
- Simple development setup (overkill)
- Don't have domain name
- Want simplest possible setup

---

## 📚 Related Documentation

- **Cross-Cloud Simple:** [DEPLOYMENT_CROSS_CLOUD.md](DEPLOYMENT_CROSS_CLOUD.md)
- **Standard 2 VM:** [DEPLOYMENT_2VM.md](DEPLOYMENT_2VM.md)
- **Budget Setup:** [DEPLOYMENT_UNBALANCED.md](DEPLOYMENT_UNBALANCED.md)

---

## ✅ Success Summary

```
✅ Zero ports exposed to internet
✅ All traffic HTTPS encrypted
✅ Zero-trust security model
✅ DDoS protection by Cloudflare
✅ Automatic SSL certificate management
✅ Domain-based access
✅ Optional SSO authentication
✅ Free (Cloudflare Free tier)
✅ Professional setup
```

---

**Document Version:** 1.0
**Last Updated:** 2025-11-05
**Security Level:** 🔒🔒🔒 HIGH

**Perfect for production deployments! 🚀**
