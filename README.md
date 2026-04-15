# OpenTelemetry Lab

Research and development for OpenTelemetry observability setup, targeting BCIT LTC infrastructure.

## Background

This project tracks the R&D effort to evaluate and build out an [OpenTelemetry](https://opentelemetry.io/) **application observability** stack — collecting logs, metrics, and traces from LTC apps — that can eventually be integrated into the LTC's Kubernetes-based infrastructure.

### LTC Infrastructure Summary

The production LTC environment (documented at [infrastructure-documentation.ltc.bcit.ca](https://infrastructure-documentation.ltc.bcit.ca/infrastructure-details/)) consists of:

- **Kubernetes distribution**: RKE2 (Rancher Kubernetes Engine 2), provisioned via Ansible
- **Cluster management UI**: Rancher
- **Cluster environments**:
  - **Production** — dedicated manager nodes (3×, HA) + separate worker nodes
  - **Staging** — combo (manager+worker) nodes
  - **Dev_cp / Dev_vsm** — combo nodes for development
  - **Rancher Admin** — dedicated cluster running the Rancher UI
- **Common cluster components**: nginx Ingress, Longhorn (persistent storage), GitLab Runner (CI/CD)
- **Load balancing**: Traefik (layer 4, in DMZ)
- **Secrets management**: HashiCorp Vault
- **IaC tooling**: Ansible (VM config), Terraform + Helm (K8s services)
- **Source control & CI/CD**: GitHub

## Goal

Build a local proof-of-concept OpenTelemetry environment on macOS to validate **application-level observability**: how LTC apps emit logs, metrics, and traces via OpenTelemetry, how the Collector pipelines process that telemetry, and how it surfaces in backend dashboards — before rolling the setup into staging/production.

## Suggested Local Lab Architecture

### 1. Local Kubernetes — Rancher Desktop or k3d

| Option | Why it fits |
|---|---|
| **Rancher Desktop** | Ships a single-node k3s cluster with the Rancher dashboard — closest match to the LTC's Rancher-managed RKE2 clusters. GUI included. |
| **k3d** (k3s-in-Docker) | Lightweight, lets you spin up _multiple_ named clusters (e.g. `otel-dev`, `otel-staging`) to simulate the multi-cluster layout. |

**Recommendation**: Start with **Rancher Desktop** for simplicity; switch to k3d later if multi-cluster testing is needed.

### 2. OpenTelemetry Components

| Component | Purpose | Install method |
|---|---|---|
| **OpenTelemetry Collector** | Central pipeline — receives app logs, metrics, and traces, then exports to backends | Helm chart (`open-telemetry/opentelemetry-collector`) |
| **OTel Operator** (optional) | Auto-manages Collector instances & injects auto-instrumentation into app pods | Helm chart (`open-telemetry/opentelemetry-operator`) |
| **Auto-instrumentation** | Zero-code SDK injection so apps emit traces, metrics, and structured logs without code changes | CRDs created by the Operator |

### 3. Observability Backends

| Backend | Telemetry signal | Install method |
|---|---|---|
| **Jaeger** | Application traces (request flows, latency, errors) | Helm chart or all-in-one Docker image |
| **Prometheus** | Application metrics (request rates, error rates, durations) | Helm chart (`prometheus-community/kube-prometheus-stack`) |
| **Grafana** | Unified dashboards for app logs, metrics, and traces | Bundled with kube-prometheus-stack |
| **Loki** | Application logs (structured log aggregation) | Helm chart (`grafana/loki-stack`) |

### 4. Sample Workloads

Deploy one or more demo apps to generate telemetry:

- [OpenTelemetry Demo App](https://github.com/open-telemetry/opentelemetry-demo) — multi-service microservices app with pre-wired OTel instrumentation
- Custom lightweight services matching LTC app patterns

### 5. High-Level Data Flow

```
[ LTC App Pods ]
  │  OTel SDK / auto-instrumentation emits:
  │    • traces  (OTLP)
  │    • metrics (OTLP)
  │    • logs    (OTLP)
  ▼
[ OpenTelemetry Collector ]
  │  pipelines: receive → process (batch, filter) → export
  ├──► Jaeger      (traces)
  ├──► Prometheus  (metrics)
  └──► Loki        (logs)
         │
         ▼
      Grafana  (unified app dashboards)
```

## Getting Started (TODO)

> Steps below will be fleshed out as we iterate.

1. **Install Rancher Desktop** (or k3d) on macOS
2. **Add Helm repos** for OTel, Prometheus, Grafana, Jaeger
3. **Deploy observability backends** (Jaeger, Prometheus + Grafana, Loki)
4. **Deploy the OpenTelemetry Collector** with a pipeline config exporting to the backends
5. **Deploy sample app(s)** with auto-instrumentation enabled
6. **Validate** traces in Jaeger, metrics in Prometheus/Grafana, logs in Loki
7. **Iterate** on collector config (filtering, sampling, batching, etc.)

## Repo Structure (Planned)

```
opentelemetry-lab/
├── README.md
├── helm-values/            # Custom Helm values files
│   ├── otel-collector.yaml
│   ├── jaeger.yaml
│   ├── prometheus-stack.yaml
│   └── loki.yaml
├── manifests/              # Raw K8s manifests (if needed)
├── demo-apps/              # Sample instrumented applications
└── docs/                   # Additional research notes
```

## Open Questions

- Which LTC apps should be instrumented first?
- What languages/frameworks do the target apps use? (determines SDK & auto-instrumentation approach)
- Should the Collector run as a DaemonSet (sidecar-like, per-node) or a centralized Deployment (gateway)?
- What trace sampling strategy is appropriate for production traffic volumes?
- Structured logging format — should apps adopt OTel log bridging or emit OTLP logs natively?
- Integration with HashiCorp Vault for collector and exporter secrets?
- Multi-cluster collection — one central collector or per-cluster?

## References

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [OpenTelemetry Operator for Kubernetes](https://github.com/open-telemetry/opentelemetry-operator)
- [LTC Infrastructure Documentation](https://infrastructure-documentation.ltc.bcit.ca/infrastructure-details/)
- [Rancher Desktop](https://rancherdesktop.io/)
- [k3d](https://k3d.io/)
