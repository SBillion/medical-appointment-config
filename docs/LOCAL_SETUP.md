# Local Development Setup

This guide will help you set up the medical-appointment system locally using minikube, ArgoCD, and Helm.

## Prerequisites

```bash
# Install required tools (macOS)
brew install minikube kubectl helm argocd

# Verify installations
minikube version
kubectl version --client
helm version
argocd version
```

## Step 1: Start Minikube

```bash
# Start minikube with adequate resources
minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=20gb

# Enable ingress addon
minikube addons enable ingress

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

## Step 2: Build Docker Images

```bash
# Navigate to the main application repository
cd ../medical-appointment

# Set minikube docker environment
eval $(minikube docker-env)

# Build backend image
docker build -t medical-appointment-backend:local ./backend

# Build frontend image
docker build -t medical-appointment-frontend:local ./frontend

# Verify images
docker images | grep medical-appointment
```

## Step 3: Install ArgoCD

```bash
# Create argocd namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 in your browser.

**Initial Login:**
- Username: `admin`
- Password: Get it with:
  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  ```

## Step 4: Deploy Application

```bash
# Navigate to config repository
cd ../medical-appointment

# Create local dev namespace
kubectl create namespace medical-appointment-dev

# Install Helm chart (chart lives in the app repo now)
helm install medical-appointment deploy/charts/medical-appointment \
  --namespace medical-appointment-dev \
  --values deploy/charts/medical-appointment/values-develop.yaml \
  --debug

# Wait for pods to be ready
kubectl wait --for=condition=ready --timeout=300s pod -l app=backend -n medical-appointment-dev
kubectl wait --for=condition=ready --timeout=300s pod -l app=frontend -n medical-appointment-dev
kubectl wait --for=condition=ready --timeout=300s pod -l app=postgres -n medical-appointment-dev
```

## Step 5: Access Application

```bash
# Get minikube IP
MINIKUBE_IP=$(minikube ip)

# Add to /etc/hosts (requires sudo)
echo "$MINIKUBE_IP medical-appointment.local" | sudo tee -a /etc/hosts

# Access the application
open http://medical-appointment.local
```

## Step 6: Verify Deployment

```bash
# Check all pods
kubectl get pods -n medical-appointment-dev

# Check services
kubectl get svc -n medical-appointment-dev

# Check ingress
kubectl get ingress -n medical-appointment-dev

# View backend logs
kubectl logs -f deployment/backend -n medical-appointment-dev

# View frontend logs
kubectl logs -f deployment/frontend -n medical-appointment-dev
```

## Step 7: Test the Application

```bash
# Test backend health
curl http://medical-appointment.local/api/health

# Test API endpoints
curl http://medical-appointment.local/api/slots
```

## Development Workflow

### Making Changes to Backend

```bash
# 1. Make code changes in ../medical-appointment/backend

# 2. Rebuild backend image
cd ../medical-appointment
eval $(minikube docker-env)
docker build -t medical-appointment-backend:local ./backend

# 3. Restart backend deployment
kubectl rollout restart deployment/backend -n medical-appointment-dev

# 4. Watch the rollout
kubectl rollout status deployment/backend -n medical-appointment-dev
```

### Making Changes to Frontend

```bash
# 1. Make code changes in ../medical-appointment/frontend

# 2. Rebuild frontend image
cd ../medical-appointment
eval $(minikube docker-env)
docker build -t medical-appointment-frontend:local ./frontend

# 3. Restart frontend deployment
kubectl rollout restart deployment/frontend -n medical-appointment-dev

# 4. Watch the rollout
kubectl rollout status deployment/frontend -n medical-appointment-dev
```

### Updating Helm Values

```bash
# 1. Edit values file (chart lives in the app repo)
vim ../medical-appointment/deploy/charts/medical-appointment/values-develop.yaml

# 2. Upgrade Helm release
helm upgrade medical-appointment ../medical-appointment/deploy/charts/medical-appointment \
  --namespace medical-appointment-dev \
  --values ../medical-appointment/deploy/charts/medical-appointment/values-develop.yaml
```

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl describe pod <pod-name> -n medical-appointment-dev

# Check logs
kubectl logs <pod-name> -n medical-appointment-dev

# Check events
kubectl get events -n medical-appointment-dev --sort-by='.lastTimestamp'
```

### Database Connection Issues

```bash
# Check PostgreSQL pod
kubectl get pods -n medical-appointment-dev -l app=postgres

# Connect to PostgreSQL
kubectl exec -it statefulset/postgres -n medical-appointment-dev -- psql -U app -d medical_appointment

# Check database
\dt
```

### Image Pull Errors

```bash
# Verify images exist in minikube
eval $(minikube docker-env)
docker images | grep medical-appointment

# If images are missing, rebuild them
cd ../medical-appointment
docker build -t medical-appointment-backend:local ./backend
docker build -t medical-appointment-frontend:local ./frontend
```

### Ingress Issues

```bash
# Check ingress controller
kubectl get pods -n ingress-nginx

# Check ingress resources
kubectl describe ingress medical-appointment -n medical-appointment-dev

# Test DNS resolution
nslookup medical-appointment.local
```

## Cleanup

```bash
# Uninstall Helm release
helm uninstall medical-appointment -n medical-appointment-dev

# Delete namespace
kubectl delete namespace medical-appointment-dev

# Remove from /etc/hosts
sudo sed -i '/medical-appointment.local/d' /etc/hosts

# Stop ArgoCD
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Stop minikube
minikube stop

# Optional: Delete minikube cluster
minikube delete
```

## Next Steps

- Set up ArgoCD Application for automated GitOps
- Configure monitoring and logging
- Add network policies for security
- Set up CI/CD pipeline for image building
- Configure secret management (Sealed Secrets, External Secrets)

## Useful Commands

```bash
# Port forward to services
kubectl port-forward svc/backend 8000:80 -n medical-appointment-dev
kubectl port-forward svc/frontend 5173:80 -n medical-appointment-dev

# Exec into containers
kubectl exec -it deployment/backend -n medical-appointment-dev -- /bin/bash
kubectl exec -it statefulset/postgres -n medical-appointment-dev -- /bin/bash

# Get resource usage
kubectl top pods -n medical-appointment-dev
kubectl top nodes

# Get Helm release status
helm status medical-appointment -n medical-appointment-dev
helm get values medical-appointment -n medical-appointment-dev

# ArgoCD CLI (if configured)
argocd app get medical-appointment-dev
argocd app sync medical-appointment-dev
```