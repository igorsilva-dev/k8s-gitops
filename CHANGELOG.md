# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html) for milestone releases. Day-to-day commits land on `main` continuously; milestone tags (`v0.1.0`, `v0.2.0`, etc.) mark phase completions of the [Portfolio Roadmap](https://github.com/igorsilva-dev). Every Argo CD-managed change here is reconciled by the `root-app` Application created by [civo-infrastructure](https://github.com/igorsilva-dev/civo-infrastructure).

## [Unreleased]

Phase 2 work begins after `v0.1.0`: Istio Application + values, kube-prometheus-stack Application + values, Grafana dashboards committed as raw manifests.

### Added

- **Tailscale operator (EDGE-001)** in `applications/tailscale-operator.yaml`: chart `tailscale-operator 1.104.2` from `https://pkgs.tailscale.com/helmcharts`, namespace `tailscale`, sync-wave `1` (after the Doppler `ClusterSecretStore`). The `operator-oauth` Secret is an ExternalSecret from Doppler (`TAILSCALE_OAUTH_CLIENT_ID` / `TAILSCALE_OAUTH_CLIENT_SECRET`), synced at wave `-1` inside the Application so the operator starts with credentials; nothing sensitive goes into git or Helm values. The operator joins the tailnet as `tailscale-operator` (`tag:k8s-operator`); the proxies it creates are tagged `tag:k8s`. First step of the private edge: nothing is exposed yet (that's EDGE-002).

### Changed

- **Argo CD chart `7.7.10` → `10.11.6` (Argo CD v2.13 → v3.5.6) for Kubernetes 1.35.** On the `1.35.0-k3s1` cluster, Argo CD 2.13 could not diff Deployments (`.status.terminatingReplicas: field not declared in schema`), leaving `argocd`, `crashdummy` and `external-secrets` sync `Unknown`. Argo CD 3.5 is tested against Kubernetes 1.33–1.36. Bumped together with the Terraform bootstrap in civo-infrastructure, so both pin the same chart. `argocd/values.yaml` needs no changes: every key still exists in chart 10, and `configs.params` overrides (`server.insecure`, `rootpath`, `basehref`) keep working after chart 9 dropped their defaults. The Application already syncs with `ServerSideApply=true`, which Argo CD 3.3+ requires for self-managed upgrades (the ApplicationSet CRD exceeds the client-side apply annotation limit).

## [0.1.0] — 2026-05-18

**Phase 1 — GitOps source of truth established.** First milestone release. Argo CD self-manages from this repo, External Secrets Operator is installed and wired to Doppler, and the workflow for adding new workloads + rotating secrets is documented.

### Added

- **Repository scaffold** (CIVO-005 paired work): `README.md`, `.gitignore`, and an empty `applications/` directory ready to be discovered by Argo CD's `root-app` Application (created by [civo-infrastructure](https://github.com/igorsilva-dev/civo-infrastructure)).
- **Argo CD self-managed** (CIVO-006): `applications/argocd.yaml` declares a multi-source Application — chart `argo-cd 7.7.10` from `https://argoproj.github.io/argo-helm` plus values from this repo via `$values/argocd/values.yaml`. `releaseName: argocd` matches the Terraform-installed Helm release so Argo CD adopts the live resources without churn. `argocd/values.yaml` mirrors what Terraform applied at install (`server.insecure`, single-replica controller with metrics, `redis-ha` / `dex` / `notifications` disabled).
- **External Secrets Operator** (CIVO-007): `applications/external-secrets.yaml` installs the `external-secrets 2.4.1` chart at sync-wave `-2`. `applications/external-secrets-config.yaml` applies the runtime config at sync-wave `0`, with retry backoff to handle CRD-propagation delays. `external-secrets/values.yaml` for the chart; `external-secrets/config/cluster-secret-store.yaml` for the Doppler `ClusterSecretStore` (project `k8s-codeup`, config `dev`); `external-secrets/config/sandbox-demo-secret.yaml` for a demo `ExternalSecret` pulling `TEST_SECRET` into a `sandbox/sandbox-demo` Kubernetes Secret.
- **Documentation pass** (CIVO-008): README expanded with the full current layout, an "Adding a new workload" worked example (kube-prometheus-stack), the sync-wave conventions table, an ExternalSecret rotation template, and the doubled-URL `kubectl proxy` workaround for accessing the Argo CD UI on K3s 1.34. Cross-links to the `civo-infrastructure` ADRs.

### Fixed

- **ESO chart 2.4.1 doesn't serve `external-secrets.io/v1beta1`** (PR #3, commit `c4e0716`): the chart ships v1 GA CRDs only; `v1beta1.served = false`. Bumped `apiVersion` on `ClusterSecretStore` and `ExternalSecret` to `external-secrets.io/v1`. Validated with `kubectl apply --dry-run=server` against the live cluster.
- **Argo CD UI returned 404 through `kubectl proxy`** (PR #4, commit `24d6e95`): Argo CD's SPA mounts at `/` by default and builds `/api/...` URLs accordingly, which the kube-apiserver service proxy then 404s for. Added `server.rootpath` and `server.basehref` to `argocd/values.yaml` so Argo CD generates URLs prefixed with the proxy path. UI works via the resulting doubled URL (apiserver-proxy prefix + Argo CD rootpath, since the apiserver strips one). Removed when Phase 2 puts Istio + ingress in front.

### Notes

- **Sync wave conventions**: `-2` for operators whose CRDs others consume (ESO chart); `0` for default Applications including Argo CD's self-management and ESO's runtime config. Documented in `README.md`.
- **Argo CD `automated.retry` doesn't re-evaluate against HEAD**: discovered during the ESO `v1beta1 → v1` fix. The retry replays the original failed operation; pushing a fix doesn't unstick the loop. Delete the Application — `root-app` recreates it from HEAD within seconds. Documented in [civo-infrastructure/docs/architecture.md](https://github.com/igorsilva-dev/civo-infrastructure/blob/main/docs/architecture.md#argo-cd-automatedretry-doesnt-re-evaluate-against-head).
- **Pairs with [civo-infrastructure `v0.1.0`](https://github.com/igorsilva-dev/civo-infrastructure/releases/tag/v0.1.0)** and [tf-modules `v0.1.0`](https://github.com/igorsilva-dev/tf-modules/releases/tag/v0.1.0).

[Unreleased]: https://github.com/igorsilva-dev/k8s-gitops/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/igorsilva-dev/k8s-gitops/releases/tag/v0.1.0
