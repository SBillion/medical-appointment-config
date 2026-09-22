# Contributing

This project uses **GitOps**: the config repo (`medical-appointment-config`) is the single source of truth for the cluster. Every change goes through Git, and ArgoCD pulls it into the cluster automatically.

---

## Prerequisites

| Tool | Install (macOS) | Verify |
|------|-----------------|--------|
| minikube | `brew install minikube` | `minikube version` |
| kubectl | `brew install kubectl` | `kubectl version --client` |
| helm | `brew install helm` | `helm version` |
| argocd CLI | `brew install argocd` | `argocd version` |

You also need the **application repo** cloned next to this one:

```
~/Projects/
├── medical-appointment/          # app source code
└── medical-appointment-config/   # this repo (Helm + ArgoCD)
```

---

## 1. Start the local cluster

```bash
minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=20gb
minikube addons enable ingress
kubectl get nodes   # should show 1 Ready node
```

## 2. Build images inside minikube's Docker daemon

```bash
eval $(minikube docker-env)          # redirect docker CLI to minikube

cd ../medical-appointment
docker build -t medical-appointment-backend:local  ./backend
docker build -t medical-appointment-frontend:local ./frontend

docker images | grep medical-appointment   # verify
```

> `imagePullPolicy: Never` in `values-develop.yaml` ensures Kubernetes uses these local images instead of trying to pull from a registry.

## 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### Access the UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open **https://localhost:8080** — log in with:

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" \| base64 -d` |

### Login with the CLI

```bash
argocd login localhost:8080 \
  --username admin \
  --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)" \
  --insecure
```

## 4. Deploy via ArgoCD (GitOps)

Register the applications so ArgoCD watches this repo + the app repo's chart:

```bash
kubectl apply -f apps/medical-appointment.yaml
```

ArgoCD will now (for development):
1. Clone the **app repo** at `main` for the chart (`deploy/charts/medical-appointment`)
2. Merge the env override from `envs/development/values.yaml` (this repo, `main`)
3. Sync resources into the `medical-appointment-development` namespace
4. Keep polling — any new commit auto-syncs (development)

Production follows the same flow but its deploy PR is gated for manual
approval before merge.

Watch it happen:

```bash
argocd app get medical-appointment-development --watch
```

## 5. Access the application

```bash
# Route the domain to minikube's IP
echo "$(minikube ip) medical-appointment.local" | sudo tee -a /etc/hosts

# Test the backend
curl http://medical-appointment.local/api/health

# Open the frontend
open http://medical-appointment.local
```

If ingress isn't resolving, fall back to port-forward:

```bash
kubectl port-forward svc/backend  8000:80 -n medical-appointment-dev &
kubectl port-forward svc/frontend 5173:80 -n medical-appointment-dev &
```

---

## Development workflows

### Change application code (backend / frontend)

```bash
cd ../medical-appointment

# 1. Edit source code
vim backend/src/app/main.py

# 2. Rebuild the image inside minikube
eval $(minikube docker-env)
docker build -t medical-appointment-backend:local ./backend

# 3. Restart the pod so it picks up the new image
kubectl rollout restart deployment/backend -n medical-appointment-dev
kubectl rollout status deployment/backend -n medical-appointment-dev
```

Same pattern for the frontend — just swap `backend` → `frontend`.

### Change Kubernetes config (Helm values / templates)

Chart templates now live in the **app repo** at
`deploy/charts/medical-appointment`. Per-env values live here under `envs/`.

```bash
# 1. Edit an env value override (this repo)
vim envs/development/values.yaml

# 2. Commit and push
git add -A
git commit -m "feat: increase backend memory limit"
git push origin main

# 3. ArgoCD auto-syncs within ~3 min, or trigger immediately:
argocd app sync medical-appointment-development
```

Because `syncPolicy.automated.selfHeal: true`, any manual `kubectl edit` you do will also be reverted to match Git.

### Test a Helm render locally (no cluster needed)

```bash
helm template medical-appointment ../medical-appointment/deploy/charts/medical-appointment \
  --values ../medical-appointment/deploy/charts/medical-appointment/values-develop.yaml
```

### Diff what a values change would apply

```bash
helm diff upgrade medical-appointment ../medical-appointment/deploy/charts/medical-appointment \
  --namespace medical-appointment-dev \
  --values ../medical-appointment/deploy/charts/medical-appointment/values-develop.yaml
```

(Requires the `helm-diff` plugin: `helm plugin install https://github.com/databus23/helm-diff`)

---

## Database

PostgreSQL runs as a StatefulSet with a PersistentVolumeClaim. Init scripts (schema + seed) are baked into a ConfigMap mounted at `/docker-entrypoint-initdb.d`.

```bash
# Connect
kubectl exec -it statefulset/postgres -n medical-appointment-dev -- \
  psql -U app -d medical_appointment

# Check tables
\dt

# Reset everything (destructive)
kubectl delete pvc data-postgres-0 -n medical-appointment-dev
kubectl rollout restart statefulset/postgres -n medical-appointment-dev
```

---

## Useful commands

| What | Command |
|------|---------|
| List pods | `kubectl get pods -n medical-appointment-dev` |
| Backend logs | `kubectl logs -f deployment/backend -n medical-appointment-dev` |
| Frontend logs | `kubectl logs -f deployment/frontend -n medical-appointment-dev` |
| DB logs | `kubectl logs -f statefulset/postgres -n medical-appointment-dev` |
| Resource usage | `kubectl top pods -n medical-appointment-dev` |
| Shell into backend | `kubectl exec -it deployment/backend -n medical-appointment-dev -- /bin/bash` |
| ArgoCD app status | `argocd app get medical-appointment-dev` |
| Force sync | `argocd app sync medical-appointment-dev --force` |
| Helm release values | `helm get values medical-appointment -n medical-appointment-dev` |

---

## Troubleshooting

### ImagePullBackOff / ErrImagePull

Minikube can't find the image locally. Rebuild it:

```bash
eval $(minikube docker-env)
docker build -t medical-appointment-backend:local  ../medical-appointment/backend
docker build -t medical-appointment-frontend:local ../medical-appointment/frontend
```

### CrashLoopBackOff

```bash
kubectl logs <pod-name> -n medical-appointment-dev
kubectl describe pod <pod-name> -n medical-appointment-dev
```

### ArgoCD shows "Out of Sync"

```bash
argocd app diff medical-appointment-dev   # see what drifted
argocd app sync medical-appointment-dev   # force sync
```

### Ingress not working

```bash
kubectl get ingress -n medical-appointment-dev
kubectl describe ingress -n medical-appointment-dev
minikube addons enable ingress   # make sure the addon is on
```

---

## Cleanup

```bash
# Remove the app
argocd app delete medical-appointment-dev --cascade=true

# Remove ArgoCD
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd

# Remove /etc/hosts entry
sudo sed -i '' '/medical-appointment.local/d' /etc/hosts

# Stop or delete minikube
minikube stop        # pause
minikube delete      # full reset
```

---

## Two-environment model

| | develop | production |
|---|---|---|
| Branch | `develop` | `main` |
| Namespace | `medical-appointment-dev` | `medical-appointment-production` |
| Values | `values-develop.yaml` | `values-production.yaml` |
| Images | Local (`:local`, `Never`) | GHCR (`:1.0.0`, `Always`) |
| HPA | Off | On (2–5 replicas) |
| TLS | None | cert-manager + Let's Encrypt |

To promote to production, merge `develop` → `main` and update `values-production.yaml` with the new image tag. ArgoCD syncs automatically.