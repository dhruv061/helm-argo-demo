# ArgoCD Gateway App - ignoreDifferences Configuration

## Summary

After extensive testing, we've configured comprehensive `ignoreDifferences` rules using `jqPathExpressions` to ignore all controller-added metadata fields.

## What's Being Ignored

### HTTPRoute Resources
Controller-added fields in:
- `parentRefs`: kind, group, name, namespace, sectionName
- `backendRefs`: kind, group, name, namespace, weight
- `certificateRefs` (in TLS): kind, group, name, namespace

### Gateway Resources  
Controller-added fields in:
- `listeners[].tls.certificateRefs`: kind, group, name, namespace
- `status`: Entire status section

### Certificate Resources
All cert-manager added fields:
- `secretTemplate`
- `privateKey` (including algorithm, size, rotationPolicy)

## Why jqPathExpressions Instead of jsonPointers

`jqPathExpressions` provides better array handling:
- `[]?` - Iterate through array elements safely (optional chaining)
- Works with nested arrays and dynamic structures
- More powerful than `jsonPointers` with wildcards

## Full Configuration

```yaml
ignoreDifferences:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    jqPathExpressions:
      - '.spec.parentRefs[]?.kind'
      - '.spec.parentRefs[]?.name'
      - '.spec.rules[]?.backendRefs[]?.kind'
      - '.spec.rules[]?.backendRefs[]?.weight'
      # ... etc
  
  - group: gateway.networking.k8s.io
    kind: Gateway
    jqPathExpressions:
      - '.spec.listeners[]?.tls.certificateRefs[]?.kind'
      - '.status'
```

## Testing

After applying and syncing:
1. Sync the application once in ArgoCD UI
2. If still showing OutOfSync, check what specific fields are different
3. Add those fields to jqPathExpressions as needed

## If Still Not Working

If ignoreDifferences continues to not work, there are two alternatives:

### Alternative 1: Add Fields to Templates
Add the controller-expected fields directly in Helm templates (not recommended - pollutes templates).

### Alternative 2: Disable Auto-Sync Entirely
Accept that Gateway resources will always show OutOfSync and manually sync when needed.
