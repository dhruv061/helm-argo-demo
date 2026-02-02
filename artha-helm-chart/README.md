# Gateway Migration Helm Chart

This Helm chart is designed to safely migrate **hundreds of domains** from **NGINX Ingress** to **NGINX Gateway Fabric**, while handling:

- Client-controlled DNS cutover
- cert-manager + Let’s Encrypt
- Cloudflare-managed `www` SSL (optional)
- Per-domain SSL and redirect behavior
- Zero-downtime, gradual migration

This README explains **folder structure**, **all flags**, and **when to use what**.

---

## Folder Structure

```
artha-helm-chart/
├── Chart.yaml
├── values.yaml                  # Combined values (optional - for single-file usage)
├── gateway-values.yaml         # Gateway API configuration only
├── application-values.yaml     # Application components configuration only
├── README.md
└── templates/
    ├── gateway/             # Gateway API resources
    │   ├── gateway.yaml    # Gateway + HTTPS listeners (TLS control)
    │   ├── httproute-apex.yaml  # Serve apex domains (domain.com)
    │   ├── httproute-www.yaml   # Serve www domains (www.domain.com)
    │   ├── httproute-redirect.yaml # Redirect apex → www
    │   ├── reference-grant.yaml # Cross-namespace access
    │   └── artha-httproutes/    # Primary domain routes
    │       ├── api-httproute.yaml
    │       ├── app-httproute.yaml
    │       └── wildcard-httproute.yaml
    │
    └── applications/        # Application components
        ├── deployments/    # 5 deployments
        │   ├── admin.yaml
        │   ├── backend.yaml
        │   ├── cron-service.yaml
        │   ├── email-alert-job.yaml
        │   └── frontend.yaml
        ├── services/      # 3 services
        │   ├── admin-service.yaml
        │   ├── backend-service.yaml
        │   └── frontend-service.yaml
        ├── hpa/           # 3 horizontal pod autoscalers
        │   ├── admin-hpa.yaml
        │   ├── backend-hpa.yaml
        │   └── frontend-hpa.yaml
        ├── pdb/           # 2 pod disruption budgets
        │   ├── backend-pdb.yaml
        │   └── frontend-pdb.yaml
        └── secrets/       # SSL secret reference
            └── ssl-secret.yaml
```


---

## High-Level Architecture

**Gateway**
- Owns LoadBalancer
- Owns TLS termination
- Triggers cert-manager via `hostname`

**HTTPRoute**
- Owns routing logic
- Decides serve vs redirect
- One route per hostname behavior

**Helm**
- Generates everything from a single domain inventory
- Prevents human error

---

## values.yaml – Single Source of Truth

This file describes **what is true in the real world**.

```yaml
gateway:
  name: nginx-gateway
  namespace: nginx-gateway
  className: nginx
  issuer: letsencrypt-prod

routesNamespace: default

domains:
  - domain: example.com
    service: frontend-service
    port: 80

    dnsReady: true
    certificateScope: non-www
    canonicalHost: www
```

You should **never edit templates** during normal operations.

---

## Using Split Values Files

This chart supports two approaches for managing configurations:

### Option 1: Single Values File (Traditional)

Use the combined `values.yaml` containing both gateway and application configurations:

```bash
helm install artha .
# or
helm upgrade artha .
```

### Option 2: Split Values Files (Recommended)

Use separate files for better organization and independent updates:

- **`gateway-values.yaml`** - Gateway API, domains, TLS, HTTPRoute configurations
- **`application-values.yaml`** - Application components (deployments, services, HPAs, PDBs)

**Install with both files:**
```bash
helm install artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

**Update only Gateway configuration:**
```bash
helm upgrade artha . \
  -f gateway-values.yaml \
  --reuse-values
```

**Update only Application configuration:**
```bash
helm upgrade artha . \
  -f application-values.yaml \
  --reuse-values
```

**Update application image version:**
```bash
# Edit application-values.yaml, change image tag
helm upgrade artha . -f application-values.yaml --reuse-values
```

**Benefits of split files:**
- 🎯 Update gateway config without touching application config
- 🔄 Update application versions independently
- 👥 Different teams can manage different files
- 📝 Cleaner version control and change tracking

---


## Application Components Configuration

In addition to Gateway API resources, this Helm chart manages your complete application stack. All application components are configured in the `applications` section of `values.yaml`.

### Components Included

1. **Deployments** (5 total):
   - `admin` - Admin panel application
   - `backend` - Backend API service
   - `frontend` - Frontend web application
   - `cronService` - Cron job service
   - `emailAlertJob` - Email notification service

2. **Services** (3 total):
   - `admin-service` - Exposes admin deployment
   - `backend-service` - Exposes backend deployment
   - `frontend-service` - Exposes frontend deployment

3. **Horizontal Pod Autoscalers** (3 total):
   - `admin-hpa` - CPU-based scaling for admin
   - `backend-hpa` - CPU & memory scaling with custom behavior
   - `frontend-hpa` - CPU & memory scaling with custom behavior

4. **Pod Disruption Budgets** (2 total):
   - `backend-pdb` - Ensures minimum availability for backend
   - `frontend-pdb` - Ensures minimum availability for frontend

### Example Application Configuration

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

### Enabling/Disabling Components

Each application component can be individually enabled or disabled:

```yaml
applications:
  admin:
    enabled: true    # Set to false to disable
  backend:
    enabled: true
  frontend:
    enabled: true
  cronService:
    enabled: true
  emailAlertJob:
    enabled: true
```

### Updating Image Tags

To update an application version, modify the `tag` value:

```yaml
applications:
  backend:
    image:
      tag: migrated-v5.0.3  # Update version here
```

Then run `helm upgrade` to deploy the new version.

---


## Domain Flags Explained

Each domain has **three explicit flags**. They represent **real-world state**, not Kubernetes internals.

---

### 1. `dnsReady`

```yaml
dnsReady: true | false
```

**Question it answers:**
> Has the client already pointed DNS to the Gateway LoadBalancer IP?

| Value | Meaning | Result |
|------|--------|--------|
| `false` | DNS still points to old Ingress | Nothing is created (safe) |
| `true` | DNS points to Gateway | Listeners + routes allowed |

Why this exists:
- Prevents premature Let’s Encrypt validation
- Avoids ACME failures and rate limits
- Enables gradual client-by-client migration

---

### 2. `certificateScope`

```yaml
certificateScope: both | www | non-www
```

**Question it answers:**
> Which hostnames should get Let’s Encrypt certificates from cert-manager?

| Value | Certificates Issued For |
|-----|-------------------------|
| `both` | `domain.com` and `www.domain.com` |
| `www` | `www.domain.com` only |
| `non-www` | `domain.com` only |

Notes:
- Used only when `dnsReady: true`
- Independent of redirect behavior
- Required because NGINX Gateway Fabric issues certs via `hostname`

---

### 3. `canonicalHost`

```yaml
canonicalHost: www | apex
```

**Question it answers:**
> Which hostname should actually serve the application?

| Value | Behavior |
|------|----------|
| `www` | Serve `www.domain.com`, redirect `domain.com → www` |
| `apex` | Serve `domain.com` only, no redirect |

This flag controls **routing only**, not TLS.

---

## How Templates Use These Flags

### gateway.yaml

- Creates **HTTPS listeners only when `dnsReady=true`**
- Creates listeners only for hostnames allowed by `certificateScope`
- Triggers cert-manager automatically

### httproute-apex.yaml

- Created only when:
  - `dnsReady=true`
  - `canonicalHost=apex`

### httproute-www.yaml

- Created only when:
  - `dnsReady=true`
  - `canonicalHost=www`

### httproute-redirect.yaml

- Created only when:
  - `dnsReady=true`
  - `canonicalHost=www`

---

## Common Scenarios (When to Use What)

### 1. Cloudflare handles `www`, redirect apex → www (MOST COMMON)

```yaml
dnsReady: true
certificateScope: non-www
canonicalHost: www
```

Result:
- cert-manager issues cert only for apex
- `domain.com → www.domain.com`
- Cloudflare terminates SSL for www

---

### 2. Kubernetes handles SSL for both www and apex

```yaml
dnsReady: true
certificateScope: both
canonicalHost: www
```

Result:
- cert-manager issues 2 certs
- Gateway terminates TLS for both
- Apex redirects to www

---

### 3. Serve only apex domain (no www)

```yaml
dnsReady: true
certificateScope: non-www
canonicalHost: apex
```

Result:
- Only `domain.com` works
- No redirect
- No www route created

---

### 4. Client not migrated yet (SAFE STATE)

```yaml
dnsReady: false
```

Result:
- No listeners
- No cert issuance
- Old Ingress continues to serve traffic

---

## Migration Workflow (Recommended)

For each client:

1. Client updates DNS → Gateway LB IP
2. You verify DNS (`dig domain.com`)
3. Set `dnsReady: true`
4. Choose `certificateScope`
5. Choose `canonicalHost`
6. Merge to GitLab → Helm deploys

Repeat per client.

---

## Why This Design Works

- Explicit intent (no magic)
- Safe with external DNS ownership
- Prevents SSL outages
- Git-reviewed, auditable changes
- Scales to 200+ domains

---

## Golden Rules

- Never set `dnsReady: true` before DNS cutover
- Never edit Gateway YAML manually
- Always change behavior via `values.yaml`
- Treat Gateway as **generated output**, not infra code

---

## Useful Helm Commands

This section lists the **day-to-day Helm commands** you will actually use with this chart.

---

### Install (first-time deployment)

```bash
helm install artha-helm-chart ./artha-helm-chart
```

Use this when:
- Deploying Gateway for the first time
- No existing Helm release exists

---

### Upgrade (most common operation)

```bash
helm upgrade --install artha-helm-chart ./artha-helm-chart/
```

Use this when:
- Adding a new domain
- Flipping `dnsReady` to `true`
- Changing `certificateScope`
- Changing `canonicalHost`

This is the **normal production workflow**.

---

### Dry run (ALWAYS use before merge if unsure)

```bash
helm upgrade artha-helm-chart ./artha-helm-chart \
  --dry-run
```

Use this when:
- Reviewing a risky change
- Verifying what Kubernetes objects will be created
- Debugging template logic

---

### Render manifests locally (debugging)

```bash
helm template artha-helm-chart ./artha-helm-chart
```

Use this when:
- You want to see the **exact YAML** Helm will generate
- Debugging conditions (`dnsReady`, `certificateScope`, `canonicalHost`)
- Reviewing output in code review

---

### Rollback (emergency recovery)

```bash
helm rollback artha-helm-chart <REVISION>
```

Find available revisions:

```bash
helm history artha-helm-chart
```

Use this when:
- A bad `dnsReady=true` was merged
- A certificate was triggered too early
- You need instant recovery

---

### Uninstall (DO NOT use during migration)

```bash
helm uninstall artha-helm-chart
```

⚠️ **Warning**:
- Removes Gateway and all routes
- Drops traffic immediately
- Only use when fully decommissioning

---

## Operational Tips

- **Never** run `helm uninstall` during active migration
- Prefer `helm upgrade` over delete/recreate
- Use `helm template` in code review to validate behavior
- Treat `values.yaml` as your audit log

---

## Summary

This Helm chart provides:
- Safe, incremental migration
- Explicit SSL and redirect control
- Git-reviewed, auditable operations
- Predictable Helm workflows

If something changes in production, the answer is always in:

```yaml
values.yaml
```
