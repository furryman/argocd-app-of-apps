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

## Usage

This chart is automatically deployed by Terraform via the helm-argocd module. The parent Application is created pointing to this repository.

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
