# ArgoCD Setup for demo-helm-app

## Repository Information

- **Repository**: https://github.com/dhruv061/helm-argo-demo.git
- **Branch**: master
- **Helm Chart Path**: artha-helm-chart/
- **Application Name**: demo-helm-app

## Quick Start

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

### 3. Deploy the Application

```bash
# Apply the ArgoCD Application
kubectl apply -f argocd/demo-helm-app/argocd-application.yaml

# Verify application is created
kubectl get application -n argocd demo-helm-app
```

### 4. Sync the Application

#### Via ArgoCD UI:
1. Open ArgoCD UI at https://localhost:8080
2. Click on **demo-helm-app** application
3. Review the resources
4. Click **Sync** button
5. Click **Synchronize**

#### Via CLI:
```bash
# Install ArgoCD CLI first (optional)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Login to ArgoCD
argocd login localhost:8080 --username admin --insecure

# Sync the application
argocd app sync demo-helm-app

# Wait for sync to complete
argocd app wait demo-helm-app --timeout 300
```

### 5. Verify Deployment

```bash
# Check application status
kubectl get application -n argocd demo-helm-app

# Check deployed resources
kubectl get all -n default

# Check pods
kubectl get pods -n default

# Check services
kubectl get svc -n default
```

## Configuration Details

### Values Files Used
- `artha-helm-chart/application-values.yaml` - Application configuration (image tags)
- `artha-helm-chart/gateway-values.yaml` - Gateway API configuration

### Deployed Components
Based on your values files, this will deploy:
- **Admin Panel** - 2 replicas with HPA
- **Backend API** - 3 replicas with HPA and PDB
- **Frontend Web** - 3 replicas with HPA and PDB
- Services for all components
- Optional: Gateway API resources (if enabled)

### Sync Policy
- **Manual Sync** - Changes require explicit sync via UI or CLI
- **No auto-sync** - You control when deployments happen

## Enabling ArgoCD Image Updater (Optional)

To automatically detect new image tags from ACR and update values files:

### 1. Install ArgoCD Image Updater

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

### 2. Create ACR Credentials Secret

```bash
kubectl create secret docker-registry acr-credentials \
  --namespace argocd \
  --docker-server=helmtest.azurecr.io \
  --docker-username=<YOUR_ACR_USERNAME> \
  --docker-password=<YOUR_ACR_PASSWORD>
```

### 3. Create Git Credentials Secret

```bash
# For GitHub Personal Access Token
kubectl create secret generic git-creds \
  --namespace argocd \
  --from-literal=username=dhruv061 \
  --from-literal=password=<YOUR_GITHUB_PERSONAL_ACCESS_TOKEN>
```

### 4. Enable Image Updater in ArgoCD Application

Edit `argocd/demo-helm-app/argocd-application.yaml` and **uncomment** the Image Updater annotations (lines 14-41).

### 5. Re-apply the Application

```bash
kubectl apply -f argocd/demo-helm-app/argocd-application.yaml
```

### 6. Verify Image Updater is Working

```bash
# Check Image Updater logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f

# You should see logs like:
# time="..." level=info msg="Checking application demo-helm-app" application=demo-helm-app
```

## Workflow with Image Updater Enabled

1. **Developer pushes code** to app repository
2. **CI/CD builds image** and pushes to ACR: `helmtest.azurecr.io/arthanode:v1.2.0`
3. **Image Updater detects** new tag in ACR (every 2 minutes)
4. **Image Updater updates** `application-values.yaml` with new tag
5. **Image Updater commits** to GitHub: `build: automatic update of arthanode to v1.2.0`
6. **ArgoCD detects** Git change and shows "OutOfSync"
7. **You manually sync** via ArgoCD UI to deploy

## Useful Commands

```bash
# Get application status
argocd app get demo-helm-app

# View application in UI
argocd app get demo-helm-app --show-operation

# Sync application
argocd app sync demo-helm-app

# View sync history
argocd app history demo-helm-app

# Rollback to previous version
argocd app rollback demo-helm-app

# Delete application
argocd app delete demo-helm-app
```

## Troubleshooting

### Application shows "OutOfSync"
This is expected with manual sync. Click "Sync" in UI or run:
```bash
argocd app sync demo-helm-app
```

### Cannot access ArgoCD UI
Make sure port-forward is running:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Image pull errors
Ensure image registry credentials are configured:
```bash
kubectl get secret -n default
# You should see image pull secrets for ACR
```

### Sync fails
Check application details:
```bash
argocd app get demo-helm-app
kubectl get events -n default
```

## Repository Structure

```
helm-argo-demo/
├── artha-helm-chart/              # Helm chart
│   ├── Chart.yaml
│   ├── application-values.yaml    # App config (image tags updated here)
│   ├── gateway-values.yaml        # Gateway config
│   └── templates/                 # Kubernetes manifests
│       ├── applications/
│       └── gateway/
└── argocd/                        # ArgoCD configuration
    └── demo-helm-app/
        └── argocd-application.yaml  # ArgoCD app definition
```

## Next Steps

1. ✅ ArgoCD Application configured and validated
2. Install ArgoCD on your cluster
3. Apply the ArgoCD Application manifest
4. Perform initial sync
5. (Optional) Set up Image Updater for automatic tag detection
6. Start deploying!

For complete documentation, see:
- [ARGOCD-SETUP.md](../ARGOCD-SETUP.md) - Detailed setup guide
- [ARGOCD-COMMANDS.md](../ARGOCD-COMMANDS.md) - Command reference
