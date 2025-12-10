# ForkFlow GitOps Repository

GitOps repository for ForkFlow Platform microservices deployment.

## Structure

```
forkflow-gitops/
├── apps/                    # ArgoCD Application definitions
│   └── catalog-service.yaml
├── environments/            # Environment-specific manifests
│   ├── local/              # Local development (Docker Desktop K8s)
│   │   └── catalog-service/
│   ├── staging/            # Staging environment
│   └── production/         # Production environment
└── README.md
```

## GitOps Workflow

1. **Code changes** → Pushed to `forkflow-services` repo
2. **CI builds** → Docker image tagged and pushed
3. **Update manifests** → This repo updated with new image tag
4. **ArgoCD syncs** → Deploys to Kubernetes automatically

## ArgoCD Applications

Each microservice has an ArgoCD Application manifest in `apps/`.

ArgoCD watches this repository and automatically syncs changes to the cluster.

## Environments

- **local**: Docker Desktop Kubernetes for development
- **staging**: Pre-production testing environment
- **production**: Live production environment

## Deploying a Service

```bash
# Apply ArgoCD Application
kubectl apply -f apps/catalog-service.yaml

# ArgoCD will automatically sync and deploy
```

## Accessing ArgoCD

```bash
# Port-forward to ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login: https://localhost:8080
# Username: admin
# Password: (from kubectl -n argocd get secret argocd-initial-admin-secret)
```

## License

MIT
