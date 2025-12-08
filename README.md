# ForkFlow GitOps

GitOps deployment configurations using ArgoCD and Kustomize.

## Overview

**Part of:** [Platform Chronicles](https://github.com/Platform-Chronicles)
**Chapter:** 2 - "The Rollback at 2am"
**Technologies:** ArgoCD, Kustomize

Declarative deployment configurations for ForkFlow platform.

## Quick Start

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f argocd/install/install.yaml

# Deploy applications
kubectl apply -f environments/dev/apps.yaml
```

## License

MIT
