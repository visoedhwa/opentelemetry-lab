# Commands run (kubectl)

## Cluster validation (Rancher Desktop)
```bash
kubectl version --client
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

## Deploy verification (OpenTelemetry demo)
```bash
kubectl create namespace otel-demo || true
kubectl get pods -n otel-demo -w
kubectl get svc -n otel-demo
```

## Port-forwards
```bash
kubectl port-forward -n otel-demo svc/frontend-proxy 8080:8080
kubectl port-forward -n otel-demo svc/jaeger 16686:16686
kubectl port-forward -n otel-demo svc/prometheus 9090:9090
kubectl port-forward -n otel-demo svc/grafana 3000:80
```