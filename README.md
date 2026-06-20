# observability-kubernetes

Kubernetes observability stack for SUNBOYS services.

## What It Deploys

- `kube-prometheus-stack`: Prometheus, Grafana, Alertmanager, node and Kubernetes metrics.
- `loki`: centralized log storage.
- `alloy`: Kubernetes log collector that ships container stdout/stderr logs to Loki.

The stack is deployed into namespace `monitoring`.

## Deploy

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values helm/kube-prometheus-stack/values.yaml

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
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
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
