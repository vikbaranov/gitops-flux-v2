# Agent guide

This is a Flux v2 fleet monorepo. Manifests in git are the desired state. Bootstrap and human docs: `README.md`.

Clusters: `prod-k3s-proxmox`, `staging-eu-1`. Overlay directories use those names, not `production` / `staging`.

## Layout

```text
apps/base/<app>                         # HelmRelease, OCIRepository, namespace
apps/<cluster>/<app>                    # HTTPRoute, PVC size, other cluster delta
infrastructure/controllers/base/<name>  # operator (namespace + HelmRelease + source)
infrastructure/controllers/<cluster>    # which operators this cluster runs
infrastructure/configs/base/<name>      # CRs that need those operators (Issuer, Gateway)
infrastructure/configs/<cluster>        # secrets, local-path, other cluster-only CRs
clusters/<cluster>                      # FluxInstance, flux-vars, Kustomization CRs only
```

Flux on a cluster reconciles `clusters/<cluster>` (thin: no workloads). That emits:

```text
infra-controllers  →  infra-configs  →  apps
./infrastructure/controllers/<cluster>
./infrastructure/configs/<cluster>
./apps/<cluster>
```

Do not point a Flux Kustomization at a path that contains another Flux Kustomization (no overlapping paths). Cluster `kustomization.yaml` must not list `infrastructure/` or `apps/` as resources.

## Invariants

- **Base has no cluster identity.** Domain, LB IP, storage class, node paths, Cloudflare tokens stay in overlay or `flux-vars`.
- **Versions live in the cluster overlay** (`OCIRepository.spec.ref.tag`, Helm `chart.spec.version`, image tags). Base has chart/source wiring without pins. No `pins.yaml`, no bundles.
- **Overlay subdir only if there is a delta** (including version pins). Otherwise list `../base/<name>` from the cluster kustomization. Empty include-only folders are wrong.
- **Patches are full YAML** (strategic merge), not JSON6902.
- **Helm charts use `chartRef` + `OCIRepository`**, not `HelmRepository` with `type: oci`.
- **`flux-vars` is cluster facts only** (`cluster_subdomain`, `cluster_lb_ip`, `storage_class`). App values belong in `apps/<cluster>/<app>/`.
- Do not add `tenants/`, `policies/`, bundles, or ArtifactGenerator unless there is a real second team, a real policy, or a real reconcile-isolation need.
- Do not commit `.env`, `age.key`, or plaintext secrets. SOPS: `.sops.yaml` covers `infrastructure/configs/**`. Encrypt `data` / `stringData` with age.

## Add an operator

1. `infrastructure/controllers/base/<name>/` — namespace, OCIRepository (no tag), HelmRelease (`chartRef`).
2. CRs → `infrastructure/configs/base/<name>/` with `${cluster_subdomain}` / `${cluster_lb_ip}` where needed.
3. Enable it: cluster overlay dir with version pin (and any Helm patches), listed from `infrastructure/controllers/<cluster>/kustomization.yaml`. If the only delta is versions, still use a local dir — do not pin on base.

## Add an app

1. `apps/base/<app>/` — namespace, OCIRepository/HelmRepository, HelmRelease, default PVC/CNPG if any (no chart/image pins).
2. `apps/<cluster>/<app>/` includes `../../base/<app>` plus version pins, cluster resources, and patches.
3. List `<app>` in `apps/<cluster>/kustomization.yaml`.

## Add a cluster

1. `clusters/<name>/` with FluxInstance (`spec.sync.path` = this dir), `flux-vars.yaml` (`reconcile.fluxcd.io/watch: Enabled`), `infrastructure.yaml`, `apps.yaml`.
2. Overlay roots: `infrastructure/controllers/<name>/`, `infrastructure/configs/<name>/`, `apps/<name>/`. Empty `resources: []` is OK until the cluster needs workloads; that file is the Flux path, not a stub component.

## Validate

After any layout or manifest change:

```bash
kustomize build clusters/prod-k3s-proxmox
kustomize build clusters/staging-eu-1
kustomize build infrastructure/controllers/prod-k3s-proxmox
kustomize build infrastructure/configs/prod-k3s-proxmox
kustomize build apps/prod-k3s-proxmox
kustomize build infrastructure/controllers/staging-eu-1
```

Do not invent chart versions. If a HelmRelease has no version today, leave it unpinned.
