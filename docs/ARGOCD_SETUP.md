# ArgoCD Admin Guide

Advanced ArgoCD configuration: RBAC, SSO, repository credentials, sync
windows, and performance tuning.

For installation, app deployment, and day-to-day workflows, see
[CONTRIBUTING.md](../CONTRIBUTING.md).

---

## Repository Credentials

If either repo is private, register credentials so ArgoCD can clone them:

```bash
argocd repo add https://github.com/SBillion/medical-appointment.git \
  --username <github-username> --password <github-token>

argocd repo add https://github.com/SBillion/medical-appointment-config.git \
  --username <github-username> --password <github-token>

# Verify
argocd repo list
```

For public repos, no configuration is needed.

---

## RBAC

Restrict who can sync which environments:

```bash
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
    p, role:developer, applications, sync, medical-appointment-development/*, allow
    p, role:developer, applications, get, medical-appointment-development/*, allow
    p, role:ops, applications, sync, */*, allow
    p, role:ops, applications, get, */*, allow
  policy.default: role:readonly
EOF
```

---

## SSO (Optional)

Enable OIDC (e.g. Keycloak):

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '{
  "data": {
    "url": "https://argocd.example.com",
    "oidc.config": "name: Keycloak\nissuer: https://keycloak.example.com/auth/realms/argocd\nclientId: argocd\nclientSecret: $oidc.keycloak.clientSecret\nrequestedScopes: [openid,profile,email,groups]\nrequestedIDTokenClaims: [groups]"
  }
}'
```

---

## Change Admin Password

```bash
# Generate bcrypt hash
htpasswd -bnBC 10 "" newpassword | tr -d ':\n'

# Update the secret
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "<bcrypt-hash>"}}'

# Restart the server
kubectl rollout restart deployment argocd-server -n argocd
```

---

## Sync Windows

Restrict production syncs to specific time windows (e.g. 2 AM UTC only):

```bash
kubectl edit configmap argocd-cm -n argocd
```

```yaml
data:
  sync.window: |
    - kind: allow
      schedule: '0 2 * * *'
      duration: 2h
      applications:
      - medical-appointment-production
```

---

## Notifications

Subscribe to sync events via Slack:

```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: channel-name
    notifications.argoproj.io/subscribe.on-sync-failed.slack: channel-name
    notifications.argoproj.io/subscribe.on-health-degraded.slack: channel-name
```

---

## Git Webhook (Optional)

By default ArgoCD polls Git every 3 minutes. To get instant syncs on push,
configure a GitHub webhook that POSTs to ArgoCD's `/api/webhook` endpoint —
no ArgoCD CLI or GitHub Actions needed.

### 1. Create the webhook on GitHub

On each repository (config repo **and** app repo):

1. Go to **Settings → Webhooks → Add webhook**
2. **Payload URL**: `https://<your-argocd-server>/api/webhook`
3. **Content type**: `application/json` (required — the default
   `application/x-www-form-urlencoded` is not supported)
4. **Secret**: any secure value (optional but recommended)
5. **Events**: "Just the push event"
6. Click **Add webhook**

### 2. Configure the secret in ArgoCD

Store the same secret in the `argocd-secret` Kubernetes secret:

```bash
kubectl edit secret argocd-secret -n argocd
```

Add under `stringData`:

```yaml
stringData:
  webhook.github.secret: <same-secret-as-github>
```

The changes take effect automatically — no restart needed.

### How it works

When GitHub pushes to either repo, it POSTs to `/api/webhook`. ArgoCD
refreshes only the Applications whose `repoURL` matches, and auto-sync
kicks in immediately.

> **Tip:** You can also enable "Packages" events on the app repo webhook
> to trigger a refresh when new images are pushed to GHCR.

---

## Performance Tuning

Adjust ArgoCD server resources and reconciliation timeouts:

```bash
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

```bash
kubectl edit configmap argocd-cm -n argocd
# Set:
#   timeout.reconciliation: 300s
#   timeout.hard.reconciliation: 0s
```

---

## Debugging ArgoCD

```bash
# ArgoCD server logs
kubectl logs -n argocd deployment/argocd-server

# Application controller logs
kubectl logs -n argocd deployment/argocd-application-controller -f

# Repo server logs
kubectl logs -n argocd deployment/argocd-repo-server -f

# Check events
kubectl get events -n argocd --sort-by='.lastTimestamp'
```
