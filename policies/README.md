# Image Signature Policies

## Overview

Sigstore Policy Controller enforces image signature verification using Cosign.

## ClusterImagePolicy

### Structure

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
  - glob: "ghcr.io/vovudn/**"  # Images to verify
  
  authorities:
  - name: cosign-key
    key:
      data: |
        -----BEGIN PUBLIC KEY-----
        <public key content>
        -----END PUBLIC KEY-----
  
  match:
  - selector:
      matchLabels:
        policy.sigstore.dev/include: "true"  # Only enforced on labeled namespaces
```

### Key Components

1. **images**: Glob pattern matching images to verify
   - `ghcr.io/vovudn/**` - All images from your org
   - `**` - All images (use with caution!)

2. **authorities**: How to verify signatures
   - `key.data`: Public key for verification
   - Can also use: `keyless` (OIDC), `static`, `attestations`

3. **match**: Which pods to enforce on
   - `selector.matchLabels`: Namespace label selector
   - Only enforces on namespaces with `policy.sigstore.dev/include=true`

## Namespace Labeling

### Enable Policy

```bash
# Label namespace to enable enforcement
kubectl label namespace demo policy.sigstore.dev/include=true

# Verify
kubectl get namespace demo --show-labels
```

### Disable Policy

```bash
# Remove label
kubectl label namespace demo policy.sigstore.dev/include-

# Or set to false
kubectl label namespace demo policy.sigstore.dev/include=false --overwrite
```

### Check All Labeled Namespaces

```bash
kubectl get namespaces -L policy.sigstore.dev/include
```

## Testing

### Test Signed Image

```bash
# Deploy image from CI (signed automatically)
kubectl run test-signed \
  --image=ghcr.io/vovudn/w10-api:v0.0.2 \
  --namespace=demo

# Should succeed
```

### Test Unsigned Image

```bash
# Try unsigned image
kubectl run test-unsigned \
  --image=nginx:latest \
  --namespace=demo

# Should fail with error:
# Error: admission webhook denied the request: no matching signatures
```

## Troubleshooting

### Check Policy Status

```bash
# Get policy
kubectl get clusterimagepolicy

# Describe
kubectl describe clusterimagepolicy require-signed-images
```

### Check Webhook

```bash
# Get webhook config
kubectl get validatingwebhookconfigurations

# Logs
kubectl logs -n cosign-system -l app=policy-controller-webhook -f
```

### Debug Image Verification

```bash
# Manual verification
cosign verify \
  --key signing/cosign.pub \
  ghcr.io/vovudn/w10-api:v0.0.2

# Check signature exists
cosign tree ghcr.io/vovudn/w10-api:v0.0.2
```

## Advanced Configuration

### Multiple Authorities

```yaml
spec:
  authorities:
  - name: team-a-key
    key:
      data: |
        <team A public key>
  - name: team-b-key
    key:
      data: |
        <team B public key>
```

### Keyless Verification (OIDC)

```yaml
spec:
  authorities:
  - keyless:
      url: https://fulcio.sigstore.dev
      identities:
      - issuer: https://token.actions.githubusercontent.com
        subject: https://github.com/vovudn/Lab-W10/.github/workflows/build-push.yml@refs/heads/main
```

### Attestation Policies

```yaml
spec:
  authorities:
  - name: attestation-authority
    attestations:
    - name: must-have-sbom
      predicateType: https://spdx.dev/Document
      policy:
        type: cue
        data: |
          predicateType: "https://spdx.dev/Document"
```

## Best Practices

✅ **DO:**
- Start with `warn` mode in dev/staging
- Label only production namespaces
- Use specific image patterns (not `**`)
- Rotate keys periodically
- Monitor policy controller logs

❌ **DON'T:**
- Enable globally without testing
- Use same key for all environments
- Disable verification in production
- Ignore policy failures

## References

- [Policy Controller Docs](https://docs.sigstore.dev/policy-controller/overview/)
- [ClusterImagePolicy Spec](https://docs.sigstore.dev/policy-controller/clusterimagepolicy/)
