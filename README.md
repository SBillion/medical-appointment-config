# Medical Appointment Config

GitOps configuration repository for the medical-appointment system. ArgoCD
watches this repo and syncs cluster state automatically.

> The Helm chart lives in the **app repository** at
> [`SBillion/medical-appointment`](https://github.com/SBillion/medical-appointment)
> under `deploy/charts/medical-appointment`. ArgoCD renders it via
> multi-source (chart from app repo, env values from this repo).

## Repository Structure

```
medical-appointment-config/
├── envs/
│   ├── development/
│   │   └── values.yaml               # Dev overrides + image tag (CI-bumped)
│   └── production/
│       └── values.yaml                # Prod overrides + image tag (CI-bumped)
├── apps/
│   └── medical-appointment.yaml       # ApplicationSet (generates dev + prod apps)
├── secrets/                           # Encrypted secrets (SOPS)
└── docs/
    └── ARGOCD_SETUP.md                # ArgoCD admin guide (RBAC, SSO, tuning)
```

## GitOps Workflow

1. A PR merged into `main` on the app repo triggers its `Release` workflow.
2. The workflow builds images (SHA-tagged) and opens/updates stacked deploy
   PRs on this repo — one per environment.
3. Each deploy PR lists the app PRs being deployed and is re-pointed to the
   latest SHA on `main` until merged.
   - **development**: auto-merged (fast-tracked) → ArgoCD auto-syncs.
   - **production**: manual approval gate → ArgoCD auto-syncs after merge.

## Two-Environment Model

| | development | production |
|---|---|---|
| Branch | `main` | `main` |
| Namespace | `medical-appointment-development` | `medical-appointment-production` |
| Values | `envs/development/values.yaml` | `envs/production/values.yaml` |
| Images | GHCR (SHA-tagged, `Always`) | GHCR (SHA-tagged, `Always`) |
| HPA | Off | On (2–5 replicas) |
| TLS | None | cert-manager + Let's Encrypt |
| Deploy PR | Auto-merged | Manual approval |

## Auto-Sync

ArgoCD keeps the cluster in sync with Git via:
1. **Git Webhook** (optional): GitHub POSTs to ArgoCD's `/api/webhook` →
   instant refresh (see [docs/ARGOCD_SETUP.md](docs/ARGOCD_SETUP.md#git-webhook))
2. **Polling**: ArgoCD polls every 3 minutes by default
3. **Self-Healing**: auto-corrects drift between Git and cluster state

## Quick Start

See **[CONTRIBUTING.md](CONTRIBUTING.md)** for the complete setup guide
(cluster, ArgoCD, ApplicationSet, access). The only manual `kubectl apply`
is registering the ApplicationSet — ArgoCD handles everything else.

## Documentation

- [CONTRIBUTING.md](CONTRIBUTING.md) — Setup, development workflow, troubleshooting
- [docs/ARGOCD_SETUP.md](docs/ARGOCD_SETUP.md) — ArgoCD admin (RBAC, SSO, sync windows, performance)
- [Medical Appointment App Repository](https://github.com/SBillion/medical-appointment)
