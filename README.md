# Medical Appointment Config

GitOps configuration repository for the medical-appointment system using ArgoCD + Helm.

## 📋 Overview

This repository contains Kubernetes manifests and environment value overrides for deploying the medical appointment booking system across environments (development, production) using GitOps principles with ArgoCD.

> **Note:** The Helm chart (templates + base `values.yaml`) now lives in the
> **application repository** at [`SBillion/medical-appointment`](https://github.com/SBillion/medical-appointment)
> under `deploy/charts/medical-appointment`. ArgoCD renders the chart from
> that repo (multi-source) and applies the per-env values from this repo.

## 🏗️ Repository Structure

```
medical-appointment-config/
├── envs/
│   ├── development/
│   │   └── values.yaml               # Development overrides + image tag (CI-bumped)
│   └── production/
│       └── values.yaml                # Production overrides + image tag (CI-bumped)
├── apps/
│   └── medical-appointment.yaml       # ApplicationSet (generates dev + prod apps)
├── secrets/                          # Encrypted secrets (SOPS)
└── docs/                             # Documentation
```

## 🔄 Auto-Sync Configuration

This repository is configured with **automatic synchronization** enabled. When you push changes to the config repository, ArgoCD will automatically update your Kubernetes cluster.

### How Auto-Sync Works

1. **Continuous Polling**: ArgoCD polls the Git repository every 3 minutes
2. **Webhook Triggers**: GitHub Actions workflow triggers immediate sync on push
3. **Self-Healing**: ArgoCD automatically corrects drift between Git and cluster state

### Triggering Syncs

**Automatic:**
- A PR merged into `main` of the app repo triggers its `Release` workflow,
  which builds images (tagged with the commit SHA) and opens/updates one
  stacked deploy PR per environment on this repo (branch `deploy/<env>-pending`).
- **development** deploy PR is fast-tracked (auto-merged) → ArgoCD auto-syncs
  `medical-appointment-development`.
- **production** deploy PR requires manual approval → once merged, ArgoCD
  auto-syncs `medical-appointment-production`.

**Manual:**
```bash
# Sync specific application
argocd app sync medical-appointment-development

# Sync with force
argocd app sync medical-appointment-development --force

# Watch sync progress
argocd app watch medical-appointment-development
```

### Sync Configuration

Both environments are configured with:
- **Prune**: Automatically removes resources not in Git
- **Self-Heal**: Auto-corrects manual changes
- **Retry**: Up to 5 attempts with exponential backoff

See [docs/AUTO_SYNC.md](docs/AUTO_SYNC.md) for detailed sync configuration and troubleshooting.

## 🚀 Quick Start

### Prerequisites

- Kubernetes cluster (minikube, kind, or cloud provider)
- kubectl configured
- helm installed
- argocd CLI installed

### Local Development with Minikube

1. **Start Minikube:**
   ```bash
   minikube start --driver=docker --cpus=4 --memory=8192
   minikube addons enable ingress
   ```

2. **Install ArgoCD:**
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

3. **Access ArgoCD UI:**
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   # Open https://localhost:8080
   # Get password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
   ```

4. **Build Local Docker Images:**
   ```bash
   cd ../medical-appointment
   eval $(minikube docker-env)
   docker build -t medical-appointment-backend:local ./backend
   docker build -t medical-appointment-frontend:local ./frontend
   ```

5. **Deploy Application:**
   ```bash
   # Create local dev namespace
   kubectl create namespace medical-appointment-dev

   # Install chart locally (chart lives in the app repo now)
   cd ../medical-appointment
   helm install medical-appointment deploy/charts/medical-appointment \
     --namespace medical-appointment-dev \
     --values deploy/charts/medical-appointment/values-develop.yaml
   ```

6. **Access Application:**
   ```bash
   # Add to /etc/hosts
   echo "$(minikube ip) medical-appointment.local" | sudo tee -a /etc/hosts

   # Access at http://medical-appointment.local
   ```

### Deployments (development / production)

Image builds, pushes, and value updates are now automated by the `Release`
workflow in the **app repository**. When a PR merges into `main` there:

1. Backend + frontend images are pushed to GHCR, tagged with the merge commit SHA.
2. Two stacked deploy PRs are opened (or updated) on this repo:
   - `deploy/development-pending` → `main` (auto-merged, fast-tracked)
   - `deploy/production-pending` → `main` (manual approval gate)
3. Each deploy PR body lists the app PRs that will be deployed (with a
   hyperlink + checkbox) and is re-pointed to the latest SHA on `main` until
   it is merged.
4. Once merged, ArgoCD auto-syncs the matching Application.

**Manual (first-time setup):**
```bash
# Apply ArgoCD application manifests
kubectl apply -f apps/medical-appointment.yaml

# Monitor sync
argocd app get medical-appointment-development
argocd app get medical-appointment-production
```

## 🔧 Configuration

### Environment Variables

**Backend:**
- `DATABASE_URL`: PostgreSQL connection string
- `CORS_ORIGINS`: Allowed CORS origins

**Frontend:**
- `VITE_API_BASE_URL`: Backend API URL

**PostgreSQL:**
- `POSTGRES_USER`: Database username
- `POSTGRES_PASSWORD`: Database password
- `POSTGRES_DB`: Database name

### Helm Values

Key configurable parameters:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.backend.repository` | Backend image repository | `ghcr.io/sbillion/medical-appointment-backend` |
| `image.backend.tag` | Backend image tag | `latest` |
| `image.frontend.repository` | Frontend image repository | `ghcr.io/sbillion/medical-appointment-frontend` |
| `image.frontend.tag` | Frontend image tag | `latest` |
| `postgres.enabled` | Enable PostgreSQL deployment | `true` |
| `postgres.postgresqlDatabase` | Database name | `medical_appointment` |
| `ingress.enabled` | Enable ingress | `true` |
| `autoscaling.enabled` | Enable HPA | `false` |
| `resources.backend` | Backend resource limits/requests | See values.yaml |
| `resources.frontend` | Frontend resource limits/requests | See values.yaml |

## 🔄 GitOps Workflow

1. **Development:**
   - Make changes to application code
   - Build and push Docker images
   - Update `values-develop.yaml` with new image tags
   - Commit and push to `develop` branch
   - ArgoCD automatically syncs changes

2. **Production:**
   - Create release branch from main
   - Update `values-production.yaml` with new image tags
   - Create pull request to main
   - After merge, ArgoCD syncs to production

## 🛠️ Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n medical-appointment-dev
kubectl logs -f deployment/backend -n medical-appointment-dev
kubectl logs -f deployment/frontend -n medical-appointment-dev
```

### Debug ArgoCD
```bash
argocd app get medical-appointment-development
argocd app logs medical-appointment-development
argocd app sync medical-appointment-development --force
```

### Database Issues
```bash
kubectl exec -it statefulset/postgres -n medical-appointment-dev -- psql -U app -d medical_appointment
```

### Port Forwarding
```bash
# Backend
kubectl port-forward svc/backend 8000:80 -n medical-appointment-dev

# Frontend
kubectl port-forward svc/frontend 5173:80 -n medical-appointment-dev
```

## 📊 Monitoring

### Health Checks
- **Backend:** `/api/health` endpoint
- **Frontend:** TCP port 5173
- **PostgreSQL:** `pg_isready` command

### Logs
```bash
# All pods
kubectl logs -f -n medical-appointment-dev --all-containers=true

# Specific component
kubectl logs -f deployment/backend -n medical-appointment-dev
```

## 🔐 Security

### Secrets Management
- PostgreSQL credentials stored in Kubernetes Secrets
- For production, consider using:
  - Sealed Secrets
  - External Secrets Operator
  - Vault Operator

### Network Policies
Consider adding network policies to restrict traffic:
- Frontend → Backend only
- Backend → PostgreSQL only
- Deny all other traffic

## 🧹 Cleanup

```bash
# Uninstall Helm release
helm uninstall medical-appointment -n medical-appointment-dev

# Delete namespace
kubectl delete namespace medical-appointment-dev

# Remove ArgoCD
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Stop minikube
minikube stop
```

## 📚 Additional Resources

- [ArgoCD Documentation](https://argoproj.github.io/argo-cd/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Medical Appointment App Repository](https://github.com/SBillion/medical-appointment)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## 📝 License

This project is licensed under the MIT License - see the main application repository for details.