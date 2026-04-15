# OpenTelemetry Demo — Service Architecture

This document describes every component deployed in the `otel-demo` namespace, its role, and how it connects to other components.

---

## 1. Application Services (microservices)

These are the demo "business" services. Each one is instrumented with an OpenTelemetry SDK and sends traces, metrics, and logs to the **OpenTelemetry Collector** via OTLP (gRPC on port 4317 or HTTP on port 4318).

| Service | Role | Key dependencies |
|---|---|---|
| **frontend** | Web storefront UI (Next.js). Entry point for user traffic. | `frontend-proxy`, `product-catalog`, `cart`, `checkout`, `recommendation`, `ad`, `currency` |
| **frontend-proxy** (Envoy) | Reverse proxy that sits in front of `frontend` and routes `/jaeger`, `/grafana`, `/loadgen`, etc. to the correct backend UIs. This is what you port-forward to on port 8080. | `frontend`, `jaeger`, `grafana`, `load-generator`, `flagd` |
| **cart** | Manages shopping cart state. | `valkey-cart` (in-memory store) |
| **checkout** | Orchestrates the checkout flow — calls payment, shipping, email, and product-catalog. | `cart`, `payment`, `shipping`, `email`, `product-catalog`, `currency`, `kafka` |
| **product-catalog** | Serves the product list and product details. | None (self-contained data) |
| **product-reviews** | Serves product reviews. | None |
| **recommendation** | Returns product recommendations based on the catalog. | `product-catalog` |
| **currency** | Converts prices between currencies. | None |
| **payment** | Simulates payment processing. | None |
| **shipping** | Calculates shipping cost and simulates shipment. | None |
| **email** | Sends order confirmation emails (simulated). | None |
| **ad** | Returns contextual advertisements. | None |
| **quote** | Returns shipping quotes. | None |
| **image-provider** | Serves product images. | None |
| **accounting** | Consumes order events from Kafka for bookkeeping. | `kafka` |
| **fraud-detection** | Consumes order events from Kafka to flag fraud. | `kafka` |
| **llm** | Simulated LLM service for AI-powered features. | None |

### How they connect

```
                    ┌─────────────────┐
  User traffic ───► │  frontend-proxy  │ (port 8080)
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │    frontend      │
                    └──┬──┬──┬──┬──┬──┘
                       │  │  │  │  │
          ┌────────────┘  │  │  │  └──────────┐
          ▼               ▼  ▼  ▼             ▼
   product-catalog    cart  ad  currency  recommendation
                       │
                       ▼
                  valkey-cart
                       │
          ┌────────────┘  (checkout reads cart)
          ▼
      checkout ──► payment, shipping, email, currency
          │
          ▼
        kafka ──► accounting, fraud-detection
```

---

## 2. OpenTelemetry Collector

| Component | Role |
|---|---|
| **otel-collector** (DaemonSet agent) | Central telemetry pipeline. Every application service sends OTLP data (traces, metrics, logs) to the collector. The collector then processes (batches, filters) and exports to the backends below. |

### Connections

- **Receives from**: all application services (OTLP gRPC :4317, OTLP HTTP :4318)
- **Exports to**:
  - **Jaeger** — traces (via OTLP or Jaeger exporter on :14250)
  - **Prometheus** — metrics (scraped from the collector's Prometheus exporter on :8888, or via remote-write)
  - **OpenSearch** — logs (via the OpenSearch exporter)

```
[ All App Services ]
        │  OTLP (gRPC/HTTP)
        ▼
[ otel-collector ]
   ├──► jaeger       (traces)
   ├──► prometheus   (metrics)
   └──► opensearch   (logs)
```

---

## 3. Observability Backends

| Component | Role | How it gets data | UI port |
|---|---|---|---|
| **jaeger** | Trace storage and query engine. Lets you search traces by service, operation, duration, and errors. | Receives traces from the OTel Collector (OTLP/Jaeger protocol on :4317 or :14250). | 16686 |
| **prometheus** | Metrics storage and query engine (PromQL). Stores time-series app metrics (request rates, error rates, latencies). | Scrapes metrics endpoints exposed by the OTel Collector and possibly directly from app services. | 9090 |
| **grafana** | Unified dashboards and visualization. Pre-configured with data sources pointing to Jaeger, Prometheus, and OpenSearch. | Queries Jaeger (traces), Prometheus (metrics), and OpenSearch (logs) as configured data sources. Does not receive telemetry directly. | 80 (port-forward to 3000) |
| **opensearch** | Log storage and search engine (like Elasticsearch). Stores structured application logs. | Receives logs from the OTel Collector via the OpenSearch exporter on :9200. | 9200 |

### How backends connect

```
                  ┌──────────┐
                  │  Grafana  │  (queries all three backends)
                  └──┬──┬──┬─┘
                     │  │  │
         ┌───────────┘  │  └───────────┐
         ▼              ▼              ▼
      Jaeger       Prometheus      OpenSearch
      (traces)      (metrics)       (logs)
```

---

## 4. Supporting Infrastructure

| Component | Role | Used by |
|---|---|---|
| **kafka** | Message broker. The `checkout` service publishes order events; `accounting` and `fraud-detection` consume them. | `checkout` → `kafka` → `accounting`, `fraud-detection` |
| **valkey-cart** | In-memory key-value store (Valkey/Redis-compatible). Persists shopping cart data. | `cart` service |
| **postgresql** | Relational database. Stores persistent app data (e.g., feature flags). | `flagd` and potentially other services |
| **flagd** | Feature flag evaluation engine (OpenFeature). Services query flagd to check if features are enabled/disabled. | Queried by multiple app services; configured via `postgresql` |
| **load-generator** | Locust-based traffic generator. Continuously sends synthetic user requests to `frontend` so telemetry is always flowing. | `frontend` (via HTTP) |

---

## 5. End-to-End Data Flow Summary

```
1. load-generator sends HTTP requests to frontend (via frontend-proxy)
2. frontend calls downstream app services (cart, checkout, product-catalog, etc.)
3. App services call each other as needed (checkout → payment, shipping, etc.)
4. checkout publishes order events to kafka
5. accounting + fraud-detection consume from kafka

--- telemetry path (parallel to above) ---

6. Every app service emits OTLP traces/metrics/logs → otel-collector
7. otel-collector exports:
     traces  → jaeger
     metrics → prometheus
     logs    → opensearch
8. grafana queries jaeger, prometheus, opensearch to render dashboards
9. You access everything via frontend-proxy (port 8080) or individual port-forwards
```

## 6. Port-Forward Quick Reference

| URL | Service | What you see |
|---|---|---|
| `http://localhost:8080` | frontend-proxy | Demo storefront + links to Grafana/Jaeger/LoadGen |
| `http://localhost:16686` | jaeger | Trace search and detail view |
| `http://localhost:9090` | prometheus | PromQL query UI |
| `http://localhost:3000` | grafana | Pre-built dashboards (traces, metrics, logs) |