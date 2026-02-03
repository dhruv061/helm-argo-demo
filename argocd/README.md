# ArgoCD Setup and Configuration Guide

Complete guide for deploying and managing applications with ArgoCD, covering setup, two-application architecture, image updates, and troubleshooting.

---

## Table of Contents

- [Repository Information](#repository-information)
- [Two-Application Architecture](#two-application-architecture)
- [Quick Start Guide](#quick-start-guide)
- [Image Update Workflow](#image-update-workflow)
- [Common Operations](#common-operations)
- [Troubleshooting](#troubleshooting)
- [Understanding OutOfSync Status](#understanding-outofsync-status)
- [Repository Structure](#repository-structure)

---

## Repository Information

- **Repository**: https://github.com/dhruv061/helm-argo-demo.git
- **Branch**: master
- **Helm Chart Path**: `artha-helm-chart/`
- **Applications**: `demo-artha-app` and `demo-gateway-app`

---

## Two-Application Architecture

This setup uses **two independent ArgoCD applications** for better separation of concerns:

### 1. Application Components (`demo-artha-app`)
**File**: `argocd/demo-artha-app/argocd-application.yaml`

**Manages**:
- ✅ Admin Panel deployments
- ✅ Backend API deployments  
- ✅ Frontend Web deployments
- ✅ Services (admin-service, backend-service, frontend-service)
- ✅ HPA (Horizontal Pod Autoscalers)
- ✅ PDB (Pod Disruption Budgets)

**Values File**: `application-values.yaml` only

**Image Updater**: ✅ Enabled - automatically updates image tags from ACR

### 2. Gateway API Resources (`demo-gateway-app`)
**File**: `argocd/demo-gateway-app/argocd-application.yaml`

**Manages**:
- ✅ Gateway configuration
- ✅ HTTPRoutes for domains
- ✅ TLS certificates
- ✅ Domain routing rules

**Values File**: `gateway-values.yaml` only

**Image Updater**: ❌ Not needed (no container images)

### Benefits of This Setup

**✅ Separation of Concerns**
- Application deployments managed separately from gateway config
- Changes to routing don't affect app deployments
- Can update gateway without redeploying apps

**✅ Independent Lifecycle**
- Deploy new app versions without touching gateway
- Update gateway routes without restarting apps
- Different sync policies if needed

**✅ Clear Ownership**
- **demo-artha-app**: Owns deployments, services, HPAs, PDBs
- **demo-gateway-app**: Owns gateway, HTTPRoutes, certificates

**✅ Image Updater**
- Only runs on demo-artha-app (where images exist)
- Gateway app doesn't need image tracking

---

## Quick Start Guide

### 1. Install ArgoCD (if not already installed)

```bash
# Create argocd namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### 2. Access ArgoCD UI

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo

# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open browser: https://localhost:8080
# Username: admin
# Password: (from above command)
```

### 3. Deploy Both Applications

```bash
# Deploy application components
kubectl apply -f argocd/demo-artha-app/argocd-application.yaml

# Deploy gateway resources
kubectl apply -f argocd/demo-gateway-app/argocd-application.yaml

# Verify both are created
kubectl get applications -n argocd
```

### 4. Sync Applications

#### Via ArgoCD UI:
1. Open ArgoCD UI at https://localhost:8080
2. You'll see two applications:
   - **demo-artha-app** (OutOfSync)
   - **demo-gateway-app** (OutOfSync)
3. Click on **demo-artha-app** → **Sync** → **Synchronize**
4. Click on **demo-gateway-app** → **Sync** → **Synchronize**

#### Via CLI:
```bash
# Install ArgoCD CLI (optional)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Login to ArgoCD
argocd login localhost:8080 --username admin --insecure

# Sync both applications
argocd app sync demo-artha-app
argocd app sync demo-gateway-app

# Wait for sync to complete
argocd app wait demo-artha-app --timeout 300
argocd app wait demo-gateway-app --timeout 300
```

### 5. Verify Deployment

```bash
# Check application status
kubectl get applications -n argocd

# Check deployed resources
kubectl get all -n default

# Check pods
kubectl get pods -n default

# Check services
kubectl get svc -n default
```

---

## Image Update Workflow

### Without Image Updater (Manual)

1. CI/CD pushes image to ACR: `helmtest.azurecr.io/arthanode:v1.2.0`
2. **You manually update** `application-values.yaml`:
   ```yaml
   applications:
     backend:
       image:
         tag: v1.2.0
   ```
3. Commit and push to GitHub
4. ArgoCD shows "OutOfSync"
5. You sync via UI

### With Image Updater (Automatic Tag Detection)

1. CI/CD pushes image to ACR: `helmtest.azurecr.io/arthanode:v1.2.0`
2. **Image Updater automatically**:
   - Detects new tag in ACR (every 2 minutes)
   - Updates `application-values.yaml` with new tag
   - Commits to GitHub: `build: automatic update of arthanode to v1.2.0`
3. **demo-artha-app** shows "OutOfSync"
4. You manually sync via ArgoCD UI to deploy

**To enable Image Updater**: See [IMAGE-UPDATER-SETUP.md](IMAGE-UPDATER-SETUP.md)

---

## Common Operations

### Update Application Image

Happens automatically via Image Updater, or manually:

```bash
# Edit application-values.yaml
nano artha-helm-chart/application-values.yaml
# Change image tag
git commit -m "Update backend to v1.2.0"
git push origin master
# Sync demo-artha-app in ArgoCD UI
```

### Add New Domain to Gateway

```bash
# Edit gateway-values.yaml
nano artha-helm-chart/gateway-values.yaml
# Add new domain to domains[] array
git commit -m "Add new domain example.com"
git push origin master
# Sync demo-gateway-app in ArgoCD UI
```

### Scale Applications

```bash
# Edit application-values.yaml
nano artha-helm-chart/application-values.yaml
# Change replicaCount or HPA settings
git commit -m "Scale backend to 3 replicas"
git push origin master
# Sync demo-artha-app in ArgoCD UI
```

### Useful CLI Commands

```bash
# Get application status
argocd app get demo-artha-app
argocd app get demo-gateway-app

# View sync history
argocd app history demo-artha-app

# Rollback to previous version
argocd app rollback demo-artha-app

# Delete application (WARNING: removes all resources)
argocd app delete demo-artha-app
```

---

## Troubleshooting

### Resources Already Exist Error

This happens when resources are managed by old app. Solution:

```bash
# Delete the conflicting resources
kubectl delete pdb backend-pdb frontend-pdb -n default
kubectl delete deployment admin backend frontend -n default
kubectl delete hpa admin-hpa backend-hpa frontend-hpa -n default

# Then sync the new application
```

### Both Apps Show Same Resources

Check the exclude values - each app should have different resources:
- **demo-artha-app**: `gateway.enabled=false`
- **demo-gateway-app**: `applications.*.enabled=false`

### Cannot Access ArgoCD UI

Make sure port-forward is running:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Image Pull Errors

Ensure image registry credentials are configured:

```bash
kubectl get secret -n default
# You should see image pull secrets for ACR
```

### Sync Fails

Check application details:

```bash
argocd app get demo-artha-app
kubectl get events -n default
```

### Image Updater Not Working

Check:

```bash
# View Image Updater logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f

# Verify ACR credentials secret exists
kubectl get secret acr-credentials -n argocd

# Verify Git credentials secret exists
kubectl get secret git-creds -n argocd
```

---

## Understanding OutOfSync Status

### Why Does ArgoCD Show "OutOfSync"?

When Gateway API resources (HTTPRoutes, Certificates, etc.) are deployed, **controllers automatically modify and normalize certain fields**. This causes ArgoCD to detect differences between Git and the cluster.

### Common Controller Modifications

#### 1. HTTPRoute - ParentRefs Normalization

**What you write in Git:**
```yaml
spec:
  parentRefs:
    - name: nginx-gateway
      namespace: nginx-gateway
```

**What the controller adds:**
```yaml
spec:
  parentRefs:
    - kind: Gateway  # ← Controller adds this
      name: nginx-gateway
      namespace: nginx-gateway
```

#### 2. HTTPRoute - BackendRefs Normalization

**What you write:**
```yaml
backendRefs:
  - name: frontend-service
    port: 80
```

**What controller adds:**
```yaml
backendRefs:
  - kind: Service  # ← Controller adds this
    name: frontend-service
    port: 80
```

#### 3. Certificate - Secret Template

**What you write:**
```yaml
spec:
  secretName: my-tls-cert
```

**What cert-manager adds:**
```yaml
spec:
  secretName: my-tls-cert
  secretTemplate:  # ← cert-manager adds this
    labels:
      controller: cert-manager
```

### The Solution: ignoreDifferences

ArgoCD applications are configured with `ignoreDifferences` to ignore controller-managed fields:

```yaml
spec:
  ignoreDifferences:
    # Ignore HTTPRoute field normalization
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      jqPathExpressions:
        - '.spec.parentRefs[]?.kind'
        - '.spec.rules[]?.backendRefs[]?.kind'
    
    # Ignore Certificate fields added by cert-manager
    - group: cert-manager.io
      kind: Certificate
      jqPathExpressions:
        - '.spec.secretTemplate'
        - '.spec.privateKey.rotationPolicy'
    
    # Ignore Gateway status
    - group: gateway.networking.k8s.io
      kind: Gateway
      jqPathExpressions:
        - '.status'
    
    # Ignore Deployment replicas (modified by HPA)
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

### What This Does

✅ ArgoCD will **ignore** differences in these specific fields  
✅ You'll only see "OutOfSync" for **real changes**  
✅ Controllers can still manage these fields  
✅ No manual syncing needed for controller modifications  

### Expected OutOfSync Behavior

**Image Updater Updates**: When Image Updater updates `application-values.yaml` with new image tags, ArgoCD will show "OutOfSync". This is **correct behavior** for manual sync workflow!

---

## Repository Structure

```
ingress-to-gateway-api/
└── Helm_Argo/
    ├── argocd/
    │   ├── README.md                           # This file
    │   ├── IMAGE-UPDATER-SETUP.md             # Image updater guide
    │   ├── demo-artha-app/
    │   │   └── argocd-application.yaml        # Application components
    │   └── demo-gateway-app/
    │       └── argocd-application.yaml        # Gateway resources
    │
    └── artha-helm-chart/
        ├── Chart.yaml
        ├── README.md                           # Helm chart documentation
        ├── application-values.yaml             # App config (image tags)
        ├── gateway-values.yaml                 # Gateway config
        └── templates/
            ├── applications/                   # App deployments
            └── gateway/                        # Gateway resources
```

---

## Next Steps

1. ✅ ArgoCD Applications configured and validated
2. Install ArgoCD on your cluster (if not done)
3. Apply both ArgoCD Application manifests
4. Perform initial sync for both apps
5. (Optional) Set up Image Updater for automatic tag detection
6. Start deploying!

For Image Updater setup, see [IMAGE-UPDATER-SETUP.md](IMAGE-UPDATER-SETUP.md).

---

**Your ArgoCD automation is ready to go!** 🚀
