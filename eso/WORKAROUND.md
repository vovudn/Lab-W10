# ESO Workaround - Manual Secret

## ⚠️ Trạng thái hiện tại

ESO (External Secrets Operator) đã được **DISABLE** vì không có AWS credentials thật.

## 🔧 Workaround được áp dụng

### Secret thủ công đã tạo

Thay vì dùng ESO để tự động sync từ AWS Secrets Manager, đã tạo secret `db-secret` **thủ công**:

```bash
kubectl create secret generic db-secret \
  --from-literal=db-username=admin \
  --from-literal=db-password=changeme123 \
  --from-literal=db-host=db.example.com \
  --from-literal=db-port=5432 \
  -n demo
```

### Kiểm tra secret

```bash
# Verify secret tồn tại
kubectl get secret db-secret -n demo

# Xem nội dung
kubectl get secret db-secret -n demo -o yaml
```

## 📋 Cấu trúc Secret

Secret có format giống như ESO sẽ tạo:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: demo
type: Opaque
data:
  db-username: YWRtaW4=           # base64("admin")
  db-password: Y2hhbmdlbWUxMjM=   # base64("changeme123")
  db-host: ZGIuZXhhbXBsZS5jb20=   # base64("db.example.com")
  db-port: NTQzMg==               # base64("5432")
```

## 🎯 Dùng trong Application

Ứng dụng có thể mount secret này như bình thường:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: demo
spec:
  containers:
  - name: app
    image: your-app:latest
    volumeMounts:
    - name: db-credentials
      mountPath: /app/secrets
      readOnly: true
  volumes:
  - name: db-credentials
    secret:
      secretName: db-secret
```

Trong app, đọc từ file:
```python
# Python example
with open('/app/secrets/db-password', 'r') as f:
    db_password = f.read().strip()
```

## ⚠️ Hạn chế của Workaround

### ❌ Không có tính năng ESO:
- **Không auto-refresh**: Secret không tự động update khi thay đổi
- **Không sync từ AWS**: Phải update thủ công trong K8s
- **Không centralized management**: Phải quản lý secret ở nhiều nơi

### ✅ Ưu điểm:
- **Đơn giản**: Không cần AWS account
- **Functional**: App vẫn hoạt động bình thường
- **Testing**: Phù hợp cho lab/demo environment

## 🔄 Để Enable ESO Thật

Khi bạn có AWS account và muốn dùng ESO thật:

### Bước 1: Tạo AWS credentials

```bash
# 1. Đăng nhập AWS Console → IAM
# 2. Create user với policy SecretsManagerReadWrite
# 3. Tạo Access Key
# 4. Lưu Access Key ID và Secret Access Key
```

### Bước 2: Tạo AWS Secret

```bash
aws secretsmanager create-secret \
  --name demo/db-credentials \
  --secret-string '{"username":"admin","password":"changeme123","host":"db.example.com","port":"5432"}' \
  --region us-east-1
```

### Bước 3: Update K8s secret

```bash
# Xóa secret cũ
kubectl delete secret aws-credentials -n demo

# Tạo secret mới với credentials THẬT
kubectl create secret generic aws-credentials \
  --from-literal=access-key-id=AKIA... \
  --from-literal=secret-access-key=... \
  -n demo
```

### Bước 4: Enable ESO auto-sync

```bash
# Uncomment phần automated trong argocd/apps/eso-config.yaml
nano argocd/apps/eso-config.yaml

# Apply
kubectl apply -f argocd/apps/eso-config.yaml
kubectl apply -f eso/secret-store.yaml
kubectl apply -f eso/external-secret.yaml
```

### Bước 5: Xóa secret manual

```bash
# ESO sẽ tự động tạo lại secret db-secret
kubectl delete secret db-secret -n demo

# Verify ESO tạo lại
kubectl get externalsecret db-credentials -n demo
kubectl get secret db-secret -n demo
```

## 📚 Tham khảo

- ESO Configuration: [eso/README.md](README.md)
- ESO Troubleshooting: [FIX-CRD-CONFLICT.md](FIX-CRD-CONFLICT.md)
- Lab 2 Status: [../LAB2-CURRENT-STATUS.md](../LAB2-CURRENT-STATUS.md)

---

**TL;DR**: ESO disabled, dùng secret manual. App vẫn hoạt động nhưng không có auto-refresh.
