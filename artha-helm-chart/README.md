# Artha Helm Chart

Complete Helm chart for deploying the Artha application stack with Gateway API support and application components.

---

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Values Files](#values-files)
- [Installation Methods](#installation-methods)
- [Common Operations](#common-operations)
- [Application Components](#application-components)
- [Gateway API Configuration](#gateway-api-configuration)
- [Domain Flags Explained](#domain-flags-explained)
- [Migration Workflow](#migration-workflow)
- [Helm Commands Reference](#helm-commands-reference)
- [Troubleshooting](#troubleshooting)

---

## Overview

This Helm chart manages:

### Application Components
- **5 Deployments**: admin, backend, frontend, cron-service, email-alert-job
- **3 Services**: admin-service, backend-service, frontend-service
- **3 HPAs**: Horizontal Pod Autoscalers for admin, backend, frontend
- **2 PDBs**: Pod Disruption Budgets for backend, frontend
- **SSL Secrets**: TLS certificate management

### Gateway API Resources
- **Gateway**: NGINX Gateway Fabric with HTTPS listeners
- **HTTPRoutes**: Domain routing (apex, www, redirects)
- **Certificate Management**: cert-manager + Let's Encrypt integration
- **Multi-domain Support**: Hundreds of domains with per-domain SSL configuration

---

## Quick Start

### Prerequisites
- Kubernetes cluster with Gateway API CRDs installed
- NGINX Gateway Fabric controller running
- cert-manager installed (for SSL certificates)
- Helm 3.x

### Install Everything (Applications + Gateway)

```bash
helm install artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

### Install Applications Only (No Gateway)

```bash
helm install artha . -f application-values.yaml
```

### Verify Installation

```bash
# Check all resources
kubectl get all -n default

# Check Gateway resources
kubectl get gateway,httproute -n nginx-gateway

# Check certificates
kubectl get certificates -n nginx-gateway
```

---

## Values Files

This chart uses **two separate values files** for better organization:

### 1. `application-values.yaml`
**Contains**: Application component configurations

- Image registry and tags
- Replica counts
- Resource limits/requests
- HPA settings
- PDB configurations
- Environment variables

**Update when**: Deploying new application versions, scaling, changing resources

### 2. `gateway-values.yaml`
**Contains**: Gateway API and routing configurations

- Gateway settings
- Domain configurations
- TLS/SSL settings
- HTTPRoute rules
- Certificate scopes

**Update when**: Adding domains, changing routing, updating SSL configuration

### Optional: `values.yaml` (Combined)
For backward compatibility, you can use a single combined values file, but split files are recommended.

---

## Installation Methods

### Method 1: Split Values (Recommended)

**Install with both files:**
```bash
helm install artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

**Update only Gateway:**
```bash
helm upgrade artha . \
  -f gateway-values.yaml \
  --reuse-values
```

**Update only Applications:**
```bash
helm upgrade artha . \
  -f application-values.yaml \
  --reuse-values
```

### Method 2: Single Values File

```bash
# Install with combined configuration
helm install artha . -f values.yaml
# or simply
helm install artha .
```

### Method 3: Deploy Applications Only First

Useful when:
- 🔧 Setting up a new cluster
- 🧪 Testing application deployments before exposing them
- 📦 Managing Gateway separately
- ⚙️ Gateway API isn't installed yet

```bash
# Step 1: Deploy applications only
helm install artha . -f application-values.yaml

# Step 2: Later, add Gateway
helm upgrade artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

This deploys:
- ✅ 5 Deployments
- ✅ 3 Services
- ✅ 3 HPAs
- ✅ 2 PDBs
- ❌ No Gateway resources

---

## Common Operations

### Update Application Image Version

1. Edit `application-values.yaml`:
   ```yaml
   applications:
     backend:
       image:
         tag: migrated-v5.0.3  # Update version
   ```

2. Apply changes:
   ```bash
   helm upgrade artha . -f application-values.yaml --reuse-values
   ```

### Add New Client Domain

1. Edit `gateway-values.yaml`:
   ```yaml
   domains:
     - domain: newclient.example.com
       dnsReady: true
       certificateScope: both
       canonicalHost: www
       service: frontend-service
       port: 80
   ```

2. Apply changes:
   ```bash
   helm upgrade artha . -f gateway-values.yaml --reuse-values
   ```

### Scale Application

1. Edit `application-values.yaml`:
   ```yaml
   applications:
     backend:
       replicaCount: 5
       hpa:
         maxReplicas: 10
   ```

2. Apply changes:
   ```bash
   helm upgrade artha . -f application-values.yaml --reuse-values
   ```

### Update Both Applications and Gateway

```bash
helm upgrade artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

---

## Application Components

### Deployments Configuration

Each application component can be individually enabled/disabled:

```yaml
applications:
  imageRegistry: arthajobboard.azurecr.io
  namespace: default
  
  backend:
    enabled: true
    replicaCount: 3
    image:
      repository: arthanode
      tag: migrated-v5.0.2
    resources:
      limits:
        cpu: 1700m
        memory: 4Gi
      requests:
        cpu: 1200m
        memory: 3584Mi
    hpa:
      enabled: true
      minReplicas: 3
      maxReplicas: 4
```

### Horizontal Pod Autoscalers

- **admin-hpa**: CPU-based (70% target)
- **backend-hpa**: CPU & memory with custom scaling behavior
- **frontend-hpa**: CPU & memory with custom scaling behavior

### Pod Disruption Budgets

- **backend-pdb**: Ensures minimum 2 pods available
- **frontend-pdb**: Ensures minimum 2 pods available

---

## Gateway API Configuration

### Gateway Settings

```yaml
gateway:
  enabled: true
  name: nginx-gateway
  namespace: nginx-gateway
  className: nginx
  issuer: letsencrypt-prod
```

### Domain Configuration Example

```yaml
domains:
  - domain: example.com
    service: frontend-service
    port: 80
    dnsReady: true
    certificateScope: non-www
    canonicalHost: www
```

---

## Domain Flags Explained

Each domain has **three explicit flags** that represent real-world state:

### 1. `dnsReady` (true | false)

**Question**: Has DNS been pointed to the Gateway LoadBalancer IP?

| Value | Meaning | Result |
|-------|---------|--------|
| `false` | DNS still points to old Ingress | Nothing created (safe) |
| `true` | DNS points to Gateway | Listeners + routes created |

**Why**: Prevents premature Let's Encrypt validation and ACME failures

### 2. `certificateScope` (both | www | non-www)

**Question**: Which hostnames should get Let's Encrypt certificates?

| Value | Certificates Issued For |
|-------|------------------------|
| `both` | `domain.com` and `www.domain.com` |
| `www` | `www.domain.com` only |
| `non-www` | `domain.com` only |

**Example**: Use `non-www` when Cloudflare handles `www` SSL

### 3. `canonicalHost` (www | apex)

**Question**: Which hostname should serve the application?

| Value | Behavior |
|-------|----------|
| `www` | Serve `www.domain.com`, redirect `domain.com → www` |
| `apex` | Serve `domain.com` only, no redirect |

---

## Common Domain Scenarios

### Scenario 1: Cloudflare handles www, Gateway handles apex

```yaml
domain: example.com
dnsReady: true
certificateScope: non-www  # Only apex cert
canonicalHost: www         # Redirect apex → www
```

**Result**:
- cert-manager issues cert only for `example.com`
- `example.com` redirects to `www.example.com`
- Cloudflare terminates SSL for www

### Scenario 2: Gateway handles both www and apex

```yaml
domain: example.com
dnsReady: true
certificateScope: both     # Both certs
canonicalHost: www         # Redirect apex → www
```

**Result**:
- cert-manager issues 2 certificates
- Gateway terminates TLS for both
- Apex redirects to www

### Scenario 3: Serve only apex (no www)

```yaml
domain: example.com
dnsReady: true
certificateScope: non-www
canonicalHost: apex
```

**Result**:
- Only `example.com` works
- No www route created
- No redirect

### Scenario 4: Client not migrated yet (SAFE)

```yaml
domain: example.com
dnsReady: false
```

**Result**:
- No listeners created
- No cert issuance attempted
- Old infrastructure continues serving

---

## Migration Workflow

For gradual migration from NGINX Ingress to Gateway API:

1. **Client updates DNS** → Point to Gateway LoadBalancer IP
2. **Verify DNS**: `dig domain.com` (should show Gateway IP)
3. **Set flags** in `gateway-values.yaml`:
   - `dnsReady: true`
   - Choose `certificateScope`
   - Choose `canonicalHost`
4. **Deploy**: `helm upgrade artha . -f gateway-values.yaml --reuse-values`
5. **Verify**: Check certificate issuance and routing

Repeat per client for safe, incremental migration.

---

## Helm Commands Reference

### Installation

```bash
# First-time install
helm install artha .
```

### Upgrade

```bash
# Standard upgrade (most common)
helm upgrade --install artha .

# With specific values files
helm upgrade artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

### Validation

```bash
# Lint the chart
helm lint . -f gateway-values.yaml -f application-values.yaml

# Dry run before deployment
helm upgrade artha . \
  -f gateway-values.yaml \
  -f application-values.yaml \
  --dry-run

# View generated YAML
helm template artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

### Management

```bash
# View deployment history
helm history artha

# Rollback to previous version
helm rollback artha <REVISION>

# Uninstall (WARNING: removes all resources)
helm uninstall artha
```

### Tips

- **Always use `--dry-run`** before risky changes
- **Prefer `helm upgrade`** over delete/recreate
- **Use `helm template`** in code review to validate
- **Never run `helm uninstall`** during active migration

---

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n default

# View pod logs
kubectl logs <pod-name> -n default

# Describe pod for events
kubectl describe pod <pod-name> -n default
```

### Certificate Not Issuing

```bash
# Check certificate status
kubectl get certificates -n nginx-gateway

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Verify DNS is pointing to Gateway
dig <domain>
```

### HTTPRoute Not Working

```bash
# Check HTTPRoute status
kubectl get httproute -n default -o yaml

# Check Gateway status
kubectl get gateway -n nginx-gateway -o yaml

# Verify ReferenceGrant exists
kubectl get referencegrant -n default
```

### Gateway Shows OutOfSync in ArgoCD

This is expected due to controller field normalization. Ignore directives are configured in ArgoCD application manifests.

### Image Pull Errors

```bash
# Check imagePullSecrets exist
kubectl get secrets -n default

# Verify ACR credentials are correct
kubectl get secret <secret-name> -n default -o yaml
```

---

## Folder Structure

```
artha-helm-chart/
├── Chart.yaml                    # Chart metadata
├── README.md                     # This file
├── application-values.yaml       # Application configuration
├── gateway-values.yaml          # Gateway configuration
└── templates/
    ├── applications/            # Application resources
    │   ├── deployments/        # 5 deployments
    │   ├── services/           # 3 services
    │   ├── hpa/                # 3 HPAs
    │   ├── pdb/                # 2 PDBs
    │   └── secrets/            # SSL secrets
    │
    └── gateway/                 # Gateway resources
        ├── gateway.yaml         # Gateway + listeners
        ├── httproute-apex.yaml  # Apex domain routes
        ├── httproute-www.yaml   # WWW domain routes
        ├── httproute-redirect.yaml # Redirects
        ├── reference-grant.yaml # Cross-namespace access
        └── artha-httproutes/    # Primary routes
            ├── api-httproute.yaml
            ├── app-httproute.yaml
            └── wildcard-httproute.yaml
```

---

## Golden Rules

- ✅ Always change behavior via values files, never edit templates
- ✅ Never set `dnsReady: true` before DNS cutover
- ✅ Use `helm template` to verify changes before applying
- ✅ Treat Gateway as generated output, not infrastructure code
- ✅ Keep `values.yaml` as your audit log

---

## Summary

This Helm chart provides:

- ✅ Complete application stack management
- ✅ Safe, incremental Gateway migration
- ✅ Explicit SSL and redirect control
- ✅ Git-reviewed, auditable operations
- ✅ Predictable Helm workflows
- ✅ Flexible deployment options (apps only, gateway only, or both)

For ArgoCD integration and automated deployments, see the [ArgoCD documentation](../argocd/README.md).

---

**Ready to deploy!** 🚀
