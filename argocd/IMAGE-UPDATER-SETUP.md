# ArgoCD Image Updater Setup

## Overview

**ArgoCD Image Updater** automatically monitors your Azure Container Registry (ACR) for new image tags and updates your Helm values files in Git. When a new image is pushed to ACR, Image Updater will:

1. Detect the new image tag in ACR
2. Update the image tag in your Git repository (`production-values.yaml`)
3. Commit the change to Git
4. ArgoCD detects the Git change and shows "OutOfSync"
5. You manually sync via ArgoCD UI to deploy

## Installation

### Step 1: Install ArgoCD Image Updater

```bash
# Install via kubectl
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# Verify installation
kubectl get pods -n argocd | grep image-updater
```

### Step 2: Configure ACR Credentials

Create a secret with your ACR credentials:

```bash
# Create ACR credentials secret
kubectl create secret docker-registry acr-credentials \
  --namespace argocd \
  --docker-server=helmtest.azurecr.io \
  --docker-username=<ACR_USERNAME> \
  --docker-password=<ACR_PASSWORD>
```

### Step 3: Configure Git Write Access

ArgoCD Image Updater needs to commit changes to your Git repository. Create a secret with Git credentials:

#### For HTTPS (GitHub/GitLab with Personal Access Token):

```bash
# Create Git credentials secret
kubectl create secret generic git-creds \
  --namespace argocd \
  --from-literal=username=<YOUR_GIT_USERNAME> \
  --from-literal=password=<YOUR_PERSONAL_ACCESS_TOKEN>
```

#### For SSH:

```bash
# Create Git SSH key secret
kubectl create secret generic git-creds \
  --namespace argocd \
  --from-file=sshPrivateKey=/path/to/private/key
```

### Step 4: Update ArgoCD Application with Image Updater Annotations

Update your ArgoCD Application manifest to enable image tracking.

**File**: `argocd/demo-helm-app/argocd-application.yaml`

Add these annotations in the `metadata` section:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-helm-app
  namespace: argocd
  annotations:
    # Enable image updater
    argocd-image-updater.argoproj.io/image-list: |
      backend=helmtest.azurecr.io/arthanode,
      frontend=helmtest.azurecr.io/arthaweb,
      admin=helmtest.azurecr.io/arthaadmin
    
    # Update strategy (semver, latest-tag, or digest)
    argocd-image-updater.argoproj.io/backend.update-strategy: latest
    argocd-image-updater.argoproj.io/frontend.update-strategy: latest
    argocd-image-updater.argoproj.io/admin.update-strategy: latest
    
    # Or use semantic versioning pattern
    # argocd-image-updater.argoproj.io/backend.update-strategy: semver
    # argocd-image-updater.argoproj.io/backend.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    
    # Helm value paths to update
    argocd-image-updater.argoproj.io/backend.helm.image-name: applications.backend.image.repository
    argocd-image-updater.argoproj.io/backend.helm.image-tag: applications.backend.image.tag
    argocd-image-updater.argoproj.io/frontend.helm.image-name: applications.frontend.image.repository
    argocd-image-updater.argoproj.io/frontend.helm.image-tag: applications.frontend.image.tag
    argocd-image-updater.argoproj.io/admin.helm.image-name: applications.admin.image.repository
    argocd-image-updater.argoproj.io/admin.helm.image-tag: applications.admin.image.tag
    
    # Write back method (git or argocd)
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
    
    # ACR credentials reference
    argocd-image-updater.argoproj.io/backend.pull-secret: secret:argocd/acr-credentials
    argocd-image-updater.argoproj.io/frontend.pull-secret: secret:argocd/acr-credentials
    argocd-image-updater.argoproj.io/admin.pull-secret: secret:argocd/acr-credentials
```

## Update Strategy Options

### Option 1: Latest Tag (Simple)

Tracks the latest image tag pushed to ACR:

```yaml
argocd-image-updater.argoproj.io/backend.update-strategy: latest
```

**Use case**: Development/staging environments

### Option 2: Semantic Versioning (Recommended for Production)

Tracks semantic version tags (v1.0.0, v1.2.3, etc.):

```yaml
argocd-image-updater.argoproj.io/backend.update-strategy: semver
argocd-image-updater.argoproj.io/backend.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
```

**Use case**: Production environments with versioned releases

### Option 3: Specific Tag Pattern

Track tags matching a specific pattern:

```yaml
argocd-image-updater.argoproj.io/backend.update-strategy: latest
argocd-image-updater.argoproj.io/backend.allow-tags: regexp:^prod-.*
```

**Use case**: Only deploy tags starting with "prod-"

## How It Works

### Workflow

1. **Developer pushes code** to app repository
   ```bash
   git push origin main
   ```

2. **CI/CD pipeline builds and pushes image** to ACR
   ```bash
   docker build -t helmtest.azurecr.io/arthanode:v1.2.0 .
   docker push helmtest.azurecr.io/arthanode:v1.2.0
   ```

3. **ArgoCD Image Updater detects new tag** (runs every 2 minutes by default)
   - Polls ACR for new images
   - Finds new tag: `v1.2.0`

4. **Image Updater updates Git repository**
   - Updates `production-values.yaml` with new tag
   - Commits: `build: automatic update of arthanode to v1.2.0`
   - Pushes to Git

5. **ArgoCD detects Git change**
   - Shows application as "OutOfSync"
   - **You manually sync via UI** to deploy

6. **Deployment happens**
   - New version rolls out to Kubernetes

## Configuration for Your Setup

Since you have `application-values.yaml` and `gateway-values.yaml`, and you want image updates:

### Option A: Update Existing Values Files

Image Updater can update your existing `application-values.yaml`:

```yaml
annotations:
  argocd-image-updater.argoproj.io/write-back-method: git
  argocd-image-updater.argoproj.io/write-back-target: helmvalues:application-values.yaml
```

**Note**: This will directly modify your base values file.

### Option B: Use Separate production-values.yaml (Recommended)

Keep your base values unchanged and use `production-values.yaml` for image tag overrides:

```yaml
annotations:
  argocd-image-updater.argoproj.io/write-back-method: git
  argocd-image-updater.argoproj.io/write-back-target: helmvalues:argocd/demo-helm-app/production-values.yaml
```

**Recommended**: This keeps your base configuration stable and only updates production tags.

## Complete ArgoCD Application Example

Here's a complete example for your setup:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-helm-app
  namespace: argocd
  annotations:
    # Track backend image
    argocd-image-updater.argoproj.io/image-list: backend=helmtest.azurecr.io/arthanode
    argocd-image-updater.argoproj.io/backend.update-strategy: semver
    argocd-image-updater.argoproj.io/backend.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/backend.helm.image-tag: applications.backend.image.tag
    argocd-image-updater.argoproj.io/backend.pull-secret: secret:argocd/acr-credentials
    
    # Write back to Git
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: helmvalues:argocd/demo-helm-app/production-values.yaml
    argocd-image-updater.argoproj.io/git-branch: main
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-ORG/ingress-to-gatway-api.git
    targetRevision: main
    path: artha-helm-chart
    helm:
      valueFiles:
        - application-values.yaml
        - gateway-values.yaml
        - ../argocd/demo-helm-app/production-values.yaml
      releaseName: demo-helm-app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

## Verification

### Check Image Updater Logs

```bash
# View Image Updater logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f

# Check if images are being tracked
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater | grep demo-helm-app
```

### Test the Workflow

1. **Push a new image to ACR**:
   ```bash
   docker tag myapp:latest helmtest.azurecr.io/arthanode:v1.2.0
   docker push helmtest.azurecr.io/arthanode:v1.2.0
   ```

2. **Wait for Image Updater** (max 2 minutes)

3. **Check Git repository**:
   - You should see a new commit updating the image tag

4. **Check ArgoCD UI**:
   - Application should show "OutOfSync"

5. **Manually sync** via UI or CLI

## Alternative: Manual Tag Update (Without Image Updater)

If you don't want to use Image Updater, you can manually update tags:

### Workflow

1. **CI/CD pushes image to ACR**
   ```bash
   docker push helmtest.azurecr.io/arthanode:v1.2.0
   ```

2. **Manually update production-values.yaml**:
   ```bash
   nano argocd/demo-helm-app/production-values.yaml
   # Change tag to v1.2.0
   ```

3. **Commit and push**:
   ```bash
   git add argocd/demo-helm-app/production-values.yaml
   git commit -m "Deploy backend v1.2.0"
   git push origin main
   ```

4. **ArgoCD shows OutOfSync**

5. **Manually sync via ArgoCD UI**

## Troubleshooting

### Image Updater Not Detecting Images

Check credentials and logs:
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater --tail=100
```

### Git Commit Failures

Verify Git credentials:
```bash
kubectl get secret git-creds -n argocd -o yaml
```

### Images Not Tracked

Verify annotations:
```bash
kubectl get application demo-helm-app -n argocd -o yaml | grep annotations -A 20
```

## Recommendation for Your Use Case

Based on your description, I recommend:

1. **Use ArgoCD Image Updater** to automatically detect new tags from ACR
2. **Use production-values.yaml** for image tag overrides (already created)
3. **Keep manual sync** in ArgoCD (as configured)
4. **Use semver strategy** if your images use version tags, or `latest` for development

This gives you:
- ✅ Automatic detection of new images from ACR
- ✅ Automatic Git commit with new tag
- ✅ ArgoCD shows "OutOfSync"
- ✅ You manually sync via UI when ready
- ✅ Full GitOps workflow with audit trail
