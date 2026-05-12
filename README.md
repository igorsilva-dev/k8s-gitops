# k8s-gitops

Argo CD's source of truth for the Civo K3s portfolio platform.

> Parent repos: [`tf-modules`](https://github.com/igorsilva-dev/tf-modules) (reusable Terraform), [`civo-infrastructure`](https://github.com/igorsilva-dev/civo-infrastructure) (provisions cluster + installs Argo CD).

## What lives here

This repository holds the declarative state of every workload Argo CD manages on the cluster: Application CRs, Helm chart values, raw Kubernetes manifests, and supporting configuration. The cluster comes up via Terraform; everything that runs on it is reconciled from this repo.

## Layout

```
k8s-gitops/
├── applications/          # Argo CD Application CRs (watched by root-app)
├── argocd/                # Argo CD chart values (self-managed via applications/argocd.yaml)
└── external-secrets/      # External Secrets Operator config (ClusterSecretStore, sample ExternalSecret)
```

## How a workload reaches the cluster

1. `civo-infrastructure`'s Terraform creates a single `root-app` Argo CD Application that watches `applications/` in this repo.
2. New Application CRs added under `applications/` are picked up by `root-app` on its next sync.
3. Each Application points at its config (values file, manifest folder, or external chart) in this repo or upstream.
4. Argo CD reconciles automatically (`syncPolicy.automated` with prune + selfHeal).

## Adding a new workload

1. Create a new YAML file under `applications/` with an Argo CD `Application` CR.
2. Point its `spec.source` at the appropriate folder in this repo (or an upstream chart).
3. Commit and push to `main`. Argo CD picks it up.

For sub-charts, drop their values files under a dedicated folder (`argocd/`, `external-secrets/`, future `monitoring/`, etc.) and reference them from the Application CR.

## Conventions

- Application CRs land in `applications/`, one file per Application, named after the workload.
- Sub-folders per workload hold Helm values, raw manifests, or both.
- `syncPolicy.automated` with `prune: true` and `selfHeal: true` for everything (this is a portfolio cluster, drift is recovered automatically).
- Sync waves: ESO at `-1` (so other components can consume secrets), default `0` otherwise.

## Related

- [civo-infrastructure README](https://github.com/igorsilva-dev/civo-infrastructure) for how the cluster gets created and how Argo CD is bootstrapped.
