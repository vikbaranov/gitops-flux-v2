# VictoriaMetrics K8s Stack Design

## Goal

Add a minimal monitoring stack to `prod-k3s-proxmox` using the `victoria-metrics-k8s-stack` Helm chart. The stack should follow the repository's Flux layout and keep cluster-specific versions and exposure details in the prod overlay.

## Scope

- Add the stack only to `prod-k3s-proxmox`.
- Treat the stack as infrastructure, not an application, because it provides cluster-level monitoring components.
- Expose Grafana and the VictoriaMetrics UI through the existing internal Envoy Gateway.
- Disable Grafana authentication as requested.
- Keep the install minimal and avoid adding alerting or notification components unless they are needed later.

## Repository Layout

Create a shared base at:

```text
infrastructure/controllers/base/victoria-metrics-k8s-stack/
```

The base will contain:

- `namespace.yaml` for the monitoring namespace.
- A chart source without a version pin.
- A `HelmRelease` using `chartRef`, matching the repository's OCI chart pattern where supported.
- Default minimal chart values that are not cluster-specific.

Create a prod overlay at:

```text
infrastructure/controllers/prod-k3s-proxmox/victoria-metrics-k8s-stack/
```

The prod overlay will contain:

- A kustomization that includes the base.
- The current upstream chart version pin.
- Prod-only Helm values patches.
- HTTPRoutes for Grafana and VictoriaMetrics UI.

Enable the stack by adding the prod overlay to:

```text
infrastructure/controllers/prod-k3s-proxmox/kustomization.yaml
```

## Exposure

Use the existing `internal` Gateway in `envoy-gateway-system`.

Routes:

- Grafana: `grafana.${cluster_subdomain}`
- VictoriaMetrics UI: `vm.${cluster_subdomain}`

The routes are intentionally internal-only. Grafana auth is disabled, so these endpoints must not be moved to an internet-facing Gateway without adding authentication.

## Values Strategy

Start from minimal chart values:

- Enable Grafana and disable Grafana auth.
- Enable the VictoriaMetrics single-server UI/query path needed for `vm.${cluster_subdomain}`.
- Avoid enabling Alertmanager or extra alert routing by default.
- Use chart defaults for retention, scrape intervals, and storage unless the chart requires explicit values for a valid install.

This keeps the initial deployment small while allowing future overlays to add retention, persistence, alert rules, or remote write settings.

## Validation

After implementation, validate with:

```bash
kustomize build clusters/prod-k3s-proxmox
kustomize build infrastructure/controllers/prod-k3s-proxmox
```

Also run the repository's broader manifest validation if the change touches shared base behavior.

## Non-Goals

- Do not add the stack to `staging-eu-1` yet.
- Do not add SOPS secrets for Grafana credentials because auth is disabled.
- Do not add alert notification integrations.
- Do not add dashboards beyond what the chart provides by default.
