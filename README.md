# grafana-dashboards-kube

Grafana dashboards for a self-hosted Kubernetes cluster running Cilium CNI.

## Dashboards

| File | Title | Description |
|------|-------|-------------|
| `dashboards/cilium-bgp.json` | BGP Control Plane | Cilium BGP session state, prefix counts, peer status per node |
| `dashboards/kubernetes-cluster-status.json` | Cluster Status | Node health, pod counts, resource usage across the cluster |

## Structure

```
.
├── dashboards/
│   ├── cilium-bgp.json                  # Cilium BGP Control Plane dashboard
│   └── kubernetes-cluster-status.json   # Kubernetes Cluster Status dashboard
└── provisioning/
    ├── kustomization.yaml               # Kustomize configMapGenerator
    └── configmap.yaml                   # ConfigMap metadata (label: grafana_dashboard=1)
```

## How It Works

Grafana's `grafana-sc-dashboard` sidecar watches for ConfigMaps with the label `grafana_dashboard: "1"` in the `monitoring` namespace and auto-loads them into Grafana — no restart required.

## Deploy

### Option 1 — Kustomize (recommended)

```bash
kubectl apply -k provisioning/
```

### Option 2 — kubectl

```bash
kubectl create configmap grafana-dashboards-kube \
  --from-file=dashboards/cilium-bgp.json \
  --from-file=dashboards/kubernetes-cluster-status.json \
  -n monitoring --dry-run=client -o yaml | \
kubectl apply -f -

kubectl label configmap grafana-dashboards-kube \
  grafana_dashboard=1 -n monitoring
```

### Option 3 — Helm values (kube-prometheus-stack)

If Grafana is deployed via the `kube-prometheus-stack` Helm chart, use the built-in dashboard provisioning:

```yaml
# values.yaml
grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: kube
          folder: Kubernetes
          type: file
          options:
            path: /var/lib/grafana/dashboards/kube
  dashboardsConfigMaps:
    kube: grafana-dashboards-kube
```

## Cluster Context

- **CNI:** Cilium in kube-proxy-replacement mode with BGP Control Plane
- **BGP:** ASN 65001 (cluster) peering with ASN 65000 (two routers, active/active)
- **Nodes:** 1 control plane + 3 workers on Proxmox bare-metal
- **Metrics source:** Prometheus scraping Cilium `/metrics` and node exporters
