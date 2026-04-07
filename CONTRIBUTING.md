# Contributing

## Development setup

### Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Helm | ≥ 3.14 | `brew install helm` |
| kind | ≥ 0.22 | `brew install kind` |
| kubeconform | latest | `brew install kubeconform` |
| kubectl | ≥ 1.29 | `brew install kubectl` |

### Local cluster

```bash
# Create a single-node kind cluster
kind create cluster --name k8s-demo

# Verify
kubectl cluster-info --context kind-k8s-demo
```

## Running checks locally

```bash
# 1. Lint all environments
helm lint charts/cicd-pipeline-demo
helm lint charts/cicd-pipeline-demo -f charts/cicd-pipeline-demo/values-dev.yaml
helm lint charts/cicd-pipeline-demo -f charts/cicd-pipeline-demo/values-staging.yaml
helm lint charts/cicd-pipeline-demo -f charts/cicd-pipeline-demo/values-prod.yaml

# 2. Render and validate manifests
helm template demo charts/cicd-pipeline-demo \
  | kubeconform -strict -kubernetes-version 1.31.0 -summary -

# 3. Deploy to local kind cluster
helm install demo charts/cicd-pipeline-demo \
  -f charts/cicd-pipeline-demo/values-dev.yaml \
  --namespace dev --create-namespace

# 4. Security scan
checkov -d charts/ --framework helm
trivy config charts/cicd-pipeline-demo

# 5. Cleanup
helm uninstall demo -n dev
kind delete cluster --name k8s-demo
```

## Making changes

1. Fork the repo and create a feature branch: `git checkout -b feat/your-feature`
2. Edit the chart templates or values
3. Run the local checks above — all must pass
4. Bump `version` in `Chart.yaml` following [semver](https://semver.org/)
5. Open a PR — CI will run automatically

## Chart versioning

| Change type | Version bump | Example |
|-------------|-------------|---------|
| New feature / value | minor | `0.1.0` → `0.2.0` |
| Bug fix / tweak | patch | `0.1.0` → `0.1.1` |
| Breaking change | major | `0.1.0` → `1.0.0` |

`appVersion` tracks the deployed application version — update it when `image.tag` in `values.yaml` changes.
