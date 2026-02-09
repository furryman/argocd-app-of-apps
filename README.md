# ArgoCD App of Apps

This repository contains the parent ArgoCD Application that manages all child applications for the fuhriman.org infrastructure.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    ArgoCD App of Apps                        │
│                     (This Repository)                        │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
    ┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐
    │  cert-manager   │ │ingress-nginx│ │fuhriman-website │
    │  (sync-wave:-2) │ │(sync-wave:-1)│ │  (sync-wave:0)  │
    └─────────────────┘ └─────────────┘ └─────────────────┘
```

## Sync Waves

Applications are deployed in order using ArgoCD sync waves:

1. **cert-manager** (wave -2): Installs first to provide TLS certificates
2. **ingress-nginx** (wave -1): Installs second to provide ingress controller
3. **fuhriman-website** (wave 0): Installs last with working ingress and TLS

## Auto-Sync

All applications have automated sync enabled:
- `prune: true` - Removes resources deleted from Git
- `selfHeal: true` - Reverts manual cluster changes

## Deployment Details

ArgoCD itself is installed on the k3s cluster via cloud-init during EC2 instance launch. The bootstrap process:

1. k3s is installed on the EC2 instance (single-node Kubernetes)
2. ArgoCD is deployed via the official Helm chart (argo/argo-cd)
3. The argocd-apps Helm chart creates this App-of-Apps as the root Application
4. ArgoCD then syncs and deploys all child applications from this repository

See the [terraform repository](https://github.com/furryman/terraform) for the complete infrastructure code.

## Usage

This chart is automatically deployed by Terraform during k3s cluster initialization. The parent Application is created via the argocd-apps Helm chart in the EC2 instance's cloud-init user_data script, pointing to this repository.

## Manual Deployment

If needed, you can manually create the root Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/furryman/argocd-app-of-apps.git
    path: .
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Configuration

Edit `values.yaml` to enable/disable applications or change namespaces:

```yaml
apps:
  certManager:
    enabled: true
    namespace: cert-manager
  ingressNginx:
    enabled: true
    namespace: ingress-nginx
  fuhrimanWebsite:
    enabled: true
    namespace: default
```
