# k8s-deploy-demo

[![CI](https://github.com/hermes-93/k8s-deploy-demo/actions/workflows/ci.yml/badge.svg)](https://github.com/hermes-93/k8s-deploy-demo/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A production-ready Helm chart that deploys [cicd-pipeline-demo](https://github.com/hermes-93/cicd-pipeline-demo) — a FastAPI service — to Kubernetes with multi-environment configuration, horizontal pod autoscaling, health probes, and a full CI validation pipeline.

> Part of the [DevOps portfolio](https://github.com/hermes-93) series.

---

## Architecture

```
k8s-deploy-demo/
└── charts/
    └── cicd-pipeline-demo/
        ├── Chart.yaml            # Chart metadata & version
        ├── values.yaml           # Default values (production baseline)
        ├── values-dev.yaml       # Dev overrides (1 replica, debug logging)
        ├── values-staging.yaml   # Staging overrides (2 replicas, HPA, ingress)
        ├── values-prod.yaml      # Prod overrides (3+ replicas, TLS ingress)
        └── templates/
            ├── deployment.yaml   # Deployment with security context
            ├── service.yaml      # ClusterIP service
            ├── configmap.yaml    # App env vars (rolled on change)
            ├── hpa.yaml          # HorizontalPodAutoscaler (optional)
            ├── ingress.yaml      # Ingress (optional)
            └── serviceaccount.yaml
```

## CI Pipeline

| Stage | Tool | What it checks |
|-------|------|---------------|
| Helm lint | `helm lint` | Chart syntax, all 4 value sets |
| Schema validation | kubeconform | Rendered manifests against k8s 1.29 & 1.31 schemas |
| Smoke test | kind + helm | Real `helm install` in a local cluster |

---

## Quick start

### Prerequisites
- Helm 3.x
- A Kubernetes cluster (local: [kind](https://kind.sigs.k8s.io/), [minikube](https://minikube.sigs.k8s.io/))

### Deploy

```bash
# Dev (single replica, debug logging)
helm install demo charts/cicd-pipeline-demo \
  -f charts/cicd-pipeline-demo/values-dev.yaml \
  --namespace dev --create-namespace

# Staging (HPA enabled, nginx ingress)
helm install demo charts/cicd-pipeline-demo \
  -f charts/cicd-pipeline-demo/values-staging.yaml \
  --namespace staging --create-namespace

# Production (TLS, 3+ replicas, stricter resources)
helm install demo charts/cicd-pipeline-demo \
  -f charts/cicd-pipeline-demo/values-prod.yaml \
  --namespace prod --create-namespace
```

### Access the app

```bash
# Port-forward for local testing
kubectl port-forward -n dev svc/demo-cicd-pipeline-demo 8080:80
curl http://localhost:8080/health
```

### Upgrade

```bash
helm upgrade demo charts/cicd-pipeline-demo \
  -f charts/cicd-pipeline-demo/values-dev.yaml \
  --namespace dev
```

---

## Key design decisions

| Decision | Reason |
|----------|--------|
| `readOnlyRootFilesystem: true` | Immutable container FS — prevents runtime tampering |
| `runAsNonRoot: true` | Follows least-privilege principle |
| `checksum/config` annotation | Pods restart automatically on ConfigMap change |
| HPA on CPU 60–70% | Balances cost vs latency under load |
| Separate liveness / readiness probes | Liveness (`/health/live`) is lightweight; readiness (`/health`) validates full service |

---

## Docker image

The chart deploys `ghcr.io/hermes-93/cicd-pipeline-demo`.  
Source: [hermes-93/cicd-pipeline-demo](https://github.com/hermes-93/cicd-pipeline-demo)
