# ForkFlow GitOps

GitOps deployment configurations for the ForkFlow platform using ArgoCD and Kustomize.

## Overview

This repository contains the GitOps configurations for ForkFlow, demonstrating the evolution from manual kubectl deployments to automated, declarative GitOps workflows.

## Project Context

**Part of:** [Platform Chronicles](https://github.com/Platform-Chronicles)
**Narrative:** Chapter 2 - "The Rollback at 2am"
**Problem Solved:** 30% deployment failure rate → <5% failure rate, 2-hour deployments → 5-minute deployments

## Repository Structure

```
forkflow-gitops/
├── apps/
│   ├── base/                       # Base Kustomize configs
│   │   ├── menu-service/
│   │   ├── order-service/
│   │   └── kitchen-display/
│   ├── dev/                        # Dev overlays
│   ├── staging/                    # Staging overlays
│   └── production/                 # Production overlays
├── infrastructure/                 # Infrastructure apps (monitoring, etc)
├── environments/                   # Environment-level config
│   ├── dev/
│   │   └── apps.yaml              # App-of-apps for dev
│   ├── staging/
│   └── production/
└── argocd/
    ├── install/                    # ArgoCD installation manifests
    ├── config/                     # ArgoCD configuration
    └── applications/               # Application definitions
```

## Prerequisites

- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/getting_started/) >= 2.9
- [kubectl](https://kubernetes.io/docs/tasks/tools/) >= 1.28
- [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) >= 5.0
- Running Kubernetes cluster (see [forkflow-infrastructure](https://github.com/Platform-Chronicles/forkflow-infrastructure))

## Quick Start

### Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f argocd/install/install.yaml

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Deploy Applications

```bash
# Deploy dev environment
kubectl apply -f environments/dev/apps.yaml

# Check sync status
argocd app list
```

## Technologies

- **ArgoCD** - GitOps continuous delivery
- **Kustomize** - Kubernetes native configuration management
- **GitHub Actions** - CI/CD automation

## GitOps Workflow

```
Developer → Git Push → GitHub Actions → Build Image → Update GitOps Repo
                                                              ↓
                                                        ArgoCD Detects
                                                              ↓
                                                        Sync to Cluster
```

## Key Features

- ✅ Declarative deployments via Git
- ✅ Automatic sync to Kubernetes
- ✅ Easy rollbacks (git revert)
- ✅ Environment-specific configurations
- ✅ App-of-apps pattern
- ✅ Complete audit trail

## Sync Policies

- **Dev:** Auto-sync enabled, auto-prune, self-heal
- **Staging:** Auto-sync enabled, manual prune, self-heal
- **Production:** Manual sync (approval required), manual prune, self-heal

## Related Repositories

- [forkflow-monolith](https://github.com/Platform-Chronicles/forkflow-monolith) - Application code
- [forkflow-infrastructure](https://github.com/Platform-Chronicles/forkflow-infrastructure) - Infrastructure setup
- [forkflow-services](https://github.com/Platform-Chronicles/forkflow-services) - Microservices

## Business Impact

**Before:**
- Deployment time: 2 hours
- Deployment frequency: 5/week
- Failure rate: 30%
- Mean Time to Recovery: 3 hours

**After:**
- Deployment time: 5 minutes (96% reduction)
- Deployment frequency: 20+/week (4x increase)
- Failure rate: <5% (83% reduction)
- Mean Time to Recovery: 9 minutes (95% reduction)

**ROI:** 585% in first year

## Rollback Procedure

```bash
# Find the commit to rollback to
git log --oneline

# Rollback via git
git revert <commit-sha>
git push

# ArgoCD will automatically sync the rollback
```

## Documentation

See the [docs](./docs) directory for:
- Deployment workflows
- Rollback procedures
- Troubleshooting guides

## License

MIT
