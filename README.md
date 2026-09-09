# FluxCD fleet monorepo

This repository **is** the fleet-infra repo. It holds cluster composition roots and the shared infrastructure/app pieces those roots assemble. It does not nest content under a `fleet-infra/` subdirectory.

Each cluster describes *what* it contains. Implementations live under `infrastructure/` and `apps/`, so you compose a new cluster from existing pieces instead of copying controllers or workloads into `clusters/<name>/`.

## Repository structure

```text
.
├── clusters/
│   ├── prod-eu-1/              # composition root for one production cluster
│   │   ├── flux-system/        # Flux bootstrap (gotk files appear after `flux bootstrap`)
│   │   ├── infrastructure.yaml # Flux Kustomizations: controllers → configs
│   │   ├── apps.yaml           # Flux Kustomization: applications
│   │   └── kustomization.yaml  # composes flux-system + the Flux Kustomizations above
│   └── staging-eu-1/
├── infrastructure/
│   ├── controllers/            # cluster addons (cert-manager, ingress, …)
│   └── configs/
│       ├── base/               # shared config
│       ├── production/         # production overlay (includes base)
│       └── staging/            # staging overlay (includes base)
└── apps/
    ├── overlays/
    │   ├── production/         # fleet-wide list of app production overlays
    │   └── staging/
    └── <app>/                  # add when you have a real workload
        ├── base/
        └── overlays/{production,staging}/
```

Environment, region, and provider are **config dimensions** under `infrastructure/configs/` and `apps/*/overlays/`. They are not the top-level cluster hierarchy.

## Prerequisites

- A Kubernetes cluster per fleet member (kubeconfig pointed at the cluster you bootstrap)
- [Flux CLI](https://fluxcd.io/flux/installation/) v2 (this skeleton uses `kustomize.toolkit.fluxcd.io/v1`)
- [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) for local `kustomize build` checks
- Permission to create a deploy key (or PAT) on `vikbaranov/gitops-flux-v2` for bootstrap

> [!NOTE]
> There is no `policies/` or `tenants/` tree yet. Add those layers only when you have real Kyverno/OPA policies or multi-tenant onboarding to place there.

## How a cluster is composed

Flux bootstraps a cluster by reconciling `clusters/<cluster>` (the `flux-system` GitRepository Kustomization created by bootstrap). That directory is a native Kustomize root: it includes `flux-system/` plus Flux `Kustomization` objects that pull in the rest of the fleet.

Reconciliation order uses `spec.dependsOn`:

```text
infra-controllers  →  infra-configs  →  apps
     (30m, wait)         (30m, wait)      (10m)
```

| Flux Kustomization | Path | Role |
| --- | --- | --- |
| `infra-controllers` | `./infrastructure/controllers` | Operators and CRDs |
| `infra-configs` | `./infrastructure/configs/<env>` | ClusterIssuers, storage, DNS, other config |
| `apps` | `./apps/overlays/<env>` | Workloads for that environment |

`prod-eu-1` uses the `production` overlays; `staging-eu-1` uses `staging`. Both clusters share the same controller catalog.

## Bootstrap

Install the operator into `flux-system`, see the [install guide](https://fluxoperator.dev/docs/guides/install/) :

```bash
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace
```

For a private repo, create the pull secret referenced by `FluxInstance.spec.sync.pullSecret`:

```bash
kubectl create secret generic github \
  --namespace=flux-system \
  --from-literal=username=git \
  --from-literal=password="${GITHUB_TOKEN}"
```

Create an age key file for SOPS:

```bash
age-keygen -o clusters/prod-k3s-proxmox/age.key
```

Create a secret:
```bash
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=sops.agekey="clusters/prod-k3s-proxmox/age.key"
```

Then apply a `FluxInstance` whose `spec.sync.path` is the cluster composition root:

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: FluxInstance
metadata:
  name: flux
  namespace: flux-system
spec:
  distribution:
    version: "2.x"
    registry: ghcr.io/fluxcd
  sync:
    kind: GitRepository
    url: https://github.com/vikbaranov/gitops-flux-v2
    ref: refs/heads/main
    path: clusters/prod-k3s-proxmox
    pullSecret: github
```

## Add infrastructure

**Controllers** (operators): add a subdirectory under `infrastructure/controllers/` with a `kustomization.yaml` (HelmRelease, namespace, HelmRepository as needed) and list it in `infrastructure/controllers/kustomization.yaml`. Every cluster picks it up on the next reconcile.

**Config** (CRs that need those operators): put shared objects in `infrastructure/configs/base/` and env-specific patches in `infrastructure/configs/production/` or `staging/`.

## Add an application

1. Create the app tree:

   ```text
   apps/<app>/base/kustomization.yaml
   apps/<app>/overlays/production/kustomization.yaml
   apps/<app>/overlays/staging/kustomization.yaml
   ```

2. Register the env overlay in the fleet composition:

   - `apps/overlays/production/kustomization.yaml` → `../../<app>/overlays/production`
   - `apps/overlays/staging/kustomization.yaml` → `../../<app>/overlays/staging`

Prefer empty-but-valid Kustomize bases and real HelmReleases/manifests over sample nginx Deployments.

## Split team app repos later

This monorepo can keep platform pieces while application ownership moves out:

1. Move `apps/<app>/` into a team repository (same `base/` + `overlays/` layout).
2. In this fleet repo, replace the in-repo overlay entry with a Flux `GitRepository` (or OCIRepository) plus a `Kustomization` whose `sourceRef` points at that repo and whose `path` is the env overlay.
3. Keep `dependsOn: infra-configs` so apps still wait for cluster addons.
4. Leave cluster composition roots in this repository; they remain the inventory of what each cluster runs.

You do not need ArtifactGenerator/ExternalArtifact for that split.

## Validate locally

```bash
kustomize build clusters/prod-eu-1
kustomize build clusters/staging-eu-1
kustomize build infrastructure/controllers
kustomize build infrastructure/configs/production
kustomize build infrastructure/configs/staging
kustomize build apps/overlays/production
kustomize build apps/overlays/staging
```

After bootstrap, `kustomize build clusters/<cluster>` also emits Flux controllers from `gotk-components.yaml`.
