# Medical Appointment Config

GitOps configuration repository for the medical-appointment system using ArgoCD + Helm.

## 📋 Overview

This repository contains Kubernetes manifests and Helm charts for deploying the medical appointment booking system across different environments (develop, production) using GitOps principles with ArgoCD.

## 🏗️ Repository Structure

```
medical-appointment-config/
├── charts/
│   └── medical-appointment/          # Helm chart
│       ├── Chart.yaml                # Chart metadata
│       ├── values.yaml               # Default values
│       ├── values-develop.yaml       # Development overrides
│       ├── values-production.yaml    # Production overrides
│       └── templates/                # Kubernetes templates
│           ├── backend/              # Backend deployment
│           ├── frontend/             # Frontend deployment
│           ├── postgres/             # PostgreSQL database
│           ├── ingress/              # Ingress configuration
│           ├── serviceaccount.yaml   # Service account
│           ├── _helpers.tpl          # Helm helpers
│           └── NOTES.txt             # Installation notes
├── apps/
│   ├── develop/
│   │   └── medical-appointment.yaml  # ArgoCD app for dev
│   └── production/
│       └── medical-appointment.yaml  # ArgoCD app for prod
├── secrets/                          # Encrypted secrets (SOPS)
└── docs/                             # Documentation
```

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
   # Create develop namespace
   kubectl create namespace medical-appointment-develop

   # Install chart locally
   cd ../medical-appointment-config
   helm install medical-appointment ./charts/medical-appointment \
     --namespace medical-appointment-develop \
     --values charts/medical-appointment/values-develop.yaml
   ```

6. **Access Application:**
   ```bash
   # Add to /etc/hosts
   echo "$(minikube ip) medical-appointment.local" | sudo tee -a /etc/hosts

   # Access at http://medical-appointment.local
   ```

### Production Deployment

1. **Build and Push Images:**
   ```bash
   cd ../medical-appointment/backend
   docker build -t ghcr.io/sbillion/medical-appointment-backend:1.0.0 .
   docker push ghcr.io/sbillion/medical-appointment-backend:1.0.0

   cd ../frontend
   docker build -t ghcr.io/sbillion/medical-appointment-frontend:1.0.0 .
   docker push ghcr.io/sbillion/medical-appointment-frontend:1.0.0
   ```

2. **Update Chart Values:**
   ```bash
   # Edit values-production.yaml with new image tags
   ```

3. **Deploy via ArgoCD:**
   ```bash
   # Apply ArgoCD application manifest
   kubectl apply -f apps/production/medical-appointment.yaml

   # Monitor sync
   argocd app get medical-appointment-production
   argocd app sync medical-appointment-production
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
kubectl get pods -n medical-appointment-develop
kubectl logs -f deployment/backend -n medical-appointment-develop
kubectl logs -f deployment/frontend -n medical-appointment-develop
```

### Debug ArgoCD
```bash
argocd app get medical-appointment-develop
argocd app logs medical-appointment-develop
argocd app sync medical-appointment-develop --force
```

### Database Issues
```bash
kubectl exec -it statefulset/postgres -n medical-appointment-develop -- psql -U app -d medical_appointment
```

### Port Forwarding
```bash
# Backend
kubectl port-forward svc/backend 8000:80 -n medical-appointment-develop

# Frontend
kubectl port-forward svc/frontend 5173:80 -n medical-appointment-develop
```

## 📊 Monitoring

### Health Checks
- **Backend:** `/api/health` endpoint
- **Frontend:** TCP port 5173
- **PostgreSQL:** `pg_isready` command

### Logs
```bash
# All pods
kubectl logs -f -n medical-appointment-develop --all-containers=true

# Specific component
kubectl logs -f deployment/backend -n medical-appointment-develop
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
helm uninstall medical-appointment -n medical-appointment-develop

# Delete namespace
kubectl delete namespace medical-appointment-develop

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