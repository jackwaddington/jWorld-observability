# jWorld-observability

Lightweight Prometheus + Grafana monitoring stack for a resource-constrained k3s cluster.
Designed for Argo CD — point an Application at the `monitoring/` directory.

## Components

| Component | Purpose | Resources (request) |
|---|---|---|
| Prometheus | Metrics collection & storage | 100m CPU, 128Mi RAM |
| Node Exporter | Node-level metrics (CPU, RAM, disk) | 50m CPU, 32Mi RAM per node |
| Kube State Metrics | Kubernetes object status (pod phase) | 50m CPU, 32Mi RAM |
| Grafana | Dashboard UI | 100m CPU, 64Mi RAM |

**Total per node:** ~150m CPU, ~160Mi RAM (node-exporter on every node, the rest scheduled anywhere)

## Storage & Retention

- Prometheus: 2-day retention, 1GB max TSDB size, WAL compression enabled, 2Gi PVC (`local-path` StorageClass)
- Grafana: stateless — dashboards and datasources provisioned from ConfigMaps

## Access

- **Grafana:** `http://<any-node-ip>:30300` — login: `admin` / `admin`
- **Prometheus:** ClusterIP only (access via `kubectl port-forward svc/prometheus -n monitoring 9090:9090`)

## Dashboard

The provisioned "K3s Cluster Overview" dashboard includes:

- Node CPU usage (%) per node
- Node memory usage (%) per node
- Node disk usage (%) gauge with thresholds (70% yellow, 85% red)
- Pod count by status phase (Running, Pending, Failed, Succeeded)

## Argo CD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <your-repo-url>
    targetRevision: main
    path: monitoring
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Adding Other Projects

Prometheus auto-discovers any pod with the right annotations. To monitor another app,
add these annotations to its pod spec:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"  # optional, defaults to /metrics
```

The app needs to expose a `/metrics` endpoint in Prometheus exposition format.

## Repo Structure

```
monitoring/
  namespace.yaml
  prometheus/
    serviceaccount.yaml
    clusterrole.yaml
    clusterrolebinding.yaml
    configmap.yaml
    deployment.yaml
    pvc.yaml
    service.yaml
  node-exporter/
    daemonset.yaml
    service.yaml
  kube-state-metrics/
    serviceaccount.yaml
    clusterrole.yaml
    clusterrolebinding.yaml
    deployment.yaml
    service.yaml
  grafana/
    configmap-datasources.yaml
    configmap-dashboard-providers.yaml
    configmap-dashboard.yaml
    deployment.yaml
    service.yaml
```
