# k8s-deploy-demo

[![CI](https://github.com/hermes-93/k8s-deploy-demo/actions/workflows/ci.yml/badge.svg)](https://github.com/hermes-93/k8s-deploy-demo/actions/workflows/ci.yml)
[![Security](https://github.com/hermes-93/k8s-deploy-demo/actions/workflows/security.yml/badge.svg)](https://github.com/hermes-93/k8s-deploy-demo/actions/workflows/security.yml)
[![Release](https://github.com/hermes-93/k8s-deploy-demo/actions/workflows/release.yml/badge.svg)](https://github.com/hermes-93/k8s-deploy-demo/actions/workflows/release.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A production-ready Helm chart that deploys [cicd-pipeline-demo](https://github.com/hermes-93/cicd-pipeline-demo) — a FastAPI service — to Kubernetes with multi-environment configuration, horizontal pod autoscaling, health probes, and a full CI validation pipeline.

> Part of the [DevOps portfolio](https://github.com/hermes-93) series.

---

## Kubernetes resource topology

```
                    ┌───────────────────────────────────────┐
                    │           Kubernetes Cluster          │
                    │                                       │
  External ──► Ingress ──► Service ──► Deployment           │
  Traffic      (optional)  (ClusterIP)    │                 │
                    │                     ├── Pod           │
                    │    HPA ─────────────┤   ├─ container  │
                    │  (optional)         └── Pod           │
                    │                         │             │
                    │  ConfigMap ─────────────┘             │
                    │  ServiceAccount ────────┘             │
                    └───────────────────────────────────────┘
```

### Resource summary

| Resource | Purpose |
|----------|---------|
| `Deployment` | Manages pod replicas with rolling update strategy |
| `Service` | ClusterIP — stable internal DNS for the app |
| `ConfigMap` | Injects env vars; pod restarts on change via checksum annotation |
| `HPA` | Scales pods between min/max replicas based on CPU utilization |
| `Ingress` | Optional external access with TLS termination |
| `ServiceAccount` | Least-privilege identity for pods |

---

## CI pipeline

```
push / PR
    │
    ▼
helm lint ──► kubeconform ──► kind smoke test
(4 envs)     (4 envs ×        (helm install
              k8s 1.29+1.31)   in real cluster)
```

| Stage | Tool | What it checks |
|-------|------|---------------|
| Helm lint | `helm lint` | Chart syntax, all 4 value sets |
| Schema validation | kubeconform | Rendered manifests vs k8s 1.29 & 1.31 schemas |
| Smoke test | kind + helm | `helm install` in a live local cluster |
| Misconfiguration scan | Checkov | CIS Kubernetes benchmark, 150+ security rules |
| Chart vuln scan | Trivy | CVE scan of rendered Kubernetes manifests |

---

## Quick start

### Prerequisites
- Helm 3.x — `brew install helm`
- A running cluster — [kind](https://kind.sigs.k8s.io/) or [minikube](https://minikube.sigs.k8s.io/)

### Deploy

```bash
# Dev (1 replica, debug logging, no ingress)
helm install demo charts/cicd-pipeline-demo \
  -f charts/cicd-pipeline-demo/values-dev.yaml \
  --namespace dev --create-namespace

# Staging (2 replicas, HPA, nginx ingress)
helm install demo charts/cicd-pipeline-demo \
  -f charts/cicd-pipeline-demo/values-staging.yaml \
  --namespace staging --create-namespace

# Production (3+ replicas, TLS, strict resources)
helm install demo charts/cicd-pipeline-demo \
  -f charts/cicd-pipeline-demo/values-prod.yaml \
  --namespace prod --create-namespace
```

### Access the app

```bash
kubectl port-forward -n dev svc/demo-cicd-pipeline-demo 8080:80
curl http://localhost:8080/health
# {"status":"healthy","service":"cicd-pipeline-demo", ...}
```

### Upgrade

```bash
helm upgrade demo charts/cicd-pipeline-demo \
  -f charts/cicd-pipeline-demo/values-dev.yaml \
  --namespace dev
```

### Install from OCI registry

```bash
helm install demo oci://ghcr.io/hermes-93/charts/cicd-pipeline-demo \
  --version 0.1.0 \
  --namespace prod --create-namespace
```

---

## Environment comparison

| Feature      |  dev  | staging  |    prod    |
|--------------|-------|----------|------------|
| Replicas     |   1   |   2      |     3      |
| HPA          |  ✗✗  |   2-4    |    3–10    |
| Ingress      |  ✗✗  |    yes   | yes, + TLS |
| Log level    | DEBUG |   INFO   |   WARNING  |
| CPU limit    | 100m  |   200m   |    500m    |
| Memory limit | 64Mi  |   128Mi  |    256Mi   |

---

## Key design decisions

| Decision | Reason |
|--------------------------------------|--------------------------------------------------------------------------------|
| `readOnlyRootFilesystem: true`       |  Immutable container FS — prevents runtime tampering                           |
| `runAsNonRoot: true`                 | Follows least-privilege principle                                              |
| Drop ALL capabilities                | Minimal attack surface                                                         |
| `checksum/config` annotation         | Pods auto-restart on ConfigMap change — no manual rollout                      |
| Separate liveness / readiness probes | Liveness (`/health/live`) is cheap; readiness (`/health`) validates full state |
| HPA CPU threshold 60–70%             | Headroom before saturation; avoids premature scale-down                        |
| `autoscaling/v2` HPA                 | Supports multiple metrics (CPU + memory + custom)                              |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for development setup, testing, and PR guidelines.

---

## Docker image

The chart deploys `ghcr.io/hermes-93/cicd-pipeline-demo`.
Source: [hermes-93/cicd-pipeline-demo](https://github.com/hermes-93/cicd-pipeline-demo)
