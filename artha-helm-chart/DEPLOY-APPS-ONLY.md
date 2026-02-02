# Deploying Applications Only (Without Gateway)

If you want to deploy only the application components first (deployments, services, HPAs, PDBs) without the Gateway API resources, you can do so using only `application-values.yaml`.

## Steps

### 1. Install Applications Only

```bash
helm upgrade --install artha . -f application-values.yaml
```

This will deploy:
- ✅ 5 Deployments (admin, backend, frontend, cron-service, email-alert-job)
- ✅ 3 Services (admin-service, backend-service, frontend-service)
- ✅ 3 HPAs (admin, backend, frontend)
- ✅ 2 PDBs (backend, frontend)
- ❌ No Gateway API resources

### 2. Later, Add Gateway Configuration

When you're ready to add the Gateway API and routing:

```bash
helm upgrade artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

This will add:
- ✅ 1 Gateway
- ✅ 5 HTTPRoutes (api, app, wildcard, client domains)
- ✅ 1 ReferenceGrant (if needed)

## Use Cases

**Deploy applications first when:**
- 🔧 Setting up a new cluster
- 🧪 Testing application deployments before exposing them
- 📦 You want to manage Gateway separately (different team/responsibility)
- ⚙️ Gateway API isn't installed yet in your cluster

**Deploy both together when:**
- 🚀 You have a working cluster with Gateway API already installed
- 🔄 You're updating both applications and routes
- 📝 Standard deployment scenario

## Verification

**Check applications only:**
```bash
kubectl get deployments,services,hpa,pdb -n default
```

**Check Gateway resources:**
```bash
kubectl get gateway,httproute -n nginx-gateway
```

## Disabling Gateway Deployment

If you want to temporarily disable Gateway deployment while keeping both values files, you can set `gateway.enabled: false` in `gateway-values.yaml`:

```yaml
gateway:
  enabled: false  # This will skip all Gateway resources
```

Then deploy:
```bash
helm upgrade artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

This will deploy applications only, even though you specified both files.
