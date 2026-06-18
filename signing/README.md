# Container Image Signing with Cosign

## Overview

Sử dụng Cosign để ký (sign) container images trong CI/CD pipeline, và Sigstore Policy Controller để verify trên Kubernetes cluster.

## Setup Cosign Keys

### Step 1: Install Cosign

```bash
# macOS
brew install cosign

# Linux
wget "https://github.com/sigstore/cosign/releases/download/v2.2.0/cosign-linux-amd64"
mv cosign-linux-amd64 /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign

# Windows
# Download from https://github.com/sigstore/cosign/releases
```

### Step 2: Generate Key Pair

```bash
# Generate key pair (will prompt for password)
cosign generate-key-pair

# This creates:
#   - cosign.key (PRIVATE KEY - NEVER COMMIT)
#   - cosign.pub (PUBLIC KEY - commit to Git)
```

**Output:**
```
Enter password for private key:
Enter password for private key again:
Private key written to cosign.key
Public key written to cosign.pub
```

### Step 3: Add Private Key to GitHub Secrets

```bash
# Copy private key content
cat cosign.key

# Go to GitHub repo → Settings → Secrets and variables → Actions
# Create new secret:
#   Name: COSIGN_PRIVATE_KEY
#   Value: <paste entire cosign.key content>

# Also create password secret:
#   Name: COSIGN_PASSWORD
#   Value: <your password>
```

### Step 4: Commit Public Key

```bash
# Copy public key to signing/ directory
cp cosign.pub signing/cosign.pub

# Commit to Git
git add signing/cosign.pub
git commit -m "feat: add cosign public key for image verification"
git push
```

## CI/CD Integration

The GitHub Actions workflow (`.github/workflows/build-push.yml`) includes:

1. **Trivy Scan**: Scan for vulnerabilities
2. **Build & Push**: Build image and push to registry
3. **Cosign Sign**: Sign the image with private key

### Example Workflow Steps

```yaml
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ghcr.io/${{ github.repository }}:${{ steps.version.outputs.version }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'  # Fail on HIGH/CRITICAL

- name: Sign image with Cosign
  run: |
    echo "${{ secrets.COSIGN_PRIVATE_KEY }}" > cosign.key
    cosign sign --key cosign.key \
      -a "repo=${{ github.repository }}" \
      -a "workflow=${{ github.workflow }}" \
      -a "commit=${{ github.sha }}" \
      ghcr.io/${{ github.repository }}:${{ steps.version.outputs.version }}
    rm cosign.key
  env:
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```

## Verification on Kubernetes

### Manual Verification

```bash
# Verify image signature
cosign verify \
  --key signing/cosign.pub \
  ghcr.io/your-org/api:v1.0.0

# Expected output:
# Verification for ghcr.io/your-org/api:v1.0.0 --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key
```

### Automated Verification (Sigstore Policy Controller)

Policy Controller automatically verifies images before allowing pod creation:

```yaml
# ClusterImagePolicy
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
  - glob: "ghcr.io/your-org/**"
  authorities:
  - key:
      data: |
        -----BEGIN PUBLIC KEY-----
        <content from cosign.pub>
        -----END PUBLIC KEY-----
```

## Testing

### Test 1: Sign and Verify Image

```bash
# Build and push test image
docker build -t ghcr.io/your-org/test:v1.0.0 .
docker push ghcr.io/your-org/test:v1.0.0

# Sign image
cosign sign --key cosign.key ghcr.io/your-org/test:v1.0.0

# Verify signature
cosign verify --key cosign.pub ghcr.io/your-org/test:v1.0.0
```

### Test 2: Deploy Signed Image (Should Pass)

```bash
kubectl run test-signed \
  --image=ghcr.io/your-org/api:v1.0.0 \
  --namespace=demo

# Should create successfully if:
# - Image is signed
# - Namespace has label: policy.sigstore.dev/include=true
```

### Test 3: Deploy Unsigned Image (Should Fail)

```bash
kubectl run test-unsigned \
  --image=nginx:latest \
  --namespace=demo

# Should fail with:
# Error: admission webhook denied the request: 
# no matching signatures found for image nginx:latest
```

## Namespace Configuration

### Enable Policy for Namespace

```bash
# Label namespace to enable policy enforcement
kubectl label namespace demo policy.sigstore.dev/include=true

# Verify label
kubectl get namespace demo --show-labels
```

### Disable Policy for Namespace

```bash
# Remove label to disable enforcement
kubectl label namespace demo policy.sigstore.dev/include-

# Or set to false
kubectl label namespace demo policy.sigstore.dev/include=false --overwrite
```

## Troubleshooting

### Issue: Signature verification failed

```bash
# Check if image is signed
cosign verify --key cosign.pub ghcr.io/your-org/api:v1.0.0

# List signatures
cosign tree ghcr.io/your-org/api:v1.0.0

# Check policy controller logs
kubectl logs -n cosign-system -l app=policy-controller -f
```

### Issue: Pod creation denied

```bash
# Check ClusterImagePolicy
kubectl get clusterimagepolicy

# Describe policy
kubectl describe clusterimagepolicy require-signed-images

# Check namespace label
kubectl get namespace demo --show-labels | grep sigstore

# Temporarily disable for testing
kubectl label namespace demo policy.sigstore.dev/include=false --overwrite
```

### Issue: CI fails to sign

```bash
# Check GitHub Secrets are set:
# - COSIGN_PRIVATE_KEY
# - COSIGN_PASSWORD

# Verify workflow has permissions:
# permissions:
#   packages: write
#   contents: read
#   id-token: write
```

## Security Best Practices

✅ **DO:**
- Generate strong key pair with password
- Store private key in GitHub Secrets only
- Use different keys for different environments
- Rotate keys periodically
- Audit signature verification logs
- Label only production namespaces for enforcement

❌ **DON'T:**
- Commit private key to Git
- Share private key across teams
- Use keyless signing in production (requires OIDC)
- Disable verification in production
- Use same key for all projects

## Key Rotation

### Step 1: Generate New Key Pair

```bash
cosign generate-key-pair

# This creates new cosign.key and cosign.pub
```

### Step 2: Update GitHub Secrets

```bash
# Update COSIGN_PRIVATE_KEY with new key content
# Update COSIGN_PASSWORD if changed
```

### Step 3: Re-sign Existing Images

```bash
# Re-sign all images with new key
for tag in v1.0.0 v1.0.1 v1.0.2; do
  cosign sign --key cosign.key ghcr.io/your-org/api:$tag
done
```

### Step 4: Update Public Key in Git

```bash
# Replace old public key
cp cosign.pub signing/cosign.pub
git add signing/cosign.pub
git commit -m "chore: rotate cosign public key"
git push
```

### Step 5: Update ClusterImagePolicy

```bash
# ArgoCD will auto-sync new public key from Git
# Or manually update:
kubectl edit clusterimagepolicy require-signed-images
```

## References

- [Cosign Documentation](https://docs.sigstore.dev/cosign/overview/)
- [Sigstore Policy Controller](https://docs.sigstore.dev/policy-controller/overview/)
- [Best Practices](https://docs.sigstore.dev/cosign/best-practices/)
