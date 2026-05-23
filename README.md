# ArgoCD App of Apps

This repository contains the parent ArgoCD `Application` that declares the cluster workloads for fuhriman.org. ArgoCD installs the app-of-apps chart at first boot (via cloud-init / user_data), then reconciles each child Application continuously.

## Architecture

```text
                      ┌─────────────────────┐
                      │  ArgoCD App of Apps │   (this repo, helm chart)
                      └─────────────────────┘
                                  │
       ┌────────────────┬─────────┴────────┬─────────────────┐
       ▼                ▼                  ▼                 ▼
┌──────────────┐ ┌──────────────┐ ┌────────────────┐ ┌────────────────┐
│ cert-manager │ │envoy-gateway │ │  external-dns  │ │fuhriman-website│
│ (sync -2)    │ │ (sync -1)    │ │  (sync 0)      │ │  (sync 0)      │
└──────────────┘ └──────────────┘ └────────────────┘ └────────────────┘
```

All charts source from the [`eks-helm-charts`](https://github.com/furryman/eks-helm-charts) repo.

## Sync Waves

Applications deploy in order using ArgoCD's `sync-wave` annotation:

1. **cert-manager** (wave `-2`): TLS certificate issuance via Let's Encrypt + Gateway API solver. Must be ready before the Gateway needs a cert.
2. **envoy-gateway** (wave `-1`): Gateway API control + data plane. Also bundles the shared `public` Gateway resource, the multi-SAN `Certificate`, and the `argocd-server` HTTPRoute.
3. **external-dns** (wave `0`): publishes Route53 records from HTTPRoute hostnames.
4. **fuhriman-website** (wave `0`): the Next.js portfolio + its HTTPRoute attaching to the `public` Gateway.

## Auto-Sync

All Applications have automated sync enabled:

- `prune: true` — removes resources deleted from git
- `selfHeal: true` — reverts manual cluster changes back to declared state

## Deployment Details

ArgoCD itself is installed on the k3s cluster via cloud-init during EC2 instance launch. The bootstrap:

1. k3s starts on a Packer-baked AMI (~30 sec) — see [`furryman/terraform/packer`](https://github.com/furryman/terraform/tree/main/packer)
2. `user_data.sh` (~25 lines, runtime-only) installs ArgoCD chart 9.5.15 via Helm with `configs.params.server.insecure=true` (TLS terminates at the Gateway, not the server).
3. user_data installs the `argocd-apps` chart pointing at this repo → ArgoCD's app-of-apps Application is created.
4. ArgoCD reads this repo's `templates/` → spawns the four child Applications.
5. Child Applications sync charts from `eks-helm-charts`. End-to-end convergence: ~3 minutes.

See [the terraform repository](https://github.com/furryman/terraform) for the AWS-side infrastructure.

## Configuration

`values.yaml` drives which apps deploy and into which namespaces:

```yaml
apps:
  certManager:
    enabled: true
    namespace: cert-manager
  envoyGateway:
    enabled: true
    namespace: envoy-gateway-system
  externalDns:
    enabled: true
    namespace: external-dns
  fuhrimanWebsite:
    enabled: true
    namespace: default
```

The `templates/` directory has one Application manifest per app, gated by these flags.

## Manual Deployment (recovery)

If ArgoCD is broken and you need to bootstrap by hand:

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
