# Voize case study

## What was added

- **Prometheus** - metrics aggregation
- **Loki** - log aggregation
- **Alloy** - collecting metrics and logs in cluster
- **Grafana** - show dashboards with metrics and logs collected from applications
- **Postgresql Exporter** - collects metrics from PostreSQL server

## How it was added

Deploying Loki, Alloy and Prometheus operator with Helm Charts.

Every Helm Charthas own additional configuration like Persistent Volumes. For Grafana additional dashboards were added to show metrics and logs per every application.

Password for default user in Grafana is automatically generated with Prometheus Operator. It can be extracted from Kuberbetes secrets with:

```bash
kubectl -n monitoring get secret prometheus-stack-grafana -o jsonpath='{.data.admin-password}' | base64 --decode
```

To every application deployment configuration is added additional resource `ServiceMonitor` - it'll be automatically discovered with Prometheus and scraped.

## What is monitored

- HTTP requests: alert on 5xx errors increase
- Database: default metrics collected with exporter
- Kubernetes cluster metrics and logs

Metrics and logs has retention of 24 hours or 4GB size for Prometheus.

## Prometheus Alerts

| Alert                                                  | Condition                                                  | Pending duration |
| ------------------------------------------------------ | ---------------------------------------------------------- | ---------------- |
| MLAPIMetricsUnavailable / BackendAPIMetricsUnavailable | No successful scrape, or all target series absent          | 2m               |
| WorkloadReplicasUnavailable                            | Available replicas below desired for supplied workloads    | 2m               |
| APIHighErrorRate                                       | >5% HTTP 5xx over 10m, at least 10 requests                | 2m               |
| APIHighLatency                                         | p95 over 10m >2s ML or >0.5s Backend, at least 10 requests | 5m               |
| BackendDatabaseQueryErrors                             | DB errors occurred in the preceding 5m                     | 2m               |
| ApplicationMemoryPressure                              | Working set exceeds 85% of container memory limit          | 5m               |
| ApplicationFrequentRestarts                            | More than two restarts over 10m                            | 1m               |
| AlloyUnavailable                                       | No successful self-scrape or series absent                 | 2m               |
| PostgreSQLUnavailable                                  | Exporter reports `pg_up = 0`                               | 2m               |
| PostgresExporterUnavailable                            | Exporter HTTP scrape fails or series absent                | 2m               |

## Simulate DB unavailability

To test alerting for PostgreSQL server we can setup `NetworkPolicy` that restricts access to the deployed PostgreSQL service.

`NetworkPolicy` example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: incident-block-postgres
  namespace: postgres
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes: [Ingress]
  ingress: []
```
