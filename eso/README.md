# External Secrets Operator (ESO) Configuration

## Overview

ESO tự động sync secrets từ AWS Secrets Manager vào Kubernetes với refresh interval < 60s.

## Architecture

```
AWS Secrets Manager
       ↓
  SecretStore (kết nối config)
       ↓
  ExternalSecret (mapping & refresh)
       ↓
  K8s Secret (tự động tạo)
       ↓
  Pod mount as volume (live reload)
```

## Prerequisites

### 1. AWS Account with Secrets Manager

Tạo secret trên AWS:
```bash
aws secretsmanager create-secret \
  --name demo/db-credentials \
  --description "Database credentials for demo app" \
  --secret-string '{
    "username": "admin",
    "password": "changeme123",
    "host": "db.example.com",
    "port": "5432"
  }' \
  --region us-east-1
```

### 2. AWS IAM User/Role

Tạo IAM user với policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:demo/*"
    }
  ]
}
```

Get access keys:
```bash
aws iam create-access-key --user-name eso-demo-user
```

### 3. Create K8s Secret for AWS Credentials

**⚠️ QUAN TRỌNG: TẠO THỦ CÔNG, KHÔNG COMMIT VÀO GIT**

```bash
# Create AWS credentials secret (LOCAL ONLY)
kubectl create secret generic aws-credentials \
  --from-literal=access-key-id=AKIAIOSFODNN7EXAMPLE \
  --from-literal=secret-access-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  --namespace=demo

# Verify
kubectl get secret aws-credentials -n demo
```

**NEVER commit this secret to Git!** Add to `.gitignore`:
```
eso/aws-credentials.yaml
eso/*-credentials*.yaml
```

## Deployment Order

### Wave -1: Install ESO Operator
```bash
# Deployed via argocd/apps/eso.yaml
# Creates CRDs: SecretStore, ExternalSecret, ClusterSecretStore
```

### Wave 0: SecretStore
```bash
# eso/secret-store.yaml
# Defines connection to AWS Secrets Manager
```

### Wave 1: ExternalSecret
```bash
# eso/external-secret.yaml
# Maps AWS secret -> K8s secret with refresh interval
```

## Verification

### Check ESO Operator
```bash
# Check operator running
kubectl get pods -n external-secrets-system

# Check CRDs installed
kubectl get crd | grep external-secrets
```

### Check SecretStore
```bash
# Get SecretStore
kubectl get secretstore -n demo

# Describe for status
kubectl describe secretstore aws-secrets-manager -n demo
```

### Check ExternalSecret
```bash
# Get ExternalSecret
kubectl get externalsecret -n demo

# Check status (should be SecretSynced)
kubectl get externalsecret db-credentials -n demo -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# Expected: True

# Check last sync time
kubectl get externalsecret db-credentials -n demo -o jsonpath='{.status.syncedResourceVersion}'
```

### Check Generated K8s Secret
```bash
# Secret should be auto-created by ESO
kubectl get secret db-secret -n demo

# Check content (base64 encoded)
kubectl get secret db-secret -n demo -o jsonpath='{.data.db-password}' | base64 -d
echo

# Check labels
kubectl get secret db-secret -n demo -o jsonpath='{.metadata.labels}'
```

## Testing Auto-Refresh (< 60s requirement)

### Step 1: Update Secret on AWS
```bash
# Update password on AWS
aws secretsmanager update-secret \
  --secret-id demo/db-credentials \
  --secret-string '{
    "username": "admin",
    "password": "newpassword456",
    "host": "db.example.com",
    "port": "5432"
  }' \
  --region us-east-1
```

### Step 2: Wait and Verify (< 60s)
```bash
# Watch for changes (refresh every 5s)
watch -n 5 'kubectl get secret db-secret -n demo -o jsonpath="{.data.db-password}" | base64 -d'

# Or check ExternalSecret sync time
watch kubectl get externalsecret db-credentials -n demo
```

**Expected**: Password updates within 30-60 seconds (refreshInterval: 30s)

### Step 3: Verify Pod Gets New Secret (Volume Mount)

**Pod must mount secret as volume (NOT env var) for live reload:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-secret-refresh
  namespace: demo
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    - name: db-creds
      mountPath: /etc/secrets
      readOnly: true
    command:
    - /bin/sh
    - -c
    - |
      while true; do
        echo "Password: $(cat /etc/secrets/db-password)"
        sleep 10
      done
  volumes:
  - name: db-creds
    secret:
      secretName: db-secret
```

**Test:**
```bash
# Create test pod
kubectl apply -f test-pod.yaml

# Watch logs for password changes
kubectl logs -f test-secret-refresh -n demo

# Update password on AWS → should see new password in logs within 60s
```

## Using in Application

### Example: Mount Secret in API Rollout

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api
  namespace: demo
spec:
  template:
    spec:
      containers:
      - name: api
        image: ghcr.io/your-org/api:v1.0.0
        volumeMounts:
        - name: db-credentials
          mountPath: /app/secrets
          readOnly: true
        # Application reads from: /app/secrets/db-password
        # NOT using env vars!
      volumes:
      - name: db-credentials
        secret:
          secretName: db-secret
```

**Application code** reads secret from file:
```python
# Python example
with open('/app/secrets/db-password', 'r') as f:
    db_password = f.read().strip()

# Reload on file change (inotify or periodic check)
```

## Troubleshooting

### Issue: SecretStore not ready
```bash
# Check logs
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets

# Check AWS credentials
kubectl get secret aws-credentials -n demo

# Test AWS connection manually
aws secretsmanager get-secret-value \
  --secret-id demo/db-credentials \
  --region us-east-1
```

### Issue: ExternalSecret not syncing
```bash
# Check ExternalSecret status
kubectl describe externalsecret db-credentials -n demo

# Check events
kubectl get events -n demo --sort-by='.lastTimestamp' | grep -i external

# Force refresh (delete and recreate)
kubectl delete externalsecret db-credentials -n demo
kubectl apply -f eso/external-secret.yaml
```

### Issue: Secret not updating in pod
```bash
# Check if using volume mount (NOT env var)
kubectl get pod <pod-name> -n demo -o yaml | grep -A 10 volumeMounts

# Check secret content
kubectl get secret db-secret -n demo -o yaml

# Restart pod to remount (if needed, but volume should auto-update)
kubectl rollout restart deployment/api -n demo
```

## Security Best Practices

✅ **DO:**
- Create AWS credentials secret manually via kubectl
- Use IAM roles with least privilege
- Rotate AWS access keys regularly
- Use different AWS secrets for different environments
- Monitor ESO operator logs
- Set appropriate refreshInterval (30s for < 60s requirement)
- Use volume mounts for live reload (not env vars)

❌ **DON'T:**
- Commit AWS credentials to Git
- Use root AWS account credentials
- Share AWS secrets across namespaces
- Use overly permissive IAM policies
- Set refreshInterval too low (< 10s causes API throttling)
- Use environment variables (can't reload without pod restart)

## Monitoring

### Metrics (if ESO metrics enabled)
```bash
# Check sync success rate
kubectl port-forward -n external-secrets-system svc/external-secrets-webhook 8080:8080

# Metrics endpoint
curl http://localhost:8080/metrics | grep external_secrets
```

### Alerts
Monitor:
- ExternalSecret status != Ready
- Last sync time > refreshInterval + 30s
- AWS API errors in logs

## References

- [External Secrets Operator Docs](https://external-secrets.io/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
- [Best Practices](https://external-secrets.io/latest/guides/best-practices/)
