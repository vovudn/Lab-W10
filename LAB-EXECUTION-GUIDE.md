# 🚀 Lab W10 - Execution Guide

Hướng dẫn chi tiết từng bước chạy Lab W10 với tất cả câu lệnh.

---

## 📋 Prerequisites

### Required Tools
```bash
# Verify installed tools
docker --version        # Docker Desktop (with Kubernetes)
kubectl version --client
minikube version
git --version
helm version
```

### Git Configuration
```bash
# Set up git (nếu chưa cấu hình)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

---

## 🎯 Phase 0: Setup Local Cluster

### Step 0.1: Start Minikube
```bash
# Create and start minikube cluster
minikube start -p w10 --driver=docker --cpus=4 --memory=6144

# Set as default context
kubectl config use-context w10

# Verify cluster is running
kubectl get nodes
```

**Expected Output**:
```
NAME   STATUS   ROLES           AGE   VERSION
w10    Ready    control-plane   30s   v1.28.3
```

### Step 0.2: Verify Cluster Health
```bash
# Check system namespaces
kubectl get ns

# Check system pods
kubectl get pods -n kube-system

# Verify API server is responsive
kubectl api-resources | head -5
```

---

## 🎯 Phase 1: Setup ArgoCD

### Step 1.1: Create ArgoCD Namespace
```bash
kubectl create namespace argocd
```

### Step 1.2: Install ArgoCD
```bash
# Apply ArgoCD manifests (stable version)
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for server to be ready
kubectl -n argocd wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server --timeout=300s

# Verify ArgoCD is running
kubectl get pods -n argocd
```

**Expected Output**:
```
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-application-controller-xxxxx              1/1     Running   0          45s
argocd-applicationset-controller-xxxxx           1/1     Running   0          45s
argocd-dex-server-xxxxx                          1/1     Running   0          45s
argocd-notifications-controller-xxxxx            1/1     Running   0          45s
argocd-redis-xxxxx                               1/1     Running   0          45s
argocd-repo-server-xxxxx                         1/1     Running   0          45s
argocd-server-xxxxx                              1/1     Running   0          45s
```

### Step 1.3: Access ArgoCD UI (Optional)
```bash
# Port forward to access UI
kubectl -n argocd port-forward svc/argocd-server 8080:443 &

# Get admin password
ARGOCD_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d)
echo "ArgoCD Password: $ARGOCD_PASS"

# Open browser: https://localhost:8080
# Username: admin
# Password: [printed above]
```

---

## 🎯 Phase 2: Create Demo Namespace

### Step 2.1: Create Namespace
```bash
kubectl create namespace demo
```

### Step 2.2: Verify Namespace
```bash
kubectl get namespace demo
```

---

## 🎯 Phase 3: Deploy App-of-Apps (ArgoCD Root)

### Step 3.1: Apply Root Application
```bash
# Deploy ArgoCD App-of-Apps pattern
kubectl apply -f argocd/root.yaml

# Verify root application created
kubectl get application -n argocd root
```

**Expected Output**:
```
NAME   SYNC STATUS   HEALTH STATUS
root   Syncing       Unknown
```

### Step 3.2: Wait for Sync to Complete
```bash
# Watch sync progress (Ctrl+C to stop)
watch kubectl get applications -n argocd

# Or check individual apps
kubectl get applications -n argocd -o wide
```

**Expected progression**:
```
NAME                    SYNC STATUS   HEALTH STATUS   REPO   PATH
root                    Synced        Healthy         ...
rbac                    Synced        Healthy         ...
gatekeeper              Synced        Healthy         ...    (takes 2-3 min)
app-common              Synced        Healthy         ...
k8s-prometheus          Synced        Healthy         ...    (takes 2-3 min)
k8s-rollout             Synced        Healthy         ...
app-analysis            Synced        Healthy         ...
app-alert               Synced        Healthy         ...
app-api                 Synced        Healthy         ...
```

### Step 3.3: Check Sync Waves Order
```bash
# Verify sync-wave ordering
kubectl get applications -n argocd \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.argocd\.argoproj\.io/sync-wave}{"\n"}{end}' \
  | sort -k2 -n
```

**Expected Output**:
```
rbac                  -3
gatekeeper            -2
app-common            -1
k8s-prometheus        0
k8s-rollout           0
app-analysis          1
app-alert             1
app-api               2
```

---

## 🎯 Phase 4: Verify RBAC Configuration

### Step 4.1: Check RBAC Resources
```bash
# List all roles
echo "=== Roles in demo namespace ==="
kubectl get roles -n demo

echo "=== ClusterRoles ==="
kubectl get clusterroles | grep -E "(developer|sre-pods|cluster-viewer)"

echo "=== RoleBindings in demo namespace ==="
kubectl get rolebindings -n demo

echo "=== ClusterRoleBindings ==="
kubectl get clusterrolebindings | grep -E "(alice|bob|carol)"
```

**Expected Output**:
```
=== Roles in demo namespace ===
NAME                 CREATED AT
developer-role       2024-06-18T...

=== ClusterRoles ===
NAME                      CREATED AT
cluster-viewer-role       2024-06-18T...
sre-pods-role            2024-06-18T...

=== RoleBindings ===
NAME                        ROLE                 AGE
alice-developer-binding    Role/developer-role   2m

=== ClusterRoleBindings ===
NAME                      ROLE                           AGE
bob-sre-binding          ClusterRole/sre-pods-role     2m
carol-viewer-binding     ClusterRole/cluster-viewer-role 2m
```

### Step 4.2: Test RBAC Permissions
```bash
# Test Alice (developer in demo namespace)
echo "=== Testing Alice (developer) ==="
kubectl auth can-i create deployments --namespace=demo --as=alice
# Expected: yes
kubectl auth can-i delete pods --namespace=demo --as=alice
# Expected: yes
kubectl auth can-i get pods --namespace=default --as=alice
# Expected: no

# Test Bob (sre - pods cluster-wide)
echo "=== Testing Bob (SRE) ==="
kubectl auth can-i create pods --namespace=demo --as=bob
# Expected: yes
kubectl auth can-i delete pods --namespace=default --as=bob
# Expected: yes
kubectl auth can-i create deployments --namespace=demo --as=bob
# Expected: no

# Test Carol (viewer cluster-wide)
echo "=== Testing Carol (viewer) ==="
kubectl auth can-i get pods --all-namespaces --as=carol
# Expected: yes
kubectl auth can-i list deployments --namespace=demo --as=carol
# Expected: yes
kubectl auth can-i create pods --namespace=demo --as=carol
# Expected: no
```

---

## 🎯 Phase 5: Verify Gatekeeper Controller

### Step 5.1: Wait for Gatekeeper Controller
```bash
# Wait for gatekeeper-system namespace
kubectl wait --for=condition=active namespace gatekeeper-system --timeout=300s 2>/dev/null || true

# Wait for controller pods to be ready
echo "Waiting for Gatekeeper controller (this takes 2-3 minutes)..."
kubectl wait --for=condition=ready pod \
  -l control-plane=controller-manager \
  -n gatekeeper-system --timeout=300s

# Verify all Gatekeeper pods
kubectl get pods -n gatekeeper-system
```

**Expected Output**:
```
NAME                                             READY   STATUS    RESTARTS   AGE
gatekeeper-audit-xxxxx                           1/1     Running   0          2m
gatekeeper-controller-manager-xxxxx              1/1     Running   0          2m
gatekeeper-controller-manager-yyyyy              1/1     Running   0          2m
```

### Step 5.2: Check Gatekeeper CRDs
```bash
# Verify ConstraintTemplate CRD exists
kubectl get crd constrainttemplates.templates.gatekeeper.sh

# Verify Constraint CRDs
kubectl get crd | grep constraints.gatekeeper.sh
```

---

## 🎯 Phase 6: Verify Gatekeeper Templates

### Step 6.1: Check ConstraintTemplates
```bash
# List all templates
echo "Waiting for templates to be ready..."
sleep 30

kubectl get constrainttemplates

# Verify 5 templates created
TEMPLATE_COUNT=$(kubectl get constrainttemplates --no-headers | wc -l)
echo "Total templates: $TEMPLATE_COUNT (expected: 5)"
```

**Expected Output**:
```
NAME                          AGE
k8snolatestimage             1m
k8smaxreplicas               1m
k8srequiredresources         1m
k8spsphostnetworkingports    1m
k8spspallowedusers          1m
```

### Step 6.2: Check Template Status
```bash
# Check each template
kubectl get constrainttemplates -o json | \
  jq '.items[] | {name: .metadata.name, status: .status.created}'

# Check for errors
kubectl describe constrainttemplates
```

---

## 🎯 Phase 7: Verify Gatekeeper Constraints (WARN Mode)

### Step 7.1: Check Constraints
```bash
# List all constraints
echo "Waiting for constraints to be applied..."
sleep 30

kubectl get constraints

# Verify 5 constraints created
CONSTRAINT_COUNT=$(kubectl get constraints --no-headers | wc -l)
echo "Total constraints: $CONSTRAINT_COUNT (expected: 5)"
```

**Expected Output**:
```
NAME                                                           ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8snolatestimage.constraints.gatekeeper.sh/no-latest-image   warn                  <calculating>
k8smaxreplicas.constraints.gatekeeper.sh/max-5-replicas      warn                  <calculating>
k8srequiredresources.constraints.gatekeeper.sh/require-limits warn                  <calculating>
k8spsphostnetworkingports.../no-host-network                 warn                  <calculating>
k8spspallowedusers.constraints.gatekeeper.sh/no-root-user    warn                  <calculating>
```

### Step 7.2: Wait for Audit Scan
```bash
# Audit controller scans existing resources
# This takes 60-90 seconds
echo "Waiting for audit scan to complete (90 seconds)..."
sleep 90

# Check violation counts
kubectl get constraints

# Check audit logs
echo "=== Gatekeeper Audit Logs ==="
kubectl logs -n gatekeeper-system -l control-plane=audit-controller --tail=20
```

### Step 7.3: Review Violations
```bash
# Get detailed violations
echo "=== Constraint Violations ==="
kubectl get constraints -o json | jq '.items[] | {
  name: .metadata.name,
  enforcement: .spec.enforcementAction,
  violations: .status.totalViolations
}'

# Describe specific constraint for details
kubectl describe k8snolatestimage no-latest-image
kubectl describe k8srequiredresources require-resource-limits
kubectl describe k8spsphostnetworkingports no-host-network
kubectl describe k8spspallowedusers no-root-user
kubectl describe k8smaxreplicas max-5-replicas
```

---

## 🎯 Phase 8: Verify Other Components

### Step 8.1: Check Argo Rollouts Controller
```bash
# Verify Argo Rollouts is deployed
kubectl get pods -n argo-rollouts

# Check Rollout CRD
kubectl get crd | grep rollout

# Verify API rollout
kubectl get rollout -n demo
```

**Expected Output**:
```
NAME   DESIRED   CURRENT   UP-TO-DATE   READY   AGE
api    4         4         4            4       1m
```

### Step 8.2: Check Prometheus Stack
```bash
# Verify Prometheus pods
kubectl get pods -n monitoring

# Check ServiceMonitor
kubectl get servicemonitor -n demo

# Verify API metrics are scraped
POD=$(kubectl get pod -n demo -l app=api -o name | head -1)
kubectl port-forward -n demo $POD 8080:8080 &
# Open: http://localhost:8080/metrics
# Kill port-forward: kill %1
```

### Step 8.3: Check AlertManager
```bash
# Verify AlertManager pod
kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager

# Check AlertManager config
kubectl get secret -n monitoring alertmanager-kube-prometheus-stack-alertmanager -o yaml | \
  grep -A 50 "alertmanager.yaml"
```

### Step 8.4: Check API Application
```bash
# Verify API pods are running
kubectl get pods -n demo -l app=api

# Check API service
kubectl get svc -n demo api

# Test API endpoint
kubectl run -it --rm --image=curlimages/curl:latest --restart=Never \
  --namespace=demo test-api -- \
  curl http://api/

# Expected: { "ok": true, "version": "v0.0.1" }
```

---

## 🎯 Phase 9: Test Gatekeeper Policies

### Step 9.1: Test No Latest Tag Policy ❌
```bash
# Try to create pod with :latest tag (should fail/warn)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-latest
  namespace: demo
spec:
  containers:
  - name: nginx
    image: nginx:latest
EOF

# Expected in WARN mode: Pod created with warning in audit logs
# In DENY mode: Error from server: admission webhook "validation.gatekeeper.sh" denied
```

### Step 9.2: Test Required Limits Policy ❌
```bash
# Try to create pod without resource limits (should fail/warn)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-no-limits
  namespace: demo
spec:
  containers:
  - name: nginx
    image: nginx:1.21
EOF

# Expected in WARN mode: Pod created with warning
# In DENY mode: Error about missing limits
```

### Step 9.3: Test Valid Pod ✅
```bash
# Create pod with all policies compliant
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-valid
  namespace: demo
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: nginx
    image: nginx:1.21
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 100m
        memory: 128Mi
EOF

# Expected: Pod should be created successfully
kubectl get pod -n demo test-valid
```

### Step 9.4: Cleanup Test Pods
```bash
kubectl delete pod -n demo test-latest test-no-limits test-valid --ignore-not-found=true
```

---

## 🎯 Phase 10: Monitor Violations (Optional)

### Step 10.1: Setup Monitoring
```bash
# Watch constraints for violations
watch kubectl get constraints

# Or get violation summary
watch -n 5 'kubectl get constraints -o json | jq ".items[] | {name: .metadata.name, violations: .status.totalViolations}"'
```

### Step 10.2: Export Violation Report
```bash
# Save violations to file for analysis
kubectl get constraints -o json > violations-report.json

# Pretty print
cat violations-report.json | jq '.'

# Count violations per policy
jq '.items[] | {policy: .metadata.name, count: .status.totalViolations}' violations-report.json
```

---

## 🎯 Phase 11 (Optional): Switch to DENY Mode

### ⚠️ Important: Only do this after fixing all violations

### Step 11.1: Edit Constraints
```bash
# Edit all-constraints.yaml to change enforcementAction
cat > /tmp/patch-deny.yaml <<'EOF'
spec:
  enforcementAction: deny
EOF

# Update constraints (you need to edit gatekeeper/constraints/all-constraints.yaml)
# Change all: enforcementAction: warn → enforcementAction: deny
# Commit and push to git
git add gatekeeper/constraints/all-constraints.yaml
git commit -m "chore: switch gatekeeper to deny mode"
git push origin main

# Wait for ArgoCD to sync
kubectl wait --for=jsonpath='{.status.operationState.phase}'=Succeeded \
  application gatekeeper -n argocd --timeout=300s
```

### Step 11.2: Verify DENY Mode
```bash
# Check constraints are now in DENY mode
kubectl get constraints

# Try creating invalid pod (should now be rejected)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-should-fail
  namespace: demo
spec:
  containers:
  - name: nginx
    image: nginx:latest
    resources:
      limits:
        cpu: 100m
        memory: 128Mi
EOF

# Expected: Error from server: admission webhook "validation.gatekeeper.sh" denied the request
```

---

## 🛑 Cleanup & Teardown

### Complete Cleanup
```bash
# Delete all resources in demo namespace
kubectl delete all -n demo --all

# Delete demo namespace
kubectl delete namespace demo

# Delete ArgoCD
kubectl delete -f argocd/root.yaml
kubectl delete namespace argocd

# Stop Minikube
minikube stop -p w10
minikube delete -p w10
```

---

## 🆘 Troubleshooting

### Issue: Applications not syncing
```bash
# Check application status
kubectl describe application rbac -n argocd

# Check ArgoCD server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server -f

# Manually retry sync
kubectl patch app rbac -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{}}}}'
```

### Issue: Gatekeeper controller not ready
```bash
# Check controller logs
kubectl logs -n gatekeeper-system -l control-plane=controller-manager -f

# Check webhook configuration
kubectl get validatingwebhookconfigurations

# Restart controller
kubectl rollout restart deployment gatekeeper-controller-manager -n gatekeeper-system
```

### Issue: Templates not created
```bash
# Check template status
kubectl describe constrainttemplate k8snolatestimage

# Check Rego syntax errors
kubectl logs -n gatekeeper-system -l control-plane=controller-manager -f | grep -i "rego\|error"
```

### Issue: Violations not updating
```bash
# Restart audit controller
kubectl rollout restart deployment gatekeeper-audit -n gatekeeper-system

# Force re-audit
kubectl patch constraint no-latest-image --type merge -p '{"metadata":{"annotations":{"gatekeeper.sh/audit":""}}}'

# Check audit logs
kubectl logs -n gatekeeper-system -l control-plane=audit-controller -f
```

---

## ✅ Completion Checklist

- [ ] Minikube cluster running
- [ ] ArgoCD installed and accessible
- [ ] All ArgoCD applications synced
- [ ] RBAC roles and bindings verified
- [ ] Gatekeeper controller running
- [ ] 5 ConstraintTemplates created
- [ ] 5 Constraints in WARN mode
- [ ] Violation audit completed
- [ ] Test policies working
- [ ] Documentation reviewed

---

## 📊 Expected Final State

```bash
# Check everything is working
echo "=== Cluster Health ==="
kubectl get nodes

echo "=== Namespaces ==="
kubectl get ns | grep -E "(argocd|demo|gatekeeper|monitoring|argo-rollouts)"

echo "=== ArgoCD Applications ==="
kubectl get applications -n argocd

echo "=== RBAC ==="
kubectl get roles -n demo
kubectl get clusterroles | grep -E "(sre|viewer)"

echo "=== Gatekeeper ==="
kubectl get constrainttemplates
kubectl get constraints

echo "=== API Application ==="
kubectl get pods -n demo
kubectl get svc -n demo
```

**All green = Lab completed successfully! 🎉**
