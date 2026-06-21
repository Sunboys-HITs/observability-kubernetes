# observability-kubernetes

Kubernetes observability stack for SUNBOYS services.

## What It Deploys

- `Grafana`: UI for service logs and metrics.
- `Prometheus`: metrics storage and scraping for MainService.
- `Loki`: lightweight centralized log storage.
- `Alloy`: Kubernetes log collector that ships container stdout/stderr logs to Loki.

The stack is deployed into namespace `monitoring`.

This repository uses a minimal logs-only profile for small k3s nodes. It uses
the standalone Grafana chart; Prometheus Operator and its CRDs are not installed.
Only `app` and `infra` namespace logs are collected.

Default resource caps:

- Grafana: `100m` CPU / `192Mi` memory.
- Loki: `150m` CPU / `256Mi` memory, `6h` retention, `1Gi` PVC.
- Alloy: `50m` CPU / `64Mi` memory per node.

## Deploy

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace \
  --values helm/prometheus/values.yaml

helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --values helm/grafana/values.yaml

helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --create-namespace \
  --values helm/loki/values.yaml

helm upgrade --install alloy grafana/alloy \
  --namespace monitoring \
  --create-namespace \
  --values helm/alloy/values.yaml
```

## GitHub Actions Deploy

Pushes to `main` deploy the stack automatically.

Required repository secret:

```text
KUBECONFIG_B64
GRAFANA_ADMIN_PASSWORD
```

Create it from the kubeconfig that points to the k3s API:

```bash
base64 -w0 ~/.kube/sunboys-k3s.yaml
```

## Access Grafana

Grafana is exposed through Kubernetes Ingress.

Default host:

```text
http://grafana.31.192.111.254.sslip.io
```

To override it, set the repository variable before deploy:

```text
GRAFANA_HOST=grafana.<cluster-ip>.sslip.io
```

Then open:

```text
http://grafana.<cluster-ip>.sslip.io
```

For local-only access you can still use port-forward:

```bash
kubectl -n monitoring port-forward svc/grafana 3000:80
```

Open:

```text
http://localhost:3000
```

Default credentials:

```text
admin / value from GRAFANA_ADMIN_PASSWORD
```

Do not expose Grafana with the checked-in development password.

## Useful LogQL

Grafana automatically provisions the `SUNBOYS / Service Logs` dashboard from
`k8s/dashboards/service-logs-dashboard.yaml`.

It also provisions `SUNBOYS / MainService Metrics`. Prometheus scrapes
`http://main-service.app.svc.cluster.local/metrics` automatically.

All app logs:

```logql
{namespace="app"}
```

One service:

```logql
{namespace="app", app="code-runner-service"}
```

Errors:

```logql
{namespace="app"} |~ "(?i)error|exception|failed"
```

JSON logs after services are configured for structured logging:

```logql
{namespace="app"} | json | level="error"
```

## Service Logging Contract

Services should write structured JSON logs to stdout/stderr with:

- `timestamp`
- `level`
- `service`
- `version`
- `correlationId`
- `submissionId` where applicable
- `message`
- `error.type`, `error.message`, `error.stack` for exceptions

Do not log passwords, tokens, full user source code, or full test data.
