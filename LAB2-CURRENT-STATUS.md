# 📊 Lab 2 - Trạng thái hiện tại

**Ngày cập nhật**: 2026-06-19  
**Commit cuối**: 19bb4f1 (Revert ESO từ v1 về v1beta1)

---

## 🎯 Tổng quan

Lab 2 gồm 3 phần:
1. **ESO (External Secrets Operator)** - Quản lý secrets động
2. **Trivy** - Quét lỗ hổng bảo mật trong CI
3. **Cosign** - Ký và xác thực images

---

## ✅ Phần hoàn thành

### 1. Trivy Scanning (100% hoàn thành)

**File đã tạo**:
- `.github/workflows/build-push.yml` - CI pipeline với Trivy scan
- `runbooks/trivy-exceptions.md` - Quy trình ghi nhận CVE exceptions

**Tính năng**:
- ✅ Scan image trong CI pipeline
- ✅ Fail CI nếu phát hiện HIGH/CRITICAL vulnerabilities
- ✅ Upload SARIF results lên GitHub Security tab
- ✅ Có quy trình exception (ADR format)

**Kiểm tra**:
```bash
# Xem workflow file
cat .github/workflows/build-push.yml | grep -A 10 "Trivy"

# Xem exception process
cat runbooks/trivy-exceptions.md
```

### 2. Cosign Structure (80% hoàn thành)

**File đã tạo**:
- `signing/cosign.pub` - Public key (placeholder, cần thay bằng key thật)
- `signing/README.md` - Hướng dẫn đầy đủ
- `policies/clusterimagepolicy.yaml` - Policy verification
- `policies/README.md` - Hướng dẫn policy
- `argocd/apps/policy-controller.yaml` - Deploy Sigstore controller (wave -1)
- `argocd/apps/policies.yaml` - Deploy policies (wave 0)
- `.github/workflows/build-push.yml` - CI với Cosign signing step

**Tính năng**:
- ✅ CI pipeline có step sign image với Cosign
- ✅ ClusterImagePolicy đã cấu hình
- ✅ Policy chỉ enforce trên namespace có label `policy.sigstore.dev/include=true`
- ✅ ArgoCD apps với sync-wave đúng thứ tự
- ⚠️ Cần: Generate key pair thật và update files

**Các bước còn lại**:
```bash
# 1. Generate key pair thật
cosign generate-key-pair

# 2. Update signing/cosign.pub với public key thật
cp cosign.pub signing/cosign.pub

# 3. Add private key vào GitHub Secrets
# GitHub → Settings → Secrets → Actions → New secret
# Name: COSIGN_PRIVATE_KEY
# Value: [nội dung file cosign.key]

# 4. Add password vào GitHub Secrets
# Name: COSIGN_PASSWORD
# Value: [password bạn đã nhập khi generate key]

# 5. Update policies/clusterimagepolicy.yaml với public key thật
nano policies/clusterimagepolicy.yaml
# Copy public key vào phần spec.authorities[0].key.data

# 6. Commit và push
git add signing/cosign.pub policies/clusterimagepolicy.yaml
git commit -m "Update Cosign with real key pair"
git push

# 7. Label namespace để enforce policy
kubectl label namespace demo policy.sigstore.dev/include=true
```

---

## 🔧 Phần cần sửa

### 3. ESO (External Secrets Operator) (90% hoàn thành, cần fix credentials)

**File đã tạo**:
- `eso/secret-store.yaml` - Kết nối đến AWS Secrets Manager
- `eso/external-secret.yaml` - Mapping AWS secret → K8s secret
- `eso/README.md` - Hướng dẫn đầy đủ (đã cập nhật với lỗi hiện tại)
- `eso/FIX-CRD-CONFLICT.md` - Troubleshooting CRD issues
- `argocd/apps/eso.yaml` - Deploy ESO operator (wave -1)
- `argocd/apps/eso-config.yaml` - Deploy ESO config (wave 0)

**Tính năng**:
- ✅ ESO operator đã được deploy
- ✅ CRDs đã được cài đặt (v1alpha1 và v1beta1)
- ✅ Manifests sử dụng API version đúng (v1beta1)
- ✅ Sync-wave đúng thứ tự (operator trước, config sau)
- ✅ Refresh interval = 30s (< 60s theo yêu cầu)
- ⚠️ **LỖI**: AWS credentials không hợp lệ

**Trạng thái hiện tại**:
```bash
$ kubectl get externalsecret db-credentials -n demo
NAME             STORE                 REFRESH   STATUS              READY
db-credentials   aws-secrets-manager   30s       SecretSyncedError   False

$ kubectl describe externalsecret db-credentials -n demo
...
Events:
  Warning  UpdateFailed  ...  UnrecognizedClientException: 
  The security token included in the request is invalid. status code: 400
```

**Nguyên nhân**:
Secret `aws-credentials` trong namespace `demo` có giá trị placeholder:
- `access-key-id`: `AKIAEXAMPLE` (không phải real AWS Access Key)
- `secret-access-key`: `secretEXAMPLE` (không phải real AWS Secret Key)

**Giải pháp (QUAN TRỌNG)**:

#### Bước 1: Lấy AWS Credentials

Bạn cần có AWS Access Key ID và Secret Access Key thật từ AWS IAM:

```bash
# Option 1: Tạo IAM user mới (Recommended)
# 1. Đăng nhập AWS Console: https://console.aws.amazon.com/iam/
# 2. IAM → Users → Create user
#    - User name: eso-demo-user
#    - Access type: Programmatic access
# 3. Attach policy: SecretsManagerReadWrite (hoặc custom policy)
# 4. Create user → Download credentials CSV
# 5. Lưu Access Key ID và Secret Access Key

# Option 2: Dùng user hiện có
# 1. IAM → Users → [your-user]
# 2. Security credentials → Create access key
# 3. Use case: Application running outside AWS
# 4. Lưu Access Key ID và Secret Access Key
```

**AWS Policy cần thiết**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret",
        "secretsmanager:CreateSecret",
        "secretsmanager:UpdateSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:demo/*"
    }
  ]
}
```

#### Bước 2: Tạo AWS Secret trên AWS Secrets Manager

```bash
# Cài AWS CLI nếu chưa có
# Windows: choco install awscli
# hoặc tải từ: https://aws.amazon.com/cli/

# Cấu hình AWS CLI với credentials từ Bước 1
aws configure
# AWS Access Key ID [None]: AKIA... (từ Bước 1)
# AWS Secret Access Key [None]: ... (từ Bước 1)
# Default region name [None]: us-east-1
# Default output format [None]: json

# Tạo secret trên AWS Secrets Manager
aws secretsmanager create-secret \
  --name demo/db-credentials \
  --description "Database credentials for demo app" \
  --secret-string '{"username":"admin","password":"changeme123","host":"db.example.com","port":"5432"}' \
  --region us-east-1

# Verify
aws secretsmanager get-secret-value \
  --secret-id demo/db-credentials \
  --region us-east-1
```

#### Bước 3: Update Kubernetes Secret với Real Credentials

```bash
# 1. Xóa secret placeholder cũ
kubectl delete secret aws-credentials -n demo

# 2. Tạo secret mới với credentials THẬT (từ Bước 1)
kubectl create secret generic aws-credentials \
  --from-literal=access-key-id=AKIA... \
  --from-literal=secret-access-key=... \
  -n demo

# ⚠️ QUAN TRỌNG: KHÔNG BAO GIỜ commit secret này vào Git!
# Nó chỉ tồn tại trên cluster, không trong Git repository
```

#### Bước 4: Restart ESO và Verify

```bash
# 1. Restart ESO operator để nhận credentials mới
kubectl rollout restart deployment external-secrets -n external-secrets-system

# 2. Đợi ESO ready (khoảng 30 giây)
kubectl rollout status deployment external-secrets -n external-secrets-system

# 3. Kiểm tra SecretStore
kubectl get secretstore aws-secrets-manager -n demo
# STATUS phải là "Valid", READY phải là "True"

# 4. Kiểm tra ExternalSecret
kubectl get externalsecret db-credentials -n demo
# STATUS phải chuyển từ "SecretSyncedError" → "SecretSynced"
# READY phải chuyển từ "False" → "True"

# 5. Kiểm tra K8s Secret đã được tạo
kubectl get secret db-secret -n demo
# Secret này được ESO tự động tạo từ AWS

# 6. Verify nội dung
kubectl get secret db-secret -n demo -o jsonpath='{.data.db-password}' | base64 -d
# Output: changeme123

# 7. Kiểm tra logs nếu vẫn có lỗi
kubectl logs -n external-secrets-system deployment/external-secrets --tail=50
```

#### Bước 5: Test Auto-Refresh (< 60s requirement)

```bash
# 1. Update password trên AWS
aws secretsmanager update-secret \
  --secret-id demo/db-credentials \
  --secret-string '{"username":"admin","password":"newpassword456","host":"db.example.com","port":"5432"}' \
  --region us-east-1

# 2. Đợi 30-60 giây (refresh interval = 30s)
sleep 60

# 3. Kiểm tra password mới
kubectl get secret db-secret -n demo -o jsonpath='{.data.db-password}' | base64 -d
# Output: newpassword456

# ✅ Nếu password đã thay đổi → ESO hoạt động đúng!
```

---

## 📋 Summary Commands

### Quick Status Check

```bash
echo "=== ESO Status ==="
kubectl get externalsecret -n demo
kubectl get secretstore -n demo
kubectl get secret db-secret -n demo 2>/dev/null || echo "Secret chưa được tạo"

echo -e "\n=== Cosign Status ==="
kubectl get clusterimagepolicy 2>/dev/null || echo "Policy chưa được apply"
kubectl get pods -n cosign-system 2>/dev/null || echo "Policy Controller chưa được deploy"

echo -e "\n=== ArgoCD Apps ==="
kubectl get application -n argocd | grep -E "eso|policy|cosign"
```

### Verify All Working

```bash
# 1. ESO
kubectl get externalsecret db-credentials -n demo -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# Expected: True

# 2. Trivy
# Kiểm tra GitHub Actions → Latest workflow run phải có Trivy step

# 3. Cosign
cosign verify --key signing/cosign.pub ghcr.io/vovudn/w10-api:v0.0.2
# Expected: Verification successful (nếu đã sign image)
```

---

## 🎯 Next Actions

### Để hoàn thành Lab 2:

1. **Fix ESO** (5-10 phút):
   - [ ] Lấy AWS credentials thật từ IAM
   - [ ] Tạo AWS secret trên Secrets Manager
   - [ ] Update Kubernetes secret
   - [ ] Verify ESO working

2. **Complete Cosign** (5 phút):
   - [ ] Generate key pair thật: `cosign generate-key-pair`
   - [ ] Update `signing/cosign.pub`
   - [ ] Add secrets vào GitHub
   - [ ] Update `policies/clusterimagepolicy.yaml`
   - [ ] Commit và push

3. **Test Everything** (5 phút):
   - [ ] Git push → Trigger CI
   - [ ] Verify image được sign
   - [ ] Label namespace và test policy
   - [ ] Test secret rotation

---

## 📚 Documentation Links

- **Main README**: [LAB2-README.md](LAB2-README.md)
- **Setup Guide**: [LAB2-SETUP-GUIDE.md](LAB2-SETUP-GUIDE.md)
- **Summary**: [LAB2-SUMMARY.md](LAB2-SUMMARY.md)
- **ESO Guide**: [eso/README.md](eso/README.md) ⚠️ **Xem đầu file có fix lỗi hiện tại**
- **Cosign Guide**: [signing/README.md](signing/README.md)
- **Policy Guide**: [policies/README.md](policies/README.md)
- **Trivy Exceptions**: [runbooks/trivy-exceptions.md](runbooks/trivy-exceptions.md)

---

## 🆘 Troubleshooting

### ESO vẫn lỗi sau khi update credentials?

```bash
# 1. Verify AWS credentials trong secret
kubectl get secret aws-credentials -n demo -o jsonpath='{.data.access-key-id}' | base64 -d
echo
kubectl get secret aws-credentials -n demo -o jsonpath='{.data.secret-access-key}' | base64 -d
echo

# 2. Test AWS connection từ local
aws secretsmanager get-secret-value \
  --secret-id demo/db-credentials \
  --region us-east-1
# Nếu lỗi → credentials không hợp lệ hoặc không có quyền

# 3. Xem logs chi tiết
kubectl describe externalsecret db-credentials -n demo
kubectl logs -n external-secrets-system deployment/external-secrets --tail=100

# 4. Delete và recreate ExternalSecret
kubectl delete externalsecret db-credentials -n demo
kubectl apply -f eso/external-secret.yaml
```

### ArgoCD app OutOfSync?

```bash
# Force sync
kubectl -n argocd patch application eso-config \
  --type merge \
  --patch '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"syncStrategy":{"hook":{}}}}}'

# Hoặc từ UI: ArgoCD UI → Applications → eso-config → Sync
```

---

**Tóm lại**: Lab 2 gần hoàn thành, chỉ cần fix AWS credentials và generate Cosign keys thật là xong!
