# ArgoCD Applications Overview

## Two Separate Applications

You now have two independent ArgoCD applications for better separation of concerns:

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

## Deployment Steps

### Step 1: Delete Old Application

```bash
# Delete the old demo-helm-app
kubectl delete application demo-helm-app -n argocd

# This will NOT delete your running resources (they'll become unmanaged temporarily)
```

### Step 2: Clean Up Existing Resources (Important!)

Since resources were created by the old app, we need to clean them up:

```bash
# Delete existing PDBs (causing the error)
kubectl delete pdb backend-pdb frontend-pdb -n default

# Delete existing deployments
kubectl delete deployment admin backend frontend -n default

# Delete existing HPAs
kubectl delete hpa admin-hpa backend-hpa frontend-hpa -n default

# Services can stay (or delete if you want)
# kubectl delete svc admin-service backend-service frontend-service -n default
```

### Step 3: Deploy Applications

```bash
# Deploy application components
kubectl apply -f argocd/demo-artha-app/argocd-application.yaml

# Deploy gateway resources
kubectl apply -f argocd/demo-gateway-app/argocd-application.yaml

# Verify both are created
kubectl get applications -n argocd
```

### Step 4: Sync in ArgoCD UI

1. Open ArgoCD UI: `kubectl port-forward svc/argocd-server -n argocd 8080:443`
2. Login at https://localhost:8080
3. You'll see two applications:
   - **demo-artha-app** (OutOfSync)
   - **demo-gateway-app** (OutOfSync)
4. Click on **demo-artha-app** → **Sync** → **Synchronize**
5. Click on **demo-gateway-app** → **Sync** → **Synchronize**

## Benefits of This Setup

### ✅ Separation of Concerns
- Application deployments managed separately from gateway config
- Changes to routing don't affect app deployments
- Can update gateway without redeploying apps

### ✅ Independent Lifecycle
- Deploy new app versions without touching gateway
- Update gateway routes without restarting apps
- Different sync policies if needed

### ✅ Clear Ownership
- **demo-artha-app**: Owns deployments, services, HPAs, PDBs
- **demo-gateway-app**: Owns gateway, HTTPRoutes, certificates

### ✅ Image Updater
- Only runs on demo-artha-app (where images exist)
- Gateway app doesn't need image tracking

## How It Works

### Application Updates
1. CI/CD pushes new image to ACR: `helmtest.azurecr.io/arthanode:v1.2.0`
2. Image Updater detects new tag
3. Updates `application-values.yaml` in Git
4. **demo-artha-app** shows "OutOfSync"
5. You sync → New version deploys

### Gateway Updates
1. You edit `gateway-values.yaml` (add new domain, change routes, etc.)
2. Commit and push to GitHub
3. **demo-gateway-app** shows "OutOfSync"
4. You sync → Gateway routes update

## Directory Structure

```
argocd/
├── demo-artha-app/
│   └── argocd-application.yaml     # App deployments
└── demo-gateway-app/
    └── argocd-application.yaml     # Gateway resources
```

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

## Troubleshooting

### Resources Already Exist Error
This happens when resources are managed by old app. Solution:
```bash
# Delete the conflicting resources
kubectl delete pdb,hpa,deployment -l app in (admin,backend,frontend) -n default
# Then sync the new application
```

### Both Apps Show Same Resources
Check the exclude values - each app should have different resources:
- **demo-artha-app**: gateway.enabled=false
- **demo-gateway-app**: applications.*.enabled=false

### Image Updater Not Working
Check:
```bash
# View Image Updater logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f

# Verify ACR credentials secret exists
kubectl get secret acr-credentials -n argocd
```

## Next Steps

1. ✅ Clean up old resources
2. ✅ Apply both ArgoCD applications
3. ✅ Sync both apps in ArgoCD UI
4. ✅ Verify deployments are running
5. ✅ Test image updater (push new image to ACR)
6. ✅ Test gateway updates (modify gateway-values.yaml)
