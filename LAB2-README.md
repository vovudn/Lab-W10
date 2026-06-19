# 🔐 Lab 2: Secrets Management & Image Security

**External Secrets Operator + Trivy Scanning + Cosign Image Signing**

---

## ⚠️ TRẠNG THÁI HIỆN TẠI

### ✅ Hoàn thành
- **Trivy Scanning**: CI pipeline đã tích hợp, scan và fail on HIGH/CRITICAL
- **Cosign Structure**: Manifests và pipeline đã sẵn sàng (cần generate keys thật)

### 🔧 Cần sửa - ESO
**Lỗi**: ExternalSecret không sync được do AWS credentials không hợp lệ.

```
Status: SecretSyncedError
Error: The security token included in the request is invalid (400)
```

**Nguyên nhân**: Secret `aws-credentials` có giá trị placeholder (`AKIAEXAMPLE`), không phải credentials thật.

**Sửa ngay**:
```bash
# 1. Xóa secret cũ
kubectl delete secret aws-credentials -n demo

# 2. Tạo secret với credentials THẬT (lấy từ AWS IAM)
kubectl create secret generic aws-credentials \
  --from-literal=access-key-id=YOUR_REAL_AWS_ACCESS_KEY_ID \
  --from-literal=secret-access-key=YOUR_REAL_AWS_SECRET_ACCESS_KEY \
  -n demo

# 3. Restart ESO để nhận credentials mới
kubectl rollout restart deployment external-secrets -n external-secrets-system

# 4. Verify (sau 30s)
kubectl get externalsecret db-credentials -n demo
# STATUS phải là "SecretSynced", READY phải là "True"
```

Chi tiết xem: **[eso/README.md](eso/README.md)** (có hướng dẫn đầy đủ ở đầu file)

---

## 🎯 Overview

Lab 2 mở rộng Lab 1 với 3 component bảo mật quan trọng:

1. **External Secrets Operator (ESO)** - Quản lý secrets động từ AWS
2. **Trivy** - Quét lỗ hổng bảo mật trong CI/CD
3. **Cosign** - Ký và xác thực container images

---

## 📚 Documentation Structure

### 🚀 Quick Start
**→ [LAB2-SETUP-GUIDE.md](LAB2-SETUP-GUIDE.md)**
- Step-by-step setup từ đầu
- Tất cả commands copy-paste ready
- Prerequisites và verification
- **Read this first to run the lab**

### 📋 Summary & Overview
**→ [LAB2-SUMMARY.md](LAB2-SUMMARY.md)**
- What was implemented
- Architecture diagram
- Requirements checklist
- Testing scenarios
- **Read this to understand what you built**

### 📖 Component Guides

#### External Secrets Operator
**→ [eso/README.md](eso/README.md)**
- Detailed ESO configuration
- AWS Secrets Manager setup
- Troubleshooting guide
- Auto-refresh testing

#### Cosign Signing
**→ [signing/README.md](signing/README.md)**
- Key pair generation
- GitHub Secrets setup
- Manual verification
- Key rotation process

#### Image Policies
**→ [policies/README.md](policies/README.md)**
- ClusterImagePolicy configuration
- Namespace labeling
- Testing enforcement
- Advanced config

#### Trivy Exceptions
**→ [runbooks/trivy-exceptions.md](runbooks/trivy-exceptions.md)**
- CVE exception process (ADR)
- Risk acceptance template
- Review workflow

---

## ⚡ Quick Commands

### Setup (One-time)

```bash
# 1. Create AWS secret
aws secretsmanager create-secret \
  --name demo/db-credentials \
  --secret-string '{"username":"admin","password":"changeme123","host":"db.example.com","port":"5432"}' \
  --region us-east-1

# 2. Create AWS credentials in K8s (NEVER commit!)
kubectl create secret generic aws-credentials \
  --from-literal=access-key-id=AKIAEXAMPLE \
  --from-literal=secret-access-key=secretEXAMPLE \
  --namespace=demo

# 3. Generate Cosign key pair
cosign generate-key-pair
cp cosign.pub signing/cosign.pub

# 4. Add to GitHub Secrets:
#    - COSIGN_PRIVATE_KEY (from cosign.key)
#    - COSIGN_PASSWORD (your password)

# 5. Update ClusterImagePolicy with real public key
nano policies/clusterimagepolicy.yaml

# 6. Label namespace for policy enforcement
kubectl label namespace demo policy.sigstore.dev/include=true
```

### Verify

```bash
# Check ESO
kubectl get externalsecret -n demo
kubectl get secret db-secret -n demo

# Check Policy Controller
kubectl get clusterimagepolicy
kubectl get pods -n cosign-system

# Check image signature
cosign verify --key signing/cosign.pub ghcr.io/vovudn/w10-api:v0.0.2
```

### Test

```bash
# Test signed image (should work)
kubectl run test-signed --image=ghcr.io/vovudn/w10-api:v0.0.2 -n demo

# Test unsigned image (should fail)
kubectl run test-unsigned --image=nginx:latest -n demo
```

---

## 🎓 Learning Path

### Path 1: "Quick Run" (30 min)
```
1. Read LAB2-SETUP-GUIDE.md
2. Follow steps exactly
3. Verify with commands
```

### Path 2: "Understanding" (1 hour)
```
1. Read LAB2-SUMMARY.md (overview)
2. Read LAB2-SETUP-GUIDE.md (execution)
3. Read component READMEs (deep dive)
```

### Path 3: "Complete Mastery" (2 hours)
```
1. LAB2-SUMMARY.md
2. eso/README.md
3. signing/README.md
4. policies/README.md
5. runbooks/trivy-exceptions.md
6. LAB2-SETUP-GUIDE.md
7. Test all scenarios
```

---

## 📁 Files Structure

```
Lab 2 Files:
├── LAB2-README.md              ← You are here
├── LAB2-SETUP-GUIDE.md         ← Step-by-step setup
├── LAB2-SUMMARY.md             ← What was built
│
├── eso/
│   ├── README.md               ← ESO guide
│   ├── secret-store.yaml       ← AWS connection
│   └── external-secret.yaml    ← Secret mapping
│
├── signing/
│   ├── README.md               ← Cosign guide
│   └── cosign.pub              ← Public key
│
├── policies/
│   ├── README.md               ← Policy guide
│   └── clusterimagepolicy.yaml ← Signature verification
│
├── runbooks/
│   ├── README.md
│   └── trivy-exceptions.md     ← CVE ADR
│
└── argocd/apps/
    ├── eso.yaml                ← ESO operator
    ├── eso-config.yaml         ← ESO configuration
    ├── policy-controller.yaml  ← Sigstore controller
    └── policies.yaml           ← Image policies
```

---

## 🔄 How It Works

### ESO Flow
```
AWS Secrets Manager
    ↓ (every 30s)
SecretStore (connection config)
    ↓
ExternalSecret (mapping)
    ↓
K8s Secret (auto-created)
    ↓
Pod Volume Mount (live reload)
```

### CI/CD Flow
```
Git Push
    ↓
GitHub Actions
    ├─ Build image
    ├─ Trivy scan ← Fail if HIGH/CRITICAL
    ├─ Upload SARIF to GitHub Security
    ├─ Sign with Cosign ← Auto-sign
    └─ Push to registry
```

### Runtime Flow
```
Pod Creation Request
    ↓
Policy Controller Webhook
    ├─ Check namespace label
    ├─ Verify image signature ← Cosign verify
    └─ Allow/Deny
         ↓
    Pod Starts/Rejected
```

---

## ✅ Requirements Checklist

### ESO (Part 1)
- [ ] Secrets auto-refresh < 60s
- [ ] No pod restart needed (volume mount)
- [ ] Sync-wave: operator → config
- [ ] AWS credentials manual (not in Git)

### Trivy (Part 2)
- [ ] Scan in CI pipeline
- [ ] Fail on HIGH/CRITICAL
- [ ] Upload to GitHub Security
- [ ] Exception process documented

### Cosign (Part 2)
- [ ] Auto-sign in CI
- [ ] Private key in GitHub Secrets
- [ ] Public key in Git
- [ ] Policy Controller enforcing
- [ ] Namespace label required

---

## 🎯 Success Criteria

After completing Lab 2:

✅ **ESO Working**
```bash
$ kubectl get externalsecret -n demo
NAME              STORE                 REFRESH   STATUS
db-credentials    aws-secrets-manager   30s       SecretSynced
```

✅ **Trivy Scanning**
- CI fails on vulnerabilities
- Results in GitHub Security tab
- Exception process in place

✅ **Cosign Enforcement**
```bash
$ kubectl run test-unsigned --image=nginx:latest -n demo
Error: admission webhook denied: no matching signatures
```

---

## 🆘 Need Help?

| Question | Answer |
|----------|--------|
| How do I start? | Read **LAB2-SETUP-GUIDE.md** |
| What did I build? | Read **LAB2-SUMMARY.md** |
| ESO not working? | Read **eso/README.md** → Troubleshooting |
| Cosign failing? | Read **signing/README.md** → Troubleshooting |
| Policy not enforcing? | Read **policies/README.md** → Testing |
| CI scan failing? | Read **runbooks/trivy-exceptions.md** |

---

## 🔗 Integration with Lab 1

Lab 2 extends Lab 1 with:
- **Lab 1**: RBAC + Gatekeeper + GitOps
- **Lab 2**: + ESO + Trivy + Cosign

All components work together:
```
RBAC (Lab 1)
└─ Controls who can access what

Gatekeeper (Lab 1)
└─ Enforces policy at admission

ESO (Lab 2)
└─ Manages secrets dynamically

Trivy (Lab 2)
└─ Prevents vulnerable images

Cosign (Lab 2)
└─ Ensures image integrity
```

---

## 📊 Monitoring

### ESO Health
```bash
kubectl get secretstore -n demo -o wide
kubectl get externalsecret -n demo -o wide
```

### Trivy Results
- GitHub → Security → Code scanning alerts

### Cosign Enforcement
```bash
kubectl logs -n cosign-system -l app=policy-controller-webhook -f
kubectl get events -n demo | grep "policy.sigstore"
```

---

## 🚀 Quick Start (TL;DR)

```bash
# 1. Prerequisites
aws secretsmanager create-secret --name demo/db-credentials --secret-string '{...}'
kubectl create secret generic aws-credentials --from-literal=access-key-id=... -n demo
cosign generate-key-pair

# 2. Configure
# Add COSIGN_PRIVATE_KEY to GitHub Secrets
cp cosign.pub signing/cosign.pub
# Update policies/clusterimagepolicy.yaml with real key

# 3. Deploy (via ArgoCD)
kubectl apply -f argocd/root.yaml

# 4. Label namespace
kubectl label namespace demo policy.sigstore.dev/include=true

# 5. Verify
kubectl get externalsecret -n demo
kubectl get clusterimagepolicy
cosign verify --key signing/cosign.pub <image>
```

**Done! 🎉**

---

## 📚 Next Steps

### After Lab 2:
- Review GitHub Security alerts
- Test secret rotation
- Try deploying unsigned image
- Document CVE exceptions

### Lab 3 (Coming):
- Service Mesh (Istio)
- Mutual TLS
- Traffic management
- Advanced observability

---

**Ready to start?**

👉 **[Open LAB2-SETUP-GUIDE.md](LAB2-SETUP-GUIDE.md)** to begin!

---

*Lab 2 - Secrets Management & Image Security*  
*Production-grade security for Kubernetes*
