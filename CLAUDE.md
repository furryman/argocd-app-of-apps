# CLAUDE.md

Guidance for Claude Code working in this repository.

This repo is the **parent Helm chart** that declares ArgoCD `Application` resources for the fuhriman.org cluster. ArgoCD installs this chart at first boot from `user_data.sh` in the terraform repo, then reconciles each child Application continuously from [`eks-helm-charts`](https://github.com/furryman/eks-helm-charts).

The pattern is **App-of-Apps**: one Application (this chart, installed via the `argocd-apps` chart from `user_data.sh`) creates four child Applications, one per directory in `eks-helm-charts`.

## Commands

There's no build, no tests — this is a single Helm chart of 4 templates plus a values file. Useful commands:

```bash
helm lint .                       # Lint the chart
helm template . | less            # Render the four Applications locally for review
```

You never `helm install` this chart yourself; the EC2 instance's `user_data.sh` does that on first boot via `helm install ... argocd-apps argo/argocd-apps`. After the cluster is up, edits land via:

1. Commit + push to `main`.
2. ArgoCD reconciles this chart (the top-level `app-of-apps` Application).
3. Child Applications update, each pulling from `eks-helm-charts`.

## Repository layout

```text
argocd-app-of-apps/
├── Chart.yaml                   # version: 1.0.0 — App-of-Apps wrapper, no dependencies
├── values.yaml                  # repoURL + per-app enable flags + namespaces
└── templates/
    ├── cert-manager.yaml        # wave -2
    ├── envoy-gateway.yaml       # wave -1
    ├── external-dns.yaml        # wave  0
    └── fuhriman-website.yaml    # wave  0
```

Each template is the same shape: a single `argoproj.io/v1alpha1` `Application` CR gated on `.Values.apps.<name>.enabled`, pointing at the matching subdirectory in `eks-helm-charts` (`path: cert-manager`, etc.).

## Application contract

Every Application in this repo:

- Has finalizer `resources-finalizer.argocd.argoproj.io` so child resources clean up on delete.
- Has `syncPolicy.automated: {prune: true, selfHeal: true}` — manual cluster edits get reverted; resources removed from git get pruned from the cluster.
- Has `syncOptions: [CreateNamespace=true]` so namespaces don't need to be pre-created.
- Uses `ServerSideApply=true` on cert-manager and envoy-gateway because their CRDs + control-plane resources benefit from SSA semantics (avoids `metadata.managedFields` churn on every reconcile).
- Points at `repoURL = https://github.com/furryman/eks-helm-charts.git` and `targetRevision = HEAD` from `values.yaml`. Both are configurable.

## Sync waves

ArgoCD's `sync-wave` annotation orders the children when they're all created together (e.g. on a fresh cluster). Numbers run low to high:

| Wave | App | Why |
|---|---|---|
| `-2` | `cert-manager` | Provides CRDs + the `letsencrypt-prod` ClusterIssuer that envoy-gateway's Certificate depends on. |
| `-1` | `envoy-gateway` | Provides Gateway API CRDs + the shared `public` Gateway + the multi-SAN Certificate + the ArgoCD HTTPRoute. Must precede anything that creates an HTTPRoute. |
| `0`  | `external-dns` | Polls for HTTPRoutes continuously, so order vs the workloads it watches is not load-bearing. |
| `0`  | `fuhriman-website` | The Next.js portfolio. Its HTTPRoute attaches to the Gateway from wave `-1`. |

If you add a new Application that creates HTTPRoutes or Certificates, give it wave `0` or later. If it provides CRDs anything else depends on, give it a wave less than its consumers'.

## Values

```yaml
spec:
  destination:
    server: https://kubernetes.default.svc          # in-cluster only
  source:
    repoURL: https://github.com/furryman/eks-helm-charts.git
    targetRevision: HEAD                            # bump to a tag for prod stability

apps:
  certManager:     {enabled: true, namespace: cert-manager}
  envoyGateway:    {enabled: true, namespace: envoy-gateway-system}
  externalDns:     {enabled: true, namespace: external-dns}
  fuhrimanWebsite: {enabled: true, namespace: default}
```

- `repoURL` is shared — every Application reads it from `.Values.spec.source.repoURL`. Forking `eks-helm-charts` requires updating only this one key.
- `targetRevision: HEAD` is convenient for a portfolio cluster; for a production setup you'd pin a tag and update it deliberately.
- `apps.<name>.enabled: false` is a clean way to take a component out of the cluster without deleting the manifest. (Remember `prune: true` — disabling will delete the workload's resources.)

## Bootstrap path

How this chart gets onto a brand-new cluster:

1. Terraform launches the EC2 instance from the Packer-built AMI.
2. `user_data.sh` (in the terraform repo) installs k3s, then ArgoCD via the chart 9.5.15 `argo/argo-cd` Helm release with `configs.params.server.insecure=true` (so TLS terminates at the Gateway, not at argocd-server).
3. `user_data.sh` installs the `argocd-apps` chart (`argo/argocd-apps` 1.6.2) with values pointing at this repo. That creates the parent `app-of-apps` Application.
4. ArgoCD reads this repo's `templates/` and creates the four child Applications.
5. Children sync charts from `eks-helm-charts` in wave order. End-to-end cold start: ~3 min from EC2 launch to a healthy cluster.

A recovery-bootstrap Application manifest for hand-applying when ArgoCD is broken is in the `README.md`.

## Sibling repos

- **[`furryman/terraform`](https://github.com/furryman/terraform)** — AWS infra; installs ArgoCD + this chart via `user_data.sh`. Look here for cluster credentials, EIP, SSM access.
- **[`furryman/eks-helm-charts`](https://github.com/furryman/eks-helm-charts)** — All four child chart sources. The `argocd-server` HTTPRoute lives in *that* repo (under `envoy-gateway/templates/`), not here, because it needs the Gateway API CRDs that the envoy-gateway chart installs.
- **[`furryman/fuhriman-website`](https://github.com/furryman/fuhriman-website)** — The Next.js portfolio. Not referenced from this chart directly; it ships its container image to Docker Hub and bumps the tag in `eks-helm-charts/fuhriman-chart/values.yaml`, which `fuhriman-website.yaml` here points at.

## When working in this repo

**Adding a new Application**:

1. Add `<appKey>: {enabled, namespace}` to `values.yaml`'s `apps:` block.
2. Add `templates/<app-name>.yaml` modeled on one of the four existing templates.
3. Pick a sync wave that respects CRD/issuer dependencies (-2 for CRDs, -1 for cluster-singleton infra, 0 for normal workloads).
4. Create the matching chart in `eks-helm-charts/<app-name>/` so `path: <app-name>` resolves.
5. Push. `app-of-apps` reconciles → the new Application is created → it reconciles its chart.

**Gotchas**:

- `prune: true` is enabled cluster-wide. If you rename a resource in a child chart, the old one will be deleted on the next sync — make sure that's intentional.
- `selfHeal: true` reverts manual cluster edits. If you `kubectl edit` a managed resource for debugging, expect ArgoCD to undo it within a minute or so. Disable temporarily by setting `spec.syncPolicy.automated.selfHeal: false` on the specific Application; remember to flip back.
- The `argocd-server` HTTPRoute lives in `eks-helm-charts/envoy-gateway/templates/`, not here. Don't migrate it: it depends on Gateway API CRDs that wave `-1` installs.
- `targetRevision: HEAD` means every push to `eks-helm-charts` reconciles. If you're doing a risky chart bump, consider pinning a tag temporarily.
- Don't add `ServerSideApply=true` blindly. It's set for cert-manager and envoy-gateway because of CRD/Gateway specifics. For normal workloads (external-dns, fuhriman-website), default client-side apply is fine and avoids SSA conflict noise.
