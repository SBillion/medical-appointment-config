# Contributing

This project uses **GitOps**: the config repo is the single source of truth
for the cluster. ArgoCD pulls the Helm chart from the app repo and the env
values from this repo, and syncs everything automatically.

---

## Prerequisites

| Tool | Install (macOS) | Verify |
|------|-----------------|--------|
| minikube | `brew install minikube` | `minikube version` |
| kubectl | `brew install kubectl` | `kubectl version --client` |
| argocd CLI | `brew install argocd` | `argocd version` |

> `helm` is only needed for local chart rendering
> ([see below](#test-a-helm-render-locally)).

Clone both repos side by side:

```
~/Projects/
├── medical-appointment/          # app source code + Helm chart
└── medical-appointment-config/   # this repo (env values + ArgoCD)
```

---

## 1. Start the local cluster

```bash
minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=20gb
minikube addons enable ingress
minikube addons enable metrics-server  # required for HPA (autoscaling)
kubectl get nodes
```

## 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

Access the UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
# Password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Login with the CLI:

```bash
argocd login localhost:8080 \
  --username admin \
  --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)" \
  --insecure
```

## 3. Register the ApplicationSet

This is the only manual `kubectl apply`. ArgoCD creates the Applications,
namespaces, and syncs all resources automatically:

```bash
kubectl apply -f apps/medical-appointment.yaml

# Watch ArgoCD create and sync both apps
argocd app get medical-appointment-development --watch
```

ArgoCD will:
1. Pull the Helm chart from the app repo (`main` → `deploy/charts/medical-appointment`)
2. Merge the env override from this repo (`envs/<env>/values.yaml`)
3. Sync all resources (backend, frontend, postgres, ingress) into
   `medical-appointment-<env>` namespaces
4. Keep polling — any new commit auto-syncs

## 4. Access the application

```bash
echo "$(minikube ip) medical-appointment.local" | sudo tee -a /etc/hosts
curl http://medical-appointment.local/api/health
open http://medical-appointment.local
```

If ingress isn't resolving, fall back to port-forward:

```bash
kubectl port-forward svc/backend  8000:80 -n medical-appointment-development &
kubectl port-forward svc/frontend 5173:80 -n medical-appointment-development &
```

---

## Development workflows

### Change application code (backend / frontend)

Code changes go through the GitOps pipeline — no manual image builds:

1. Open a PR on the [app repository](https://github.com/SBillion/medical-appointment)
2. Merge the PR into `main`
3. The `Release` workflow builds images (SHA-tagged) and opens/updates the
   deploy PR on this repo
4. **development**: deploy PR is auto-merged → ArgoCD syncs
5. **production**: review and merge the deploy PR manually → ArgoCD syncs

```bash
argocd app get medical-appointment-development --watch
```

### Change Kubernetes config (Helm values)

Env values live in `envs/`. Edit and push — ArgoCD picks up the change:

```bash
vim envs/development/values.yaml
git add -A && git commit -m "chore: update development values" && git push origin main

# ArgoCD auto-syncs, or trigger manually:
argocd app sync medical-appointment-development
```

Because `syncPolicy.automated.selfHeal: true`, any manual `kubectl edit`
will be reverted to match Git.

### Test a Helm render locally

```bash
helm template medical-appointment ../medical-appointment/deploy/charts/medical-appointment \
  --values envs/development/values.yaml
```

### Diff what a values change would apply

```bash
helm diff upgrade medical-appointment ../medical-appointment/deploy/charts/medical-appointment \
  --namespace medical-appointment-development \
  --values envs/development/values.yaml
```

(Requires `helm plugin install https://github.com/databus23/helm-diff`)

---

## Database

```bash
# Connect
kubectl exec -it statefulset/postgres -n medical-appointment-development -- \
  psql -U app -d medical_appointment

# Reset (destructive)
kubectl delete pvc data-postgres-0 -n medical-appointment-development
kubectl rollout restart statefulset/postgres -n medical-appointment-development
```

---

## Useful commands

| What | Command |
|------|---------|
| List pods | `kubectl get pods -n medical-appointment-development` |
| Backend logs | `kubectl logs -f deployment/backend -n medical-appointment-development` |
| Frontend logs | `kubectl logs -f deployment/frontend -n medical-appointment-development` |
| DB logs | `kubectl logs -f statefulset/postgres -n medical-appointment-development` |
| Resource usage | `kubectl top pods -n medical-appointment-development` |
| Shell into backend | `kubectl exec -it deployment/backend -n medical-appointment-development -- /bin/bash` |
| ArgoCD status | `argocd app get medical-appointment-development` |
| Force sync | `argocd app sync medical-appointment-development --force` |
| Watch sync | `argocd app watch medical-appointment-development` |

---

## Troubleshooting

### ImagePullBackOff

Images come from GHCR (`ghcr.io/sbillion/medical-appointment-*`). Check
available images at [GitHub Packages](https://github.com/SBillion/medical-appointment/pkgs).
For local dev without GHCR, point values to a locally built image with
`pullPolicy: Never`.

### CrashLoopBackOff

```bash
kubectl logs <pod-name> -n medical-appointment-development
kubectl describe pod <pod-name> -n medical-appointment-development
```

### ArgoCD shows "Out of Sync"

```bash
argocd app diff medical-appointment-development
argocd app sync medical-appointment-development
```

### Ingress not working

```bash
kubectl get ingress -n medical-appointment-development
kubectl describe ingress -n medical-appointment-development
minikube addons enable ingress
```

---

## Cleanup

```bash
# Delete the ApplicationSet (ArgoCD prunes all managed resources)
kubectl delete -f apps/medical-appointment.yaml

# Remove ArgoCD
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd

# Remove /etc/hosts entry
sudo sed -i '' '/medical-appointment.local/d' /etc/hosts

# Stop minikube
minikube stop
minikube delete
```
