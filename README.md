# FluxCD fleet monorepo

Layout follows the Flux [repository structure](https://fluxcd.io/flux/guides/repository-structure/) and [flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example): shared `apps/` and `infrastructure/`, with a thin `clusters/` dir that only holds Flux objects.

```text
.
├── apps/
│   ├── base/
│   │   └── immich/
│   ├── prod-k3s-proxmox/
│   │   └── immich/                 # HTTPRoute, PVC/CNPG patches
│   └── staging-eu-1/
├── infrastructure/
│   ├── controllers/
│   │   ├── base/
│   │   │   ├── cert-manager/
│   │   │   ├── cloudnative-pg/
│   │   │   ├── envoy-gateway/
│   │   │   └── topolvm/
│   │   ├── prod-k3s-proxmox/       # which operators + Helm patches
│   │   └── staging-eu-1/
│   └── configs/
│       ├── base/
│       │   ├── cert-manager/
│       │   └── envoy-gateway/
│       ├── prod-k3s-proxmox/       # Cloudflare token, local-path
│       └── staging-eu-1/
└── clusters/
    ├── prod-k3s-proxmox/           # FluxInstance, flux-vars, Kustomization CRs
    └── staging-eu-1/
```

Overlay dirs are named after the cluster, not `production` / `staging`. Chart and image tags live on the base `OCIRepository` and Helm values.

## How a cluster is composed

Flux reconciles `clusters/<cluster>` (FluxInstance `spec.sync.path`). That directory only emits Flux objects and `flux-vars`. Nested Flux Kustomizations pull the overlays:

```text
infra-controllers  →  infra-configs  →  apps
     (30m, wait)         (30m, wait)      (10m)
```

| Flux Kustomization | Path |
| --- | --- |
| `infra-controllers` | `./infrastructure/controllers/<cluster>` |
| `infra-configs` | `./infrastructure/configs/<cluster>` |
| `apps` | `./apps/<cluster>` |

`prod-k3s-proxmox` includes cert-manager, CNPG, Envoy Gateway, TopoLVM, cert-manager DNS patch, shared configs, Cloudflare token, and Immich. `staging-eu-1` includes cert-manager, CNPG, and Envoy Gateway.

## Prerequisites

- A Kubernetes cluster per fleet member
- [Flux CLI](https://fluxcd.io/flux/installation/) v2 (`kustomize.toolkit.fluxcd.io/v1`)
- [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) for local `kustomize build`
- Permission to create a deploy key (or PAT) on `vikbaranov/gitops-flux-v2`

> [!NOTE]
> There is no `policies/` or `tenants/` tree yet. Add those layers only when you have real Kyverno/OPA policies or multi-tenant onboarding.

## Bootstrap

Install the operator, see the [install guide](https://fluxoperator.dev/docs/guides/install/):

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

Create an age key and SOPS secret:

```bash
age-keygen -o clusters/prod-k3s-proxmox/age.key
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
  cluster:
    size: small
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

1. Operator base: `infrastructure/controllers/base/<name>/` (namespace, HelmRelease, OCIRepository). Pin the chart tag on the OCIRepository.
2. CR bases: `infrastructure/configs/base/<name>/`. Use `${cluster_subdomain}` / `${cluster_lb_ip}` from `flux-vars`.
3. List the operator in `infrastructure/controllers/<cluster>/kustomization.yaml` as `../base/<name>`. Add `infrastructure/controllers/<cluster>/<name>/` only when that cluster needs a patch. Same for configs.

## Add an application

1. Create `apps/base/<app>/` with the HelmRelease (and namespace/source). Pin chart and image tags there.
2. Add `apps/<cluster>/<app>/` with cluster resources and patches (HTTPRoute, PVC size).
3. List that directory in `apps/<cluster>/kustomization.yaml`.

## Split team app repos later

1. Move `apps/base/<app>/` into a team repository.
2. In this fleet repo, replace the include with a Flux `GitRepository` (or OCIRepository) plus a `Kustomization` whose `path` is the cluster overlay.
3. Keep `dependsOn: infra-configs`.
4. Leave cluster composition roots here.

## Validate locally

```bash
kustomize build clusters/prod-k3s-proxmox
kustomize build clusters/staging-eu-1
kustomize build infrastructure/controllers/prod-k3s-proxmox
kustomize build infrastructure/configs/prod-k3s-proxmox
kustomize build apps/prod-k3s-proxmox
kustomize build infrastructure/controllers/staging-eu-1
kustomize build infrastructure/configs/staging-eu-1
kustomize build apps/staging-eu-1
kustomize build infrastructure/controllers/base/cert-manager
kustomize build infrastructure/configs/base/cert-manager
kustomize build apps/base/immich
```
