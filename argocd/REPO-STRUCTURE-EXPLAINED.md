# Repository Setup - Simple Answer

## Your Question

**Do Helm charts and ArgoCD configs need to be in the same repository?**

## Answer: They CAN be in the same repo (and usually are)

## Your Setup (Recommended)

### Repository 1: Application Code (Your existing app repo)
- **Location**: Separate repository (e.g., `demo-app-code`)
- **Contains**: 
  - Application source code
  - Dockerfile
  - CI/CD pipeline (GitLab CI, GitHub Actions, etc.)
- **Purpose**: Build and push Docker images to ACR

### Repository 2: Helm + ArgoCD (THIS REPO)
- **Location**: `ingress-to-gateway-api` (this current repository)
- **Contains**:
  - ✅ Helm charts (`artha-helm-chart/`)
  - ✅ ArgoCD configs (`argocd/`)
  - ✅ Values files
- **Purpose**: GitOps - Kubernetes deployment manifests

## Visual Representation

```
┌─────────────────────────────────┐
│  Repo 1: demo-app-code         │
│                                 │
│  - src/                         │
│  - Dockerfile                   │
│  - .gitlab-ci.yml              │
│  - package.json                 │
│                                 │
│  Purpose: Build Docker images   │
└─────────────────────────────────┘
            │
            ├─ Builds image
            ├─ Tags: v1.2.0
            └─ Pushes to ACR
                    ↓
        ┌──────────────────────────┐
        │  ACR (helmtest.azurecr.io)│
        │  - arthanode:v1.2.0      │
        └──────────────────────────┘
                    ↓
            Image Updater watches
                    ↓
┌─────────────────────────────────┐
│  Repo 2: ingress-to-gateway-api │
│  (THIS REPO)                    │
│                                 │
│  - artha-helm-chart/           │
│    - application-values.yaml   │  ← Image tags updated here
│    - gateway-values.yaml       │
│    - templates/                │
│  - argocd/                     │
│    - demo-helm-app/            │
│      - argocd-application.yaml │
│                                 │
│  Purpose: GitOps deployment     │
└─────────────────────────────────┘
            ↓
        ArgoCD monitors
            ↓
        Shows "OutOfSync"
            ↓
        You manually sync
            ↓
    ┌──────────────────┐
    │  Kubernetes      │
    │  Deployed Pods   │
    └──────────────────┘
```

## How ArgoCD Updates Image Tags via Helm

### The Process

1. **CI/CD pushes image to ACR**:
   ```bash
   docker push helmtest.azurecr.io/arthanode:v1.2.0
   ```

2. **ArgoCD Image Updater polls ACR**:
   - Checks for new tags every 2 minutes
   - Finds new tag: `v1.2.0`

3. **Image Updater updates your values file**:
   - Opens `artha-helm-chart/application-values.yaml`
   - Changes:
   ```yaml
   applications:
     backend:
       image:
         tag: v1.0.0  # OLD
   ```
   To:
   ```yaml
   applications:
     backend:
       image:
         tag: v1.2.0  # NEW - Updated automatically!
   ```

4. **Image Updater commits to Git**:
   ```bash
   git commit -m "build: automatic update of arthanode to v1.2.0"
   git push
   ```

5. **ArgoCD detects Git change**:
   - Monitors this repository
   - Sees commit to `application-values.yaml`
   - Shows "OutOfSync" status

6. **You sync manually via ArgoCD UI**

## Your Configuration (2 Values Files Only)

Since you have only 2 values files, the image tags will be updated in `application-values.yaml`:

```yaml
# argocd/demo-helm-app/argocd-application.yaml
annotations:
  # Tell Image Updater to update application-values.yaml
  argocd-image-updater.argoproj.io/write-back-target: helmvalues:artha-helm-chart/application-values.yaml
  
spec:
  source:
    repoURL: https://github.com/YOUR-ORG/ingress-to-gateway-api.git
    path: artha-helm-chart
    helm:
      valueFiles:
        - application-values.yaml  # ← Image tags updated here
        - gateway-values.yaml
```

## Summary

### What You Have

✅ **App code repo** → Separate (your existing repo)  
✅ **Helm + ArgoCD repo** → THIS repo (`ingress-to-gateway-api`)

### This is PERFECT and recommended!

**You do NOT need to**:
- ❌ Put helm charts in your app code repo
- ❌ Create a 3rd repository
- ❌ Create production-values.yaml (can use your existing 2 files)

### Image Update Flow

1. **App repo** → Build → Push to ACR
2. **Image Updater** → Watch ACR → Update `application-values.yaml` in THIS repo
3. **ArgoCD** → Detect change → Show "OutOfSync"
4. **You** → Click Sync in UI

This is the **standard GitOps pattern** and exactly what you should do!
