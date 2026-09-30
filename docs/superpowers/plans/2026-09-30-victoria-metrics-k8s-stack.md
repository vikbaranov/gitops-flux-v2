# VictoriaMetrics K8s Stack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a minimal prod-only monitoring stack using `victoria-metrics-k8s-stack`.

**Architecture:** Add the chart as an infrastructure controller with a reusable base and a `prod-k3s-proxmox` overlay. The base owns the namespace, OCI chart source, HelmRelease, and non-cluster-specific minimal values. The prod overlay pins chart version `0.95.0`, enables internal routes, disables Grafana auth, and exposes Grafana plus VictoriaMetrics UI through the existing Envoy Gateway.

**Tech Stack:** Flux HelmRelease `helm.toolkit.fluxcd.io/v2`, Flux OCIRepository `source.toolkit.fluxcd.io/v1`, Kustomize strategic merge patches, Gateway API HTTPRoute, VictoriaMetrics chart `victoria-metrics-k8s-stack` `0.95.0`.

---

## File Structure

- Create `infrastructure/controllers/base/victoria-metrics-k8s-stack/namespace.yaml`: monitoring namespace.
- Create `infrastructure/controllers/base/victoria-metrics-k8s-stack/oci-repository.yaml`: unpinned OCI chart source at `oci://ghcr.io/victoriametrics/helm-charts/victoria-metrics-k8s-stack`.
- Create `infrastructure/controllers/base/victoria-metrics-k8s-stack/helm-release.yaml`: base HelmRelease with chartRef, remediation, drift detection, and minimal non-cluster-specific values.
- Create `infrastructure/controllers/base/victoria-metrics-k8s-stack/kustomization.yaml`: base resource list.
- Create `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/kustomization.yaml`: prod overlay including base, patches, and Grafana HTTPRoute.
- Create `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/oci-repository.yaml`: prod chart tag pin `0.95.0`.
- Create `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/helm-release.yaml`: prod-only route and Grafana auth values.
- Create `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/http-route-grafana.yaml`: route for `grafana.${cluster_subdomain}`.
- Modify `infrastructure/controllers/prod-k3s-proxmox/kustomization.yaml`: enable the prod monitoring overlay.

## Task 1: Add Base Monitoring Controller

**Files:**
- Create: `infrastructure/controllers/base/victoria-metrics-k8s-stack/namespace.yaml`
- Create: `infrastructure/controllers/base/victoria-metrics-k8s-stack/oci-repository.yaml`
- Create: `infrastructure/controllers/base/victoria-metrics-k8s-stack/helm-release.yaml`
- Create: `infrastructure/controllers/base/victoria-metrics-k8s-stack/kustomization.yaml`

- [ ] **Step 1: Create the base namespace**

Create `infrastructure/controllers/base/victoria-metrics-k8s-stack/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

- [ ] **Step 2: Create the unpinned OCI chart source**

Create `infrastructure/controllers/base/victoria-metrics-k8s-stack/oci-repository.yaml`:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: victoria-metrics-k8s-stack
  namespace: monitoring
spec:
  interval: 24h
  url: oci://ghcr.io/victoriametrics/helm-charts/victoria-metrics-k8s-stack
  layerSelector:
    mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
    operation: copy
```

- [ ] **Step 3: Create the base HelmRelease**

Create `infrastructure/controllers/base/victoria-metrics-k8s-stack/helm-release.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: victoria-metrics-k8s-stack
  namespace: monitoring
spec:
  interval: 15m
  timeout: 10m
  chartRef:
    kind: OCIRepository
    name: victoria-metrics-k8s-stack
  install:
    crds: Create
    remediation:
      retries: 3
  upgrade:
    crds: CreateReplace
    remediation:
      retries: 3
  driftDetection:
    mode: enabled
  values:
    defaultRules:
      create: false
    alertmanager:
      enabled: false
    vmalert:
      enabled: false
```

- [ ] **Step 4: Create the base kustomization**

Create `infrastructure/controllers/base/victoria-metrics-k8s-stack/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - oci-repository.yaml
  - helm-release.yaml
```

- [ ] **Step 5: Validate the base render**

Run:

```bash
kustomize build infrastructure/controllers/base/victoria-metrics-k8s-stack
```

Expected: command exits `0` and renders `Namespace/monitoring`, `OCIRepository/victoria-metrics-k8s-stack`, and `HelmRelease/victoria-metrics-k8s-stack`.

- [ ] **Step 6: Commit the base controller**

Run:

```bash
git add infrastructure/controllers/base/victoria-metrics-k8s-stack
git commit -m "feat: add victoria metrics stack base"
```

Expected: commit succeeds with only the new base files staged.

## Task 2: Add Prod Overlay and Internal Routes

**Files:**
- Create: `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/kustomization.yaml`
- Create: `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/oci-repository.yaml`
- Create: `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/helm-release.yaml`
- Create: `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/http-route-grafana.yaml`

- [ ] **Step 1: Create the prod chart version pin**

Create `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/oci-repository.yaml`:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: victoria-metrics-k8s-stack
  namespace: monitoring
spec:
  ref:
    tag: 0.95.0
```

- [ ] **Step 2: Create the prod HelmRelease patch**

Create `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/helm-release.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: victoria-metrics-k8s-stack
  namespace: monitoring
spec:
  values:
    grafana:
      grafana.ini:
        auth:
          disable_login_form: true
        auth.anonymous:
          enabled: true
          org_role: Admin
    vmsingle:
      route:
        enabled: true
        parentRefs:
          - name: internal
            namespace: envoy-gateway-system
        hostnames:
          - vm.${cluster_subdomain}
```

- [ ] **Step 3: Create the Grafana HTTPRoute**

Create `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/http-route-grafana.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
  namespace: monitoring
spec:
  parentRefs:
    - name: internal
      namespace: envoy-gateway-system
  hostnames:
    - grafana.${cluster_subdomain}
  rules:
    - backendRefs:
        - name: victoria-metrics-k8s-stack-grafana
          port: 80
```

- [ ] **Step 4: Create the prod overlay kustomization**

Create `infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/victoria-metrics-k8s-stack
  - http-route-grafana.yaml
patches:
  - path: helm-release.yaml
  - path: oci-repository.yaml
```

- [ ] **Step 5: Validate the overlay render**

Run:

```bash
kustomize build infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack
```

Expected: command exits `0`, includes `HTTPRoute/grafana`, includes the HelmRelease values for `grafana.${cluster_subdomain}` and `vm.${cluster_subdomain}`, and includes `OCIRepository.spec.ref.tag: 0.95.0`.

- [ ] **Step 6: Commit the prod overlay**

Run:

```bash
git add infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack
git commit -m "feat: add prod monitoring overlay"
```

Expected: commit succeeds with only the prod overlay files staged.

## Task 3: Enable Monitoring in Prod Controller Root

**Files:**
- Modify: `infrastructure/controllers/prod-k3s-proxmox/kustomization.yaml`

- [ ] **Step 1: Add the monitoring overlay to prod controllers**

Modify `infrastructure/controllers/prod-k3s-proxmox/kustomization.yaml` to exactly:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - cert-manager
  - cloudnative-pg
  - envoy-gateway
  - victoria-metrics-k8s-stack
  - ../base/topolvm
```

- [ ] **Step 2: Validate the prod controller root**

Run:

```bash
kustomize build infrastructure/controllers/prod-k3s-proxmox
```

Expected: command exits `0` and includes `HelmRelease/victoria-metrics-k8s-stack`, `OCIRepository/victoria-metrics-k8s-stack`, and `HTTPRoute/grafana`.

- [ ] **Step 3: Commit the root enablement**

Run:

```bash
git add infrastructure/controllers/prod-k3s-proxmox/kustomization.yaml
git commit -m "feat: enable prod monitoring stack"
```

Expected: commit succeeds with only the prod controller root change staged.

## Task 4: Run Fleet Validation

**Files:**
- Verify: repository manifests only

- [ ] **Step 1: Validate prod cluster composition**

Run:

```bash
kustomize build clusters/prod-k3s-proxmox
```

Expected: command exits `0`.

- [ ] **Step 2: Validate staging cluster composition is unchanged**

Run:

```bash
kustomize build clusters/staging-eu-1
```

Expected: command exits `0` and does not include VictoriaMetrics resources.

- [ ] **Step 3: Validate prod infrastructure controllers**

Run:

```bash
kustomize build infrastructure/controllers/prod-k3s-proxmox
```

Expected: command exits `0`.

- [ ] **Step 4: Validate prod infrastructure configs**

Run:

```bash
kustomize build infrastructure/configs/prod-k3s-proxmox
```

Expected: command exits `0`.

- [ ] **Step 5: Validate prod apps**

Run:

```bash
kustomize build apps/prod-k3s-proxmox
```

Expected: command exits `0`.

- [ ] **Step 6: Validate staging infrastructure controllers**

Run:

```bash
kustomize build infrastructure/controllers/staging-eu-1
```

Expected: command exits `0` and does not include VictoriaMetrics resources.

- [ ] **Step 7: Inspect final diff**

Run:

```bash
git status --short
git diff --stat HEAD~3..HEAD
```

Expected: the only implementation changes are the monitoring base, prod monitoring overlay, and prod controller root enablement.
