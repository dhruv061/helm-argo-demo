# Helm Chart - Quick Usage Guide

## File Structure

```
artha-helm-chart/
├── gateway-values.yaml       # Gateway API configuration
├── application-values.yaml   # Application components configuration
└── values.yaml              # Combined (optional - for backward compatibility)
```

## Installation

### Using Split Values Files (Recommended)

```bash
# Install with both configuration files
helm install artha . \
  -f gateway-values.yaml \
  -f application-values.yaml
```

### Using Single Values File

```bash
# Install with combined configuration
helm install artha . -f values.yaml
# or simply
helm install artha .
```

## Common Operations

### Update Application Version

1. Edit `application-values.yaml`
2. Change the image tag:
   ```yaml
   applications:
     backend:
       image:
         tag: migrated-v5.0.3  # Update version
   ```
3. Apply changes:
   ```bash
   helm upgrade artha . -f application-values.yaml --reuse-values
   ```

### Add New Client Domain

1. Edit `gateway-values.yaml`
2. Add new domain to the `domains` section:
   ```yaml
   domains:
     - domain: newclient.example.com
       dnsReady: true
       certificateScope: both
       canonicalHost: www
       frontend:
         service: frontend-service
         port: 5010
   ```
3. Apply changes:
   ```bash
   helm upgrade artha . -f gateway-values.yaml --reuse-values
   ```

### Scale Application

1. Edit `application-values.yaml`
2. Change replica count or HPA settings:
   ```yaml
   applications:
     backend:
       replicaCount: 5
       hpa:
         maxReplicas: 10
   ```
3. Apply changes:
   ```bash
   helm upgrade artha . -f application-values.yaml --reuse-values
   ```

### View Generated YAML

```bash
# View all resources
helm template artha . \
  -f gateway-values.yaml \
  -f application-values.yaml

# View only deployments
helm template artha . \
  -f gateway-values.yaml \
  -f application-values.yaml | grep -A 30 "kind: Deployment"
```

### Validate Configuration

```bash
# Lint the chart
helm lint . -f gateway-values.yaml -f application-values.yaml

# Dry run before actual deployment
helm upgrade artha . \
  -f gateway-values.yaml \
  -f application-values.yaml \
  --dry-run
```

## Tips

- **Gateway changes**: Only need to specify `-f gateway-values.yaml --reuse-values`
- **Application changes**: Only need to specify `-f application-values.yaml --reuse-values`
- **Both changes**: Specify both files `-f gateway-values.yaml -f application-values.yaml`
- **Rollback**: Use `helm rollback artha <REVISION>` if something goes wrong
- **History**: View deployment history with `helm history artha`
