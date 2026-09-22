# ArgoCD Auto-Sync Configuration

This directory contains ArgoCD Application manifests with automatic synchronization enabled. When you push changes to the config repository, ArgoCD will automatically update your Kubernetes cluster.

## 🔄 How Auto-Sync Works

### 1. **Automatic Polling**
ArgoCD continuously polls the Git repository (every 3 minutes by default) and automatically syncs when changes are detected.

### 2. **Webhook Triggers**
GitHub Actions workflow triggers immediate ArgoCD sync when config changes are pushed.

### 3. **Self-Healing**
ArgoCD automatically corrects drift between Git and cluster state.

## 📋 Current Configuration

### Development Environment
```yaml
syncPolicy:
  automated:
    prune: true        # Remove resources not in Git
    selfHeal: true     # Auto-correct manual changes
    allowEmpty: false  # Don't sync empty state
```

### Production Environment
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
```

## 🚀 Triggering Manual Syncs

### Using ArgoCD CLI
```bash
# Sync specific application
argocd app sync medical-appointment-development

# Sync with force override
argocd app sync medical-appointment-development --force

# Sync and wait for completion
argocd app sync medical-appointment-development --timeout 300s

# Sync all applications
argocd app sync -a app.kubernetes.io/name=medical-appointment
```

### Using ArgoCD UI
1. Open ArgoCD UI (usually `https://argocd.example.com`)
2. Navigate to the application
3. Click "Sync" button
4. Choose sync options and confirm

### Using kubectl
```bash
# Create a sync trigger
kubectl create -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: medical-appointment-development
  annotations:
    argocd.argoproj.io/sync-wave: "0"
EOF
```

## 🎛️ Sync Options

### Prune Policy
- **Enabled**: Automatically removes resources not in Git
- **Disabled**: Keeps manually created resources

### Self-Heal
- **Enabled**: Auto-corrects manual changes to match Git
- **Disabled**: Allows manual overrides

### Sync Strategies
- **Hook**: Execute resource hooks before/after sync
- **Replace**: Use replace instead of update for resources
- **ServerSideApply**: Use server-side apply (recommended for CRDs)

## ⏰ Sync Windows (Optional)

For production environments, you can restrict syncs to specific time windows:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
  managedNamespaceMetadata:
    labels:
      environment: production
  syncWindows:
  - kind: allow
    schedule: '0 2 * * *'     # 2 AM UTC
    duration: 2h
    applications:
    - medical-appointment-production
  - kind: deny
    schedule: '0 9-17 * * 1-5' # Business hours
    duration: 8h
    applications:
    - medical-appointment-production
```

## 🔔 Notification Hooks

Configure Slack/email notifications for sync events:

```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: channel-name
    notifications.argoproj.io/subscribe.on-sync-failed.slack: channel-name
    notifications.argoproj.io/subscribe.on-health-degraded.slack: channel-name
```

## 🛠️ Troubleshooting

### Check Sync Status
```bash
# Get application status
argocd app get medical-appointment-development

# Check sync history
argocd app history medical-appointment-development

# View operation logs
argocd app logs medical-appointment-development
```

### Debug Sync Issues
```bash
# Check application resources
argocd app resources medical-appointment-development

# Get application manifest
argocd app manifest medical-appointment-development

# Dry-run sync
argocd app sync medical-appointment-development --dry-run
```

### Manual Rollback
```bash
# Rollback to previous revision
argocd app rollback medical-appointment-development

# Rollback to specific revision
argocd app rollback medical-appointment-development --revision 12345
```

### Force Resync
```bash
# Force sync from Git
argocd app sync medical-appointment-development --force

# Hard refresh (delete and recreate)
argocd app sync medical-appointment-development --force --replace
```

## 📊 Monitoring Sync Operations

### Watch Real-time Sync
```bash
# Watch sync progress
argocd app watch medical-appointment-development

# Monitor application health
watch argocd app get medical-appointment-development
```

### Check Sync History
```bash
# List sync operations
argocd app history medical-appointment-development

# Get specific sync details
argocd app get medical-appointment-development --operation 12345
```

## 🔐 Security Considerations

### Secret Management
- Use Sealed Secrets or External Secrets for sensitive data
- Never commit actual secrets to Git
- Rotate ArgoCD admin credentials regularly

### RBAC Configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:developer, applications, sync, medical-appointment-development/*, allow
    p, role:developer, applications, get, medical-appointment-development/*, allow
    p, role:ops, applications, sync, medical-appointment-production/*, allow
    p, role:ops, applications, get, medical-appointment-production/*, allow
```

## 🧪 Testing Sync Configuration

### Test Auto-Sync
```bash
# Make a small change to values-develop.yaml
echo "# test" >> charts/medical-appointment/values-develop.yaml

# Commit and push
git add .
git commit -m "test: verify auto-sync"
git push origin develop

# Watch ArgoCD sync
argocd app watch medical-appointment-development
```

### Test Manual Sync
```bash
# Manually trigger sync
argocd app sync medical-appointment-development

# Verify sync completed
argocd app get medical-appointment-development --sync
```

### Test Self-Healing
```bash
# Make manual change to deployment
kubectl scale deployment backend --replicas=3 -n medical-appointment-development

# Wait for ArgoCD to auto-correct
sleep 10

# Verify replicas restored
kubectl get deployment backend -n medical-appointment-development
```

## 📈 Best Practices

1. **Use Branch Protection**: Require PR reviews for main branch
2. **Test in Dev First**: Always test changes in develop before production
3. **Monitor Sync Failures**: Set up alerts for failed syncs
4. **Document Changes**: Use commit messages to track config changes
5. **Rollback Plan**: Always have a rollback strategy
6. **Resource Limits**: Set appropriate resource requests/limits
7. **Health Checks**: Ensure all resources have proper health checks
8. **Gradual Rollouts**: Use progressive delivery for critical updates

## 🔗 Useful Links

- [ArgoCD Documentation](https://argoproj.github.io/argo-cd/)
- [ArgoCD CLI Reference](https://argoproj.github.io/argo-cd/user-guide/commands/)
- [ArgoCD Automated Sync](https://argoproj.github.io/argo-cd/user-guide/auto_sync/)
- [GitHub Actions for ArgoCD](https://github.com/argoproj/argo-cd)