# k8s-gitops

Argo CD's source of truth for the Civo K3s portfolio platform.

> Parent repos: [`tf-modules`](https://github.com/igorsilva-dev/tf-modules) (reusable Terraform), [`civo-infrastructure`](https://github.com/igorsilva-dev/civo-infrastructure) (provisions cluster + installs Argo CD + creates the `root-app` Application pointing here).

## What lives here

The declarative state of every workload Argo CD reconciles on the cluster: Application CRs, Helm chart values, and operator-runtime config. The cluster itself comes up via Terraform in `civo-infrastructure`; everything that runs on it is reconciled from this repo.

## Layout

```
k8s-gitops/
├── applications/            # Argo CD Application CRs (watched by root-app)
│   ├── argocd.yaml          # Argo CD self-managed (CIVO-006)
│   ├── external-secrets.yaml         # ESO Helm chart (CIVO-007)
│   └── external-secrets-config.yaml  # ClusterSecretStore + ExternalSecrets (CIVO-007)
├── argocd/
│   └── values.yaml          # Argo CD chart values, consumed by applications/argocd.yaml via $values
└── external-secrets/
    ├── values.yaml          # ESO chart values, consumed by applications/external-secrets.yaml via $values
    └── config/              # ESO runtime CRs (separate dir so the config Application can recurse cleanly without picking up values.yaml)
        ├── cluster-secret-store.yaml
        └── sandbox-demo-secret.yaml
```

## How a workload reaches the cluster

1. `civo-infrastructure` creates a single `root-app` Argo CD Application that watches `applications/` in this repo (via a Terraform `kubectl_manifest` resource — see [`civo-infrastructure/infrastructure/argocd_bootstrap.tf`](https://github.com/igorsilva-dev/civo-infrastructure/blob/main/infrastructure/argocd_bootstrap.tf)).
2. New Application CRs added under `applications/` are picked up by `root-app` on its next reconciliation cycle (a few minutes; force with `argocd app sync root-app` or the UI Refresh button).
3. Each Application points at its config — Helm chart from a remote repo + values from this repo (via `$values` multi-source ref), or a path in this repo containing raw manifests.
4. Argo CD reconciles automatically (`syncPolicy.automated` with `prune` + `selfHeal`).

## Adding a new workload

Example: install Prometheus stack from the prometheus-community Helm chart with custom values held here.

**Step 1.** Add chart values:
```
monitoring/values.yaml
```

**Step 2.** Add the Application CR under `applications/`:
```yaml
# applications/monitoring.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 65.0.0
      helm:
        releaseName: monitoring
        valueFiles:
          - $values/monitoring/values.yaml
    - repoURL: https://github.com/igorsilva-dev/k8s-gitops
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
      - CreateNamespace=true
```

**Step 3.** Commit, push, merge. `root-app` picks up the new Application within a few minutes; the chart installs.

For workloads that need cluster-side prerequisites (a new namespace with specific labels, RBAC, secrets, etc.), add them in `civo-infrastructure` first — anything provisioned by Terraform should be in the bootstrap repo, not here.

## Conventions

- **One Application per workload**, named after the workload, in `applications/`.
- **Per-workload subfolders** at the repo root hold Helm values, raw manifests, or both.
- **Multi-source Applications** for anything installed from an upstream Helm chart (chart from upstream repo + values from this repo via `$values` ref). Single-source when both chart and values are local.
- **`syncPolicy.automated` with `prune: true` and `selfHeal: true`** for everything. Drift is auto-recovered (portfolio cluster, no humans tweaking).
- **`syncOptions: ServerSideApply=true`** for anything with large CRDs (Argo CD, ESO, future Istio / Prometheus operator) to avoid client-side-apply annotation-size failures.

## Sync waves

`argocd.argoproj.io/sync-wave` annotation on the Application controls reconciliation order. Lower wave = earlier.

| Wave | Use case | Examples |
|---|---|---|
| `-2` | Operators whose CRDs others consume | `external-secrets` (ESO chart) |
| `-1` | Operators with cluster-wide effects, no consumers in repo yet | reserved (cert-manager, kyverno) |
| `0` | Default. Most workloads. | `argocd`, `external-secrets-config`, future monitoring / mesh |
| `+1` | Consumers that need everything else healthy first | future kagent agents |

Within a single Application the same annotation orders resources, but with weaker guarantees — prefer splitting CRDs and CRs that consume them into separate Applications at different waves (see the `external-secrets` / `external-secrets-config` split for the canonical example).

## Adding or rotating a secret (ESO + Doppler)

Secrets live in Doppler (`k8s-codeup` project, `dev` config) and are pulled into the cluster by the External Secrets Operator. Workflow:

1. Add or update the value in the Doppler dashboard.
2. Either reference it in an existing `ExternalSecret`, or create a new one under `external-secrets/config/` pointing at the Doppler key. Example template — pulls `MY_KEY` from Doppler into a Kubernetes Secret named `my-secret` in namespace `agents`:
   ```yaml
   apiVersion: external-secrets.io/v1
   kind: ExternalSecret
   metadata:
     name: my-secret
     namespace: agents
   spec:
     refreshInterval: 1h
     secretStoreRef:
       name: doppler
       kind: ClusterSecretStore
     target:
       name: my-secret
       creationPolicy: Owner
     data:
       - secretKey: MY_KEY
         remoteRef:
           key: MY_KEY
   ```
3. Commit + push. Argo CD reconciles the new ExternalSecret; ESO pulls from Doppler within `refreshInterval`.

For rotation: change the value in Doppler. ESO refreshes the Kubernetes Secret automatically (1h default, configurable per ExternalSecret). No git push needed for rotation.

Why ESO + Doppler over Sealed Secrets is in [`civo-infrastructure/docs/adr/0002-eso-over-sealed-secrets.md`](https://github.com/igorsilva-dev/civo-infrastructure/blob/main/docs/adr/0002-eso-over-sealed-secrets.md).

## Accessing the Argo CD UI

`kubectl port-forward` is broken on this cluster (K3s 1.34 + containerd 2.1.x CRI regression). Use `kubectl proxy` and the doubled URL (apiserver-proxy prefix + Argo CD's own rootpath, since the apiserver proxy strips the prefix on the way through):

```sh
kubectl proxy &
open "http://localhost:8001/api/v1/namespaces/argocd/services/http:argocd-server:http/proxy/api/v1/namespaces/argocd/services/http:argocd-server:http/proxy/"
```

Login: `admin` / value of `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`. The doubled URL goes away in Phase 2 once Istio + ingress lands.

## Related

- [`civo-infrastructure`](https://github.com/igorsilva-dev/civo-infrastructure) — cluster + Argo CD bootstrap, plus the Terraform-managed Doppler-token Secret that lets ESO authenticate. Architecture details in [`civo-infrastructure/docs/architecture.md`](https://github.com/igorsilva-dev/civo-infrastructure/blob/main/docs/architecture.md).
- [`tf-modules`](https://github.com/igorsilva-dev/tf-modules) — the reusable Terraform modules consumed by `civo-infrastructure`.
