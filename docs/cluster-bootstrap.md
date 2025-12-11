# Cluster Bootstrap Guide

This guide explains how to restore the entire ForkFlow platform from scratch using GitOps. All infrastructure, applications, and configuration are stored in Git, allowing complete cluster recovery in minutes.

## Prerequisites

- Kubernetes cluster (Docker Desktop, k3s, or similar)
- `kubectl` CLI installed and configured
- Access to this repository: https://github.com/Platform-Chronicles/forkflow-gitops.git

## Architecture Overview

The platform uses a three-tier namespace architecture:

- **argocd**: Control plane namespace (ArgoCD + platform-gateway)
- **observability**: Monitoring stack (Jaeger, Prometheus, Grafana)
- **forkflow-local**: Application services (catalog, order, fulfillment)

All services are exposed through a single shared Gateway API gateway with hostname-based routing.

## Bootstrap Steps

### 1. Install ArgoCD

ArgoCD is our GitOps controller that manages all other applications.

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment -n argocd argocd-server
```

**Expected result**: ArgoCD pods running in the `argocd` namespace.

### 2. Install Gateway API CRDs

Gateway API provides the standard for ingress routing.

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

**Expected result**: Gateway API CRDs installed (GatewayClass, Gateway, HTTPRoute, etc.)

### 3. Install NGINX Gateway Fabric

NGINX Gateway Fabric is our Gateway API implementation.

```bash
# Install NGINX Gateway Fabric CRDs
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/crds.yaml

# Install NGINX Gateway Fabric
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/default/deploy.yaml
```

**Expected result**:
- `nginx-gateway` namespace created
- NGINX Gateway Fabric deployment running
- GatewayClass `nginx` created

### 4. Deploy All ArgoCD Applications

This single command deploys all applications defined in the GitOps repository.

```bash
# Clone the GitOps repository (if not already cloned)
git clone https://github.com/Platform-Chronicles/forkflow-gitops.git
cd forkflow-gitops

# Apply all ArgoCD Application manifests
kubectl apply -f apps/
```

**What this deploys**:
- `namespaces`: Creates observability and forkflow-local namespaces
- `platform-gateway`: Shared Gateway with 4 listeners (argocd, jaeger, prometheus, grafana)
- `observability-routes`: HTTPRoutes for observability tools
- `jaeger`: Distributed tracing system
- `prometheus`: Metrics collection
- `grafana`: Visualization dashboards
- `catalog-service`: Product catalog microservice (2 replicas)
- `order-service`: Order management microservice (2 replicas)
- `fulfillment-service`: Fulfillment microservice (2 replicas)

### 5. Verify Deployment

Wait for all applications to sync and become healthy:

```bash
# Check ArgoCD Applications status
kubectl get applications -n argocd -o custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status

# Check all pods are running
kubectl get pods --all-namespaces

# Check Gateway status
kubectl get gateway platform-gateway -n argocd

# Check HTTPRoutes
kubectl get httproute -n argocd
kubectl get httproute -n observability
```

**Expected result**: All applications show as `Synced` and `Healthy`.

## Accessing Services

### Option 1: Port-Forward (Recommended for Local Development)

Access services directly via kubectl port-forward:

```bash
# ArgoCD UI
kubectl port-forward -n argocd svc/argocd-server 8080:443
# Visit: https://localhost:8080
# Username: admin
# Password: kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# Jaeger UI
kubectl port-forward -n observability svc/jaeger 16686:16686
# Visit: http://localhost:16686

# Prometheus UI
kubectl port-forward -n observability svc/prometheus 9090:9090
# Visit: http://localhost:9090

# Grafana UI
kubectl port-forward -n observability svc/grafana 3000:3000
# Visit: http://localhost:3000
# Default credentials: admin/admin

# Catalog Service API
kubectl port-forward -n forkflow-local svc/catalog-service 8001:80
# Visit: http://localhost:8001

# Order Service API
kubectl port-forward -n forkflow-local svc/order-service 8002:80
# Visit: http://localhost:8002

# Fulfillment Service API
kubectl port-forward -n forkflow-local svc/fulfillment-service 8003:80
# Visit: http://localhost:8003
```

### Option 2: Gateway API with Hostname Routing

If your LoadBalancer implementation supports it (may require Docker Desktop restart):

```bash
# Get Gateway LoadBalancer IP
kubectl get gateway platform-gateway -n argocd -o jsonpath='{.status.addresses[0].value}'
# Should return: 172.18.0.5
```

Add to `/etc/hosts` (requires sudo):

```
172.18.0.5 argocd.local
172.18.0.5 jaeger.local
172.18.0.5 prometheus.local
172.18.0.5 grafana.local
```

Then access:
- http://argocd.local
- http://jaeger.local
- http://prometheus.local
- http://grafana.local

**Note**: Docker Desktop's LoadBalancer implementation can be unreliable. If this doesn't work, use Option 1 (port-forward) instead.

## Recovery Time

**Total time from empty cluster to fully operational**: ~2-3 minutes

This includes:
- ArgoCD installation: ~30 seconds
- Gateway Fabric installation: ~15 seconds
- Application deployment and sync: ~60-90 seconds

## Architecture Details

### Gateway Configuration

The `platform-gateway` Gateway has 4 listeners with hostname-based routing:

1. **argocd listener**: Routes `argocd.local` to `argocd-server` (same namespace)
2. **jaeger listener**: Routes `jaeger.local` to `jaeger:16686` (observability namespace)
3. **prometheus listener**: Routes `prometheus.local` to `prometheus:9090` (observability namespace)
4. **grafana listener**: Routes `grafana.local` to `grafana:3000` (observability namespace)

The Gateway uses `allowedRoutes` configuration to enable cross-namespace routing:
- argocd listener: Only accepts routes from `argocd` namespace
- observability listeners: Accept routes from `observability` namespace using namespace selector

### Directory Structure

```
forkflow-gitops/
├── apps/                           # ArgoCD Application manifests
│   ├── namespaces.yaml
│   ├── platform-gateway.yaml
│   ├── observability-routes.yaml
│   ├── jaeger.yaml
│   ├── prometheus.yaml
│   ├── grafana.yaml
│   ├── catalog-service.yaml
│   ├── order-service.yaml
│   └── fulfillment-service.yaml
├── base/                           # Base Kubernetes resources
│   ├── namespaces/
│   ├── platform-gateway/
│   ├── observability-routes/
│   ├── jaeger/
│   ├── prometheus/
│   ├── grafana/
│   ├── catalog-service/
│   ├── order-service/
│   └── fulfillment-service/
└── overlays/                       # Environment-specific configs
    └── local/
        ├── namespaces/
        ├── platform-gateway/
        ├── observability-routes/
        ├── jaeger/
        ├── prometheus/
        ├── grafana/
        ├── catalog-service/
        ├── order-service/
        └── fulfillment-service/
```

## Troubleshooting

### ArgoCD Applications stuck in "Progressing"

```bash
# Check ArgoCD sync status
kubectl get applications -n argocd

# View detailed application status
kubectl describe application <app-name> -n argocd

# Manually trigger sync if needed
kubectl patch application <app-name> -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
```

### Pods not starting

```bash
# Check pod status and events
kubectl describe pod <pod-name> -n <namespace>

# Check pod logs
kubectl logs <pod-name> -n <namespace>

# Check if images are being pulled
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Gateway not accepting routes

```bash
# Check Gateway status
kubectl describe gateway platform-gateway -n argocd

# Check if HTTPRoutes are accepted
kubectl describe httproute <route-name> -n <namespace>

# Look for "Accepted" condition with Status=True
# If Status=False, check the Reason and Message fields
```

### Gateway LoadBalancer stuck at `<pending>`

This is expected with Docker Desktop when multiple LoadBalancers are requested. The platform uses a single shared Gateway to avoid this issue.

```bash
# Verify only one LoadBalancer exists
kubectl get svc --all-namespaces | grep LoadBalancer

# Should only show: nginx-gateway service in nginx-gateway namespace
```

## Next Steps

After bootstrapping the cluster:

1. **Configure Grafana**: Import dashboards for Prometheus metrics
2. **Verify tracing**: Send test requests through services and view traces in Jaeger
3. **Test services**: Use the microservices APIs to create orders and verify functionality
4. **Explore ArgoCD**: Browse the UI to see the GitOps application structure

## Reference

- **ArgoCD Docs**: https://argo-cd.readthedocs.io/
- **Gateway API**: https://gateway-api.sigs.k8s.io/
- **NGINX Gateway Fabric**: https://docs.nginx.com/nginx-gateway-fabric/
- **GitOps Repository**: https://github.com/Platform-Chronicles/forkflow-gitops
