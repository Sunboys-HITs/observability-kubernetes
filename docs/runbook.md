# Observability Runbook

## Check Stack

```bash
kubectl -n monitoring get pods
kubectl -n monitoring get svc
kubectl -n monitoring get pvc
```

## Open Grafana

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

## Check Loki Ingestion

```bash
kubectl -n monitoring logs daemonset/alloy --tail=100
kubectl -n monitoring logs statefulset/loki --tail=100
```

## Common Grafana Explore Queries

```logql
{namespace="app"}
{namespace="app", app="auth-service"}
{namespace="app", app="code-runner-service"}
{namespace="infra", app="rabbitmq"}
{namespace="infra", app="postgres"}
```

## If Logs Are Missing

1. Check that Alloy pods are running on every node.
2. Check that Loki gateway service exists.
3. Check pod labels. The preferred app label is `app.kubernetes.io/name`.
4. Check container stdout with `kubectl logs` to confirm the app emits logs.
5. Avoid using high-cardinality labels such as request IDs or user IDs as Loki labels.
