# ArgoCD Setup Guide

Complete guide for setting up ArgoCD with automated synchronization for the medical-appointment system.

## 🚀 Quick Start

### 1. Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 2. Access ArgoCD UI

```bash
# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial password
argocd admin initial-password -n argocd

# Or using kubectl
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Open https://localhost:8080 in your browser.

**Initial Login:**
- Username: `admin`
- Password: Use the command above

### 3. Install ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Verify installation
argocd version
```

### 4. Login to ArgoCD

```bash
# Using local cluster
argocd login localhost:8080 --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) --insecure

# Using remote cluster
argocd login argocd.example.com --username admin --password <password>
```

## 📦 Deploy Applications

### Option 1: Deploy via CLI

```bash
# Deploy development application
kubectl apply -f apps/develop/medical-appointment.yaml

# Deploy production application
kubectl apply -f apps/production/medical-appointment.yaml

# Wait for sync
argocd app wait medical-appointment-develop --health
argocd app wait medical-appointment-production --health
```

### Option 2: Deploy via UI

1. Open ArgoCD UI
2. Click "New App" button
3. Fill in application details:
   - **Name**: `medical-appointment-develop`
   - **Project**: `default`
   - **Sync Policy**: Automatic
   - **Repository URL**: `https://github.com/SBillion/medical-appointment-config.git`
   - **Revision**: `develop`
   - **Path**: `charts/medical-appointment`
4. Click "Create"

## 🔧 Configuration

### ArgoCD ConfigMap

```bash
# Edit ArgoCD configuration
kubectl edit configmap argocd-cm -n argocd
```

Key settings:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Application instance label key
  application.instanceLabelKey: argocd.argoproj.io/instance

  # Timeout for operations
  timeout.reconciliation: 180s
  timeout.hard.reconciliation: 0s

  # Resource tracking
  resourceTrackingMethod: label

  # Sync status
  application.syncStatus: comparison

  # Repository credentials
  repositories: |
    - url: https://github.com/SBillion/medical-appointment-config.git
      type: git
      name: medical-appointment-config
```

### Repository Credentials

```bash
# Add private repository (if needed)
argocd repo add https://github.com/SBillion/medical-appointment-config.git \
  --username <username> \
  --password <password>

# List repositories
argocd repo list

# Test repository connection
argocd repo get https://github.com/SBillion/medical-appointment-config.git
```

## 🔄 Auto-Sync Configuration

### Enable Auto-Sync

Applications are already configured with auto-sync enabled:

```yaml
syncPolicy:
  automated:
    prune: true      # Remove resources not in Git
    selfHeal: true   # Auto-correct manual changes
```

### Configure Sync Windows (Optional)

For production, restrict syncs to specific times:

```bash
# Edit ArgoCD configuration
kubectl edit configmap argocd-cm -n argocd
```

Add sync windows:
```yaml
data:
  sync.window: |
    - kind: allow
      schedule: '0 2 * * *'     # 2 AM UTC
      duration: 2h
      applications:
      - medical-appointment-production
```

## 🎛️ Managing Applications

### View Applications

```bash
# List all applications
argocd app list

# Get application details
argocd app get medical-appointment-develop

# Watch application in real-time
argocd app watch medical-appointment-develop
```

### Sync Operations

```bash
# Manual sync
argocd app sync medical-appointment-develop

# Sync with specific options
argocd app sync medical-appointment-develop --prune --self-heal

# Force sync
argocd app sync medical-appointment-develop --force

# Dry run (preview changes)
argocd app sync medical-appointment-develop --dry-run

# Sync and wait for completion
argocd app sync medical-appointment-develop --timeout 300s
```

### Rollback Operations

```bash
# Rollback to previous revision
argocd app rollback medical-appointment-develop

# Rollback to specific revision
argocd app rollback medical-appointment-develop --revision 12345

# View rollback history
argocd app history medical-appointment-develop
```

### Delete Applications

```bash
# Delete application (keeps resources)
argocd app delete medical-appointment-develop --cascade=false

# Delete application and resources
argocd app delete medical-appointment-develop --cascade=true
```

## 📊 Monitoring and Debugging

### Check Application Health

```bash
# Get application status
argocd app get medical-appointment-develop

# Check application resources
argocd app resources medical-appointment-develop

# Get application manifest
argocd app manifest medical-appointment-develop
```

### View Logs

```bash
# Application logs
argocd app logs medical-appointment-develop

# Operation logs
argocd app logs medical-appointment-develop --operation 12345

# Controller logs
kubectl logs -n argocd deployment/argocd-application-controller -f
```

### Debug Sync Issues

```bash
# Check sync status
argocd app get medical-appointment-develop --sync

# Get sync history
argocd app history medical-appointment-develop

# Check diff between Git and cluster
argocd app diff medical-appointment-develop

# Get application details
argocd app get medical-appointment-develop --hard-refresh
```

## 🔐 Security Setup

### Change Admin Password

```bash
# Generate bcrypt hash
htpasswd -bnBC 10 "" newpassword | tr -d ':\n'

# Update password in ArgoCD secret
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "<bcrypt-hash>"}}'

# Restart ArgoCD server
kubectl rollout restart deployment argocd-server -n argocd
```

### Configure RBAC

```bash
# Create RBAC ConfigMap
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:readonly, applications, get, */*, allow
    p, role:readonly, applications, list, */*, allow
    p, role:developer, applications, sync, medical-appointment-develop/*, allow
    p, role:developer, applications, get, medical-appointment-develop/*, allow
    p, role:ops, applications, sync, */*, allow
    p, role:ops, applications, get, */*, allow
  policy.default: role:readonly
EOF
```

### Configure SSO (Optional)

```bash
# Enable OIDC
kubectl patch configmap argocd-cm -n argocd --type merge -p '{
  "data": {
    "url": "https://argocd.example.com",
    "oidc.config": "name: Keycloak\nissuer: https://keycloak.example.com/auth/realms/argocd\nclientId: argocd\nclientSecret: $oidc.keycloak.clientSecret\nrequestedScopes: [openid,profile,email,groups]\nrequestedIDTokenClaims: [groups]"
  }
}'
```

## 📈 Performance Tuning

### Resource Limits

```bash
# Set resource limits for ArgoCD components
kubectl patch deployment argocd-server -n argocd --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/resources",
    "value": {
      "requests": {"cpu": "100m", "memory": "256Mi"},
      "limits": {"cpu": "500m", "memory": "512Mi"}
    }
  }
]'
```

### Sync Optimization

```bash
# Increase sync timeout
kubectl edit configmap argocd-cm -n argocd
```

Add:
```yaml
data:
  timeout.reconciliation: 300s
  timeout.hard.reconciliation: 0s
```

## 🧹 Cleanup

```bash
# Delete applications
argocd app delete medical-appointment-develop --cascade=true
argocd app delete medical-appointment-production --cascade=true

# Delete ArgoCD
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Delete namespace
kubectl delete namespace argocd

# Remove ArgoCD CLI
brew uninstall argocd  # macOS
sudo rm /usr/local/bin/argocd  # Linux
```

## 🔗 Useful Links

- [ArgoCD Documentation](https://argoproj.github.io/argo-cd/)
- [ArgoCD Getting Started](https://argoproj.github.io/argo-cd/getting_started/)
- [ArgoCD Operator Guide](https://argoproj.github.io/argo-cd/operator-manual/)
- [ArgoCD CLI Reference](https://argoproj.github.io/argo-cd/user-guide/commands/)

## 🆘 Troubleshooting

### ArgoCD Server Not Starting

```bash
# Check pod status
kubectl get pods -n argocd

# View logs
kubectl logs -n argocd deployment/argocd-server

# Check events
kubectl get events -n argocd --sort-by='.lastTimestamp'
```

### Sync Not Working

```bash
# Check ArgoCD server connectivity
argocd server --insecure

# Verify repository access
argocd repo get https://github.com/SBillion/medical-appointment-config.git

# Check application status
argocd app get medical-appointment-develop --hard-refresh
```

### Auto-Sync Not Triggering

```bash
# Check sync policy
argocd app get medical-appointment-develop --sync

# Manually trigger sync
argocd app sync medical-appointment-develop

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-repo-server -f
```

### Permission Issues

```bash
# Check current user permissions
argocd account can-i get applications

# List users and roles
argocd account list

# Check RBAC policies
kubectl get configmap argocd-rbac-cm -n argocd -o yaml
```