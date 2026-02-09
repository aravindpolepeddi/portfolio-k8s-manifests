# Portfolio Kubernetes Manifests

> GitOps-powered Kubernetes deployment manifests for production portfolio platform

[![Live Site](https://img.shields.io/badge/Live-aravindpolepeddi.uk-blue)](https://aravindpolepeddi.uk)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.34-326CE5?logo=kubernetes)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-orange?logo=argo)](https://argo-cd.readthedocs.io/)
[![Security](https://img.shields.io/badge/Security-OPA%20Gatekeeper-green)](https://www.openpolicyagent.org/)

## 🎯 Overview

This repository contains all Kubernetes manifests and configurations for deploying a production-grade portfolio website on DigitalOcean Kubernetes. The deployment leverages GitOps principles via ArgoCD for automated, declarative infrastructure management.

**Live Site:** [https://aravindpolepeddi.uk](https://aravindpolepeddi.uk)  
**Application Repo:** [portfolio-app](https://github.com/aravindpolepeddi/portfolio-app)

---

## 🏗️ Architecture

```
Internet User
      ↓
https://aravindpolepeddi.uk (Cloudflare DNS)
      ↓
DigitalOcean LoadBalancer (24.144.65.233)
      ↓
NGINX Ingress Controller (NodePorts: 30695/31499)
      ↓
portfolio-frontend-service (ClusterIP :8080)
      ↓
portfolio-frontend pods (3 replicas)
      ↓
Static Astro site served by Nginx
```

---

## 📁 Repository Structure

```
portfolio-k8s-manifests/
├── k8s/
│   ├── namespace.yaml           # portfolio-dev namespace
│   ├── deployment.yaml          # 3 replicas, resource limits
│   ├── service.yaml             # ClusterIP service
│   └── ingress.yaml             # Domain routing + SSL
├── policies/
│   ├── no-privileged-containers.yaml   # Security policy
│   └── required-resources.yaml         # Resource enforcement
├── cert-manager/
│   └── letsencrypt-issuer.yaml  # SSL certificate issuer
├── argocd/
│   └── application.yaml         # ArgoCD app definition
└── README.md
```

---
## 🚀 Deployment Flow

### **GitOps Workflow**

1. **Code Change** → Developer pushes to [portfolio-app](https://github.com/aravindpolepeddi/portfolio-app)
2. **CI Pipeline** → GitHub Actions builds Docker image
3. **Security Scan** → Trivy scans for vulnerabilities
4. **Manifest Update** → CI updates `deployment.yaml` with new image tag
5. **Git Commit** → Changes pushed to this repo
6. **ArgoCD Sync** → Detects change, syncs to cluster
7. **Rolling Update** → Kubernetes deploys new version (zero downtime)
8. **Live** → Site updated at https://aravindpolepeddi.uk

**Average deployment time:** 4-6 minutes from code push to production
---

## 🛠️ Technology Stack

### **Core Infrastructure**
- **Kubernetes:** DigitalOcean Kubernetes (v1.34)
- **Nodes:** 2x workers (4GB RAM, 2 vCPU each)
- **Region:** NYC1

Setup DigitalOcean Kubernetes Cluster:
```bash
doctl kubernetes cluster create portfolio-cluster \
  --region nyc1 \
  --node-pool "name=workers;size=s-2vcpu-4gb;count=2"
```
Configure kubectl Access:
```bash
doctl kubernetes cluster kubeconfig save portfolio-cluster
kubectl config use-context do-nyc1-portfolio-cluster
```

### **Networking & Routing**
- **Ingress:** NGINX Ingress Controller v1.11.1
- **LoadBalancer:** DigitalOcean LoadBalancer
- **DNS:** Cloudflare DNS
- **Domain:** aravindpolepeddi.

Setup NGINX Ingress Controller (Reverse proxy sitting in front of your app, Routes traffic from LoadBalancer to your services, Exposes services via NodePorts (30695, 31499))
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/baremetal/deploy.yaml
```
---

### **Security & Compliance**
- **SSL/TLS:** Cert-Manager v1.13 + Let's Encrypt
- **Policy Enforcement:** OPA Gatekeeper
- **Security Scanning:** Trivy (in CI pipeline)
Setup Cert Manager (obtains SSL certificates from Let's Encrypt, Handles the ACME HTTP-01 challenge process, 
Stores certificates as Kubernetes Secrets)
Components installed:
cert-manager controller
cert-manager-webhook
cert-manager-cainjector
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```
Created Let's Encrypt ClusterIssuer (Tells Cert-Manager how to get certificates, Uses HTTP-01 challenge)
Refer : cert-manager/letsencrypt-issuer.yaml

---

### **GitOps & Automation**
- **CD Tool:** ArgoCD
- **Auto-Sync:** Enabled
- **Self-Heal:** Enabled

Watches this repo and syncs changes to cluster
Components installed:

argocd-server (Web UI)
argocd-repo-server (Git repo watcher)
argocd-application-controller (Sync engine)
argocd-redis (Cache)
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Access ArgoCD UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open: https://localhost:8080
# Username: admin
# Password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
---

### ** OPA Gatekeeper**

**Purpose:** Policy enforcement and security compliance, Blocks non-compliant deployments

**Installation:**
```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

**Active Policies:**
1. **No Privileged Containers** - Blocks containers requesting privileged access
2. **Required Resource Limits** - Enforces CPU/memory limits on all pods

**Verify Policies:**
```bash
kubectl get constraints
```
---

## 🌐 Domain & DNS Setup

### **Domain Configuration**

**Registrar:** Cloudflare  
**Domain:** aravindpolepeddi.uk

### **DNS Records (Cloudflare)**

```
Type: A
Name: @
Value: 24.144.65.233
Proxy: OFF (☁️ gray cloud)
TTL: Auto

Type: A
Name: www
Value: 24.144.65.233
Proxy: OFF (☁️ gray cloud)
TTL: Auto
```

⚠️ **Critical:** Cloudflare proxy must be **OFF** (gray cloud) for Let's Encrypt HTTP-01 challenge to work.

### **Verification**

```bash
dig aravindpolepeddi.uk +short
# Should return: 24.144.65.233
```

---

## 🔒 SSL Certificate Management

### **ClusterIssuer Configuration**

**File:** `cert-manager/letsencrypt-issuer.yaml`


### **Certificate Lifecycle**

1. Ingress created with `cert-manager.io/cluster-issuer` annotation
2. Cert-Manager automatically creates Certificate resource
3. ACME HTTP-01 challenge initiated
4. Let's Encrypt verifies domain ownership
5. Certificate issued (valid 90 days)
6. Secret `portfolio-tls` created with cert + key
7. Auto-renewal before expiration

### **Check Certificate Status**

```bash
kubectl get certificate -n portfolio-dev
kubectl describe certificate portfolio-tls -n portfolio-dev
```

---

## ⚖️ Load Balancer Configuration

### **DigitalOcean LoadBalancer**

**IP Address:** 24.144.65.233  
**Type:** Manual (not managed by Kubernetes)
Why Manual ? because
I didn't have the DigitalOcean Cloud Controller Manager installed, so Kubernetes couldn't auto-create LoadBalancers

### **Forwarding Rules**

```
HTTP:
  Entry Port: 80
  Target Port: 30695 (NGINX Ingress HTTP NodePort)
  Protocol: HTTP

HTTPS:
  Entry Port: 443
  Target Port: 31499 (NGINX Ingress HTTPS NodePort)
  Protocol: HTTPS (Passthrough)
```

### **Health Check**

```
Protocol: HTTP
Port: 30695
Path: /healthz
Interval: 10 seconds
Timeout: 5 seconds
Healthy Threshold: 5
Unhealthy Threshold: 3
```

---

## 🔥 Firewall Rules
LoadBalancer couldn't reach worker nodes on NodePorts.

### **Required Configuration**
```bash
doctl compute firewall add-rules 8f7ea438-f8f2-45d2-b753-2141f4a67256 \
  --inbound-rules "protocol:tcp,ports:30000-32767,sources:load_balancer_uid:YOUR_LB_ID"
```
**What this did:**
Allowed the LoadBalancer to access **all NodePorts (30000-32767)** on worker nodes.


### **Inbound Rules**

```bash
# Allow HTTP/HTTPS from anywhere
protocol:tcp,ports:80,address:0.0.0.0/0
protocol:tcp,ports:443,address:0.0.0.0/0

# Allow NodePort range from LoadBalancer
protocol:tcp,ports:30000-32767,sources:load_balancer_uid:cca544fa-9352-443e-9c92-1c681dd3677e
```

---

## 📦 Application Deployment

### **Deployment Specs**

```yaml
Replicas: 3
Image: aravind1843/portfolio:SHA
Resources:
  Requests:
    CPU: 50m
    Memory: 32Mi
  Limits:
    CPU: 100m
    Memory: 64Mi
```

### **Service Configuration**

```yaml
Type: ClusterIP
Port: 8080
TargetPort: 80 (container port)
Selector: app=portfolio
```

### **Ingress Configuration**

```yaml
Hosts:
  - aravindpolepeddi.uk
  - www.aravindpolepeddi.uk
TLS:
  - secretName: portfolio-tls
Backend: portfolio-frontend-service:8080
```

---

## 🔧 Setup Instructions

### **Prerequisites**

- DigitalOcean account with Kubernetes cluster
- `kubectl` CLI installed
- `doctl` CLI configured
- Domain registered (Cloudflare recommended)

### **1. Configure kubectl Context**

```bash
# Get cluster credentials
doctl kubernetes cluster kubeconfig save portfolio-cluster

# Verify context
kubectl config current-context
# Should show: do-nyc1-portfolio-cluster

# Verify connection
kubectl get nodes
```

### **2. Install Core Components**

```bash
# NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/baremetal/deploy.yaml

# Cert-Manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# OPA Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

### **3. Deploy Application**

```bash
# Clone this repo
git clone https://github.com/aravindpolepeddi/portfolio-k8s-manifests.git
cd portfolio-k8s-manifests

# Apply manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/
kubectl apply -f cert-manager/
kubectl apply -f argocd/application.yaml

# Apply OPA policies
kubectl apply -f policies/
sleep 5
kubectl apply -f policies/  # Apply twice for ConstraintTemplates + Constraints
```

### **5. Verify Deployment**

```bash
# Check pods
kubectl get pods -n portfolio-dev

# Check ingress
kubectl get ingress -n portfolio-dev

# Check certificate
kubectl get certificate -n portfolio-dev

# Test site
curl -I https://aravindpolepeddi.uk
```

---

## 🔄 Making Changes

### **Update Application Code**

1. Make changes in [portfolio-app](https://github.com/aravindpolepeddi/portfolio-app)
2. Commit and push to `main` branch
3. GitHub Actions builds new image
4. CI updates `k8s/deployment.yaml` in this repo
5. ArgoCD syncs automatically
6. Changes live in ~5 minutes

### **Update Kubernetes Configuration**

```bash
# Edit manifests
nano k8s/deployment.yaml

# Commit changes
git add .
git commit -m "update: increase replicas to 5"
git push

# ArgoCD syncs automatically (or manual sync in UI)
```

### **Rollback**

```bash
# Option 1: Git revert
git revert HEAD
git push

# Option 2: ArgoCD UI
# Navigate to History → Rollback

# Option 3: kubectl
kubectl rollout undo deployment portfolio-frontend -n portfolio-dev
```

---

## 📊 Monitoring & Observability

### **Check Application Health**

```bash
# Pod status
kubectl get pods -n portfolio-dev

# Deployment status
kubectl get deployment portfolio-frontend -n portfolio-dev

# Service endpoints
kubectl get endpoints -n portfolio-dev

# Ingress status
kubectl describe ingress portfolio-ingress -n portfolio-dev
```

### **View Logs**

```bash
# Application logs
kubectl logs -n portfolio-dev deployment/portfolio-frontend -f

# Ingress Controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller -f

# Cert-Manager logs
kubectl logs -n cert-manager deployment/cert-manager -f
```

### **ArgoCD Dashboard**

```bash
# Port forward to ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open browser: https://localhost:8080
# Username: admin
# Password: (see command above in ArgoCD section)
```

---

## 🔒 Security Policies

### **Policy 1: No Privileged Containers**

**File:** `policies/no-privileged-containers.yaml`

**Purpose:** Prevents containers from running with elevated privileges

**Enforcement:** Blocks any pod requesting `securityContext.privileged: true`

### **Policy 2: Required Resource Limits**

**File:** `policies/required-resources.yaml`

**Purpose:** Ensures all containers have CPU and memory limits defined

**Enforcement:** Blocks pods missing:
- `resources.requests.cpu`
- `resources.requests.memory`
- `resources.limits.cpu`
- `resources.limits.memory`

### **Test Policy Enforcement**

```bash
# This should be BLOCKED
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged
  namespace: portfolio-dev
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
EOF

# Expected: Error from admission webhook
```

---

## 📈 Metrics & Performance

### **Deployment Metrics**

- **Deployment Time:** 4-6 minutes (code push to production)
- **Zero Downtime:** ✅ Rolling updates
- **Auto-Healing:** ✅ Failed pods restarted automatically
- **Scalability:** Horizontal pod autoscaling ready

### **Resource Usage**

```
Per Pod:
  CPU Request: 50m (0.05 cores)
  Memory Request: 32Mi
  CPU Limit: 100m (0.1 cores)
  Memory Limit: 64Mi

Total (3 replicas):
  CPU: 150m request, 300m limit
  Memory: 96Mi request, 192Mi limit
```

### **Availability**

- **Replicas:** 3 (high availability)
- **Pod Disruption Budget:** Ready for implementation
- **Health Checks:** Readiness + Liveness probes configured

---

## 🐛 Troubleshooting

### **Site Not Loading**

```bash
# Check pods are running
kubectl get pods -n portfolio-dev

# Check service has endpoints
kubectl get endpoints portfolio-frontend-service -n portfolio-dev

# Check ingress
kubectl describe ingress portfolio-ingress -n portfolio-dev

# Test from within cluster
kubectl run test --rm -it --image=nginx -- curl http://portfolio-frontend-service.portfolio-dev:8080
```

### **SSL Certificate Issues**

```bash
# Check certificate status
kubectl get certificate -n portfolio-dev
kubectl describe certificate portfolio-tls -n portfolio-dev

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager -f

# Check challenges
kubectl get challenge -n portfolio-dev
kubectl describe challenge -n portfolio-dev
```

### **ArgoCD Not Syncing**

```bash
# Check ArgoCD application status
kubectl get application -n argocd

# Force sync
kubectl patch application portfolio-dev -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# Or use ArgoCD UI → Sync button
```

### **LoadBalancer Health Check Failing**

```bash
# Check NodePorts
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Test NodePort directly
curl -I http://WORKER_NODE_IP:NODE_PORT/healthz

# Check firewall rules
doctl compute firewall get 8f7ea438-f8f2-45d2-b753-2141f4a67256
```

---

## 💰 Cost Breakdown

### **DigitalOcean**

- Kubernetes Cluster: **$24/month** (2x $12 nodes)
- LoadBalancer: **$12/month**
- **Total:** ~$36/month

### **Domain**

- aravindpolepeddi.uk: **~$10/year** (Cloudflare)

### **Free Tier**

- Let's Encrypt SSL: **Free**
- GitHub Actions: **2000 min/month free**
- Docker Hub: **Unlimited public images**

### **Optimization Tips**

- Delete cluster when not in use (demo/portfolio purposes)
- Use smaller nodes (2GB RAM instead of 4GB)
- Reduce replica count to 1-2
- Use DigitalOcean credits for new accounts ($200/60 days)

---

## 📚 Related Repositories

- **Application Code:** [portfolio-app](https://github.com/aravindpolepeddi/portfolio-app)
- **CI/CD Pipeline:** GitHub Actions in portfolio-app repo
- **Docker Images:** [Docker Hub](https://hub.docker.com/r/aravind1843/portfolio)

---

## 👤 Author

**Aravind Polepeddi**

- Portfolio: [aravindpolepeddi.uk](https://aravindpolepeddi.uk)
- GitHub: [@aravindpolepeddi](https://github.com/aravindpolepeddi)
- LinkedIn: [linkedin.com/in/aravindpolepeddi](https://linkedin.com/in/aravindpolepeddi)

---