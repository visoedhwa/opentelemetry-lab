# Questions & Decisions

> **Assumption**: All components (collectors, backends, dashboards) run **on-premise** on LTC-managed infrastructure. No external SaaS dependencies.

## Should the Collector run as a DaemonSet (per-node) or a centralized Deployment (gateway)?

### Collector Deployment Modes

#### DaemonSet (agent per node)
- One collector pod on **every node** in the cluster
- Apps send OTLP to `localhost` (or the node's collector via a `hostPort` / downward API)
- **Pros**: low-latency collection, data stays local until exported, natural fit for node-level log collection
- **Cons**: more collector instances to manage, each node uses memory/CPU for the collector

#### Deployment (centralized gateway)
- One (or a few replicated) collector pod(s) per cluster
- Apps send OTLP to a single `otel-collector` ClusterIP service
- **Pros**: simpler to manage, fewer instances, easier to apply uniform processing/filtering
- **Cons**: single point of failure (unless replicated), all telemetry funnels through one service (potential bottleneck under heavy load)

#### Sidecar (per-pod)
- One collector container injected into each app pod
- **Pros**: full isolation per app
- **Cons**: highest resource overhead, hardest to manage at scale

### Multi-Cluster: The Recommended Pattern (Two-Tier Architecture)

For the LTC setup (Production, Staging, Dev_cp, Dev_vsm), the common production pattern is a **two-tier architecture**:

```
  LTC On-Premise Network (Burnaby / Downtown campus)
  ────────────────────────────────────────────────────

  Cluster A (e.g. Production)              Cluster B (e.g. Staging)
  ┌──────────────────────────┐           ┌──────────────────────────┐
  │  App Pods → DaemonSet    │           │  App Pods → DaemonSet    │
  │  (agent collectors)      │           │  (agent collectors)      │
  │         │                │           │         │                │
  │         ▼                │           │         ▼                │
  │  Gateway Collector       │           │  Gateway Collector       │
  │  (Deployment, 2+ replicas)│          │  (Deployment, 2+ replicas)│
  └─────────┬────────────────┘           └─────────┬────────────────┘
            │  (on-prem LAN)                       │
            └──────────┬───────────────────────────┘
                       ▼
    Observability Cluster (on-prem, e.g. Rancher Admin or dedicated)
    ┌─────────────────────────────────────────────┐
    │  Grafana / Jaeger / Prometheus / Loki        │
    │  Longhorn PVs for retention storage          │
    │  Vault integration for secrets               │
    └─────────────────────────────────────────────┘
```

#### Why two tiers?

| Layer | Mode | Purpose |
|---|---|---|
| **Tier 1 — Agent** (DaemonSet) | Per-node in each cluster | Collect telemetry close to the source, do lightweight processing (batching, basic filtering), forward to the gateway |
| **Tier 2 — Gateway** (Deployment) | Per-cluster (replicated) | Aggregate, sample, enrich (e.g. add `cluster.name` attribute), then export to centralized backends |

#### Why per-cluster gateways (not one global gateway)?

- **Resilience**: if the network link between clusters is down, each cluster's gateway buffers locally
- **Blast radius**: a misconfigured pipeline in one cluster doesn't affect others
- **Cluster-specific processing**: you can apply different sampling rates (e.g. heavier sampling in dev, lighter in prod)
- **Attribute enrichment**: the gateway adds `cluster.name=production` or `cluster.name=staging` so backends can filter/group by cluster
- **Latency**: telemetry stays intra-cluster until the gateway forwards it

#### Where do backends live? (on-premise)

Since everything is on-prem, the recommended approach is:

- **Dedicated observability cluster** (or a dedicated Rancher "Project" / namespace on the Rancher Admin cluster). All per-cluster gateways export here over the on-prem LAN.
- No external SaaS dependency — all data stays within LTC network boundaries.

#### On-Premise Considerations

Running the full observability stack on-prem introduces concerns that a SaaS offering would handle for you:

| Concern | What it means | Mitigation |
|---|---|---|
| **Storage capacity** | Traces, metrics, and logs consume disk. Prometheus TSDB and Loki chunks grow with traffic volume and retention window. | Use Longhorn PVs with adequate disk. Set retention policies (e.g. Prometheus `--storage.tsdb.retention.time=15d`, Loki retention). Monitor disk usage with alerts. |
| **High availability** | A single Prometheus or Jaeger instance is a SPOF. | Run replicated Prometheus (or Thanos/Cortex for HA + long-term storage). Run Jaeger with an HA backend (e.g. OpenSearch or Cassandra). |
| **Backup & disaster recovery** | On-prem means you own backups. | Longhorn snapshots + off-site backup for Prometheus TSDB, Loki data, and Grafana dashboards (export as JSON or use `grafana-backup`). |
| **Network bandwidth** | All telemetry crosses the on-prem LAN from app clusters to the observability cluster. | Collector gateways batch and compress before exporting (`batch` processor + gzip). Sampling reduces volume. Burnaby ↔ Downtown traffic may traverse WAN — consider placing a backend in each campus or using aggressive sampling for cross-campus clusters. |
| **Resource planning** | Backends need dedicated CPU/memory/disk on-prem VMs. | Size based on ingest rate. Rule of thumb: ~1 GB Prometheus TSDB per 1M active series per 2h block. Loki: depends on log volume. Start small, monitor, scale. |
| **Secrets & TLS** | Collector-to-backend and backend-to-backend communication should be encrypted. | Use Vault for TLS certs and exporter credentials. Collectors authenticate to backends via mTLS or bearer tokens stored in Vault. |
| **Upgrades & maintenance** | You own patching Prometheus, Grafana, Loki, Jaeger, and the Collector. | Pin Helm chart versions. Use GitOps (Terraform + Helm values in git) to track and reproduce deployments. Test upgrades in staging first. |

### For the Lab Right Now

The current single Rancher Desktop cluster with one collector Deployment (acting as both agent + gateway) is perfectly adequate. When ready to simulate the on-prem multi-cluster layout:

1. Use **k3d** to spin up 2–3 named clusters (simulating Production, Staging, etc.)
2. Deploy a **DaemonSet agent + gateway Deployment** in each
3. Deploy backends in a separate "observability" k3d cluster (simulating the dedicated on-prem observability cluster)
4. Have all gateways export to that observability cluster over the Docker network (simulating on-prem LAN)
5. Add `resource.attributes: cluster.name=<name>` in each gateway's config
6. Configure Longhorn (or local-path-provisioner) PVs with retention limits to simulate real on-prem storage constraints
7. Test Vault integration for collector secrets (API keys, TLS certs)