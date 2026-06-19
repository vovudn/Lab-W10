# Fix External Secrets CRD Conflict

## Problem

```
CustomResourceDefinition.apiextensions.k8s.io "externalsecrets.external-secrets.io" is invalid: 
status.storedVersions[0]: Invalid value: "v1": missing from spec.versions
```

This happens when:
1. Old ESO version installed with CRD v1
2. New ESO version doesn't include v1 in spec.versions
3. Kubernetes refuses to update CRD

## Solution

### Option 1: Delete and Recreate CRDs (SAFEST)

```bash
# 1. Delete ArgoCD application (will delete all ESO resources)
kubectl delete application external-secrets -n argocd

# 2. Wait for complete deletion
kubectl wait --for=delete namespace external-secrets-system --timeout=120s

# 3. Manually delete CRDs if they still exist
kubectl delete crd externalsecrets.external-secrets.io
kubectl delete crd secretstores.external-secrets.io
kubectl delete crd clustersecretstores.external-secrets.io

# 4. Verify CRDs are gone
kubectl get crd | grep external-secrets
# Should return nothing

# 5. Re-apply ArgoCD application
kubectl apply -f argocd/apps/eso.yaml

# 6. Wait for sync
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=external-secrets \
  -n external-secrets-system --timeout=300s
```

### Option 2: Patch CRD (RISKY - may fail)

```bash
# Try to patch CRD to remove stored version
kubectl patch crd externalsecrets.external-secrets.io \
  --type=json \
  -p='[{"op": "remove", "path": "/status/storedVersions/0"}]'

# If that doesn't work, force replace
kubectl replace -f - <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: externalsecrets.external-secrets.io
spec:
  group: external-secrets.io
  names:
    kind: ExternalSecret
    listKind: ExternalSecretList
    plural: externalsecrets
    shortNames:
    - es
    singular: externalsecret
  scope: Namespaced
  versions:
  - name: v1beta1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        x-kubernetes-preserve-unknown-fields: true
EOF
```

### Option 3: Force Replace (NUCLEAR OPTION)

```bash
# WARNING: This will delete all ExternalSecret resources!

# 1. Backup existing ExternalSecrets
kubectl get externalsecret -A -o yaml > externalsecrets-backup.yaml

# 2. Delete all ExternalSecret instances
kubectl delete externalsecret --all -A

# 3. Delete CRDs
kubectl delete crd externalsecrets.external-secrets.io --force --grace-period=0
kubectl delete crd secretstores.external-secrets.io --force --grace-period=0
kubectl delete crd clustersecretstores.external-secrets.io --force --grace-period=0

# 4. Re-install ESO via ArgoCD
kubectl apply -f argocd/apps/eso.yaml

# 5. Wait for CRDs to be created
sleep 30

# 6. Restore ExternalSecrets (if needed)
kubectl apply -f externalsecrets-backup.yaml
```

## Recommended Approach

**Use Option 1** if:
- ✅ You're setting up for first time
- ✅ No critical data in ExternalSecrets
- ✅ Can recreate from Git

**Use Option 2** if:
- ⚠️ Have existing ExternalSecrets to preserve
- ⚠️ Can't afford downtime

**Use Option 3** if:
- 🚨 Options 1 & 2 failed
- 🚨 Complete reset acceptable

## Prevention

To avoid this in future:

### 1. Updated ArgoCD App with Replace Option

The `argocd/apps/eso.yaml` has been updated with:

```yaml
syncOptions:
- Replace=true  # Allow replacing CRDs
```

### 2. Use Specific CRD Version

Pin ESO version to avoid breaking changes:

```yaml
source:
  chart: external-secrets
  targetRevision: 0.9.11  # Pin specific version
```

### 3. Test Upgrades in Dev First

Before upgrading ESO:
1. Test in dev/staging cluster
2. Review CRD changes
3. Backup existing resources
4. Upgrade production

## Verification

After fix:

```bash
# Check CRDs
kubectl get crd | grep external-secrets

# Should show:
# clustersecretstores.external-secrets.io
# externalsecrets.external-secrets.io
# secretstores.external-secrets.io

# Check ESO operator
kubectl get pods -n external-secrets-system

# Check version
kubectl get deployment -n external-secrets-system external-secrets -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## If Still Failing

### Check ArgoCD App Status

```bash
kubectl describe application external-secrets -n argocd

# Look for:
# - Sync status
# - Health status
# - Last operation
# - Conditions
```

### Check ArgoCD Logs

```bash
# ArgoCD application controller
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller -f

# Look for CRD-related errors
```

### Manual CRD Installation

If ArgoCD keeps failing:

```bash
# Install CRDs manually first
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Install just the CRDs
helm template external-secrets external-secrets/external-secrets \
  --namespace external-secrets-system \
  --set installCRDs=true \
  --include-crds > /tmp/eso-crds.yaml

# Apply CRDs only
kubectl apply -f /tmp/eso-crds.yaml

# Then let ArgoCD install the operator
kubectl apply -f argocd/apps/eso.yaml
```

## References

- [External Secrets Operator CRD Issues](https://github.com/external-secrets/external-secrets/issues)
- [Kubernetes CRD Versioning](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)
- [ArgoCD Replace Sync Option](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#replace-resource-instead-of-applying-changes)
