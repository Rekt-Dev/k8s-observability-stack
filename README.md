# k8s-observability-stack

Full observability setup for a kubeadm homelab cluster using kube-prometheus-stack. Covers metrics collection, custom alerting rules, and Grafana dashboards. All values are tuned for a resource-constrained 3-node cluster rather than cloud defaults.

## Cluster Context

- **Nodes:** k8s (control plane, 192.168.1.110), k8s1, k8s2
- **CNI:** Calico (VXLAN mode)
- **Container runtime:** containerd
- **Kubernetes version:** v1.35.x

Complements the security hardening in [k8s-security-hardening](https://github.com/Rekt-Dev/k8s-security-hardening) — alert rules cover security-relevant events as well as operational health.

## Why kube-prometheus-stack

The alternative is deploying Prometheus Operator, Prometheus, Grafana, Alertmanager, node-exporter, and kube-state-metrics individually. The Helm chart bundles all of them with pre-wired ServiceMonitors for Kubernetes components (etcd, kube-scheduler, kube-controller-manager, kubelet). Starting from scratch means rewriting all of that integration.

The tradeoff is the chart is large and opinionated. The values file in this repo strips it down to what actually matters for a homelab.

## Repository Structure

```
namespace.yaml              Monitoring namespace with Pod Security labels
prometheus/
  values.yaml               kube-prometheus-stack Helm values (homelab-tuned)
  alerts/
    custom-alerts.yaml      PrometheusRule CRD for homelab-specific alerts
grafana/
  datasources.yaml          Grafana datasource provisioning
alertmanager/
  config.yaml               Alertmanager routing config
```

## Installation

```bash
# Create namespace first (sets Pod Security labels)
kubectl apply -f namespace.yaml

# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install (dry-run first)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f prometheus/values.yaml \
  --dry-run

# Actual install
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f prometheus/values.yaml

# Apply custom alert rules
kubectl apply -f prometheus/alerts/custom-alerts.yaml
```

## Resource Tuning for Homelab

The default kube-prometheus-stack values allocate approximately 2GB RAM for Prometheus alone. On a homelab with 3 nodes at 16GB each (shared with running workloads), that leaves very little headroom.

Changes made in `prometheus/values.yaml`:

| Component | Default Memory Limit | This Config |
|-----------|---------------------|-------------|
| Prometheus | 2Gi | 1Gi |
| Grafana | 512Mi | 256Mi |
| Alertmanager | 256Mi | 64Mi |

Retention is set to 15 days (default: 90 days) with a size cap of 8GB. On spinning disk, Prometheus TSDB compaction generates noticeable I/O every few hours — the shorter retention keeps compaction overhead low.

Scrape interval is 60 seconds (default: 15s). For homelab, 60s is sufficient. 15s means 4x more samples, 4x more disk I/O, and 4x more RAM for the Prometheus TSDB head.

## etcd Scraping

Scraping etcd requires TLS client certificates. The kubeadm etcd client certs are at `/etc/kubernetes/pki/etcd/` on the control plane node. Create a Secret from them:

```bash
kubectl create secret generic etcd-client-cert \
  --from-file=/etc/kubernetes/pki/etcd/ca.crt \
  --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  -n monitoring
```

Then reference it in the `prometheus/values.yaml` `kubeEtcd.serviceMonitor` section (already configured).

## Custom Alert Rules

The `custom-alerts.yaml` PrometheusRule defines alerts that the default rules don't cover or use thresholds unsuitable for homelab:

- **NodeMemoryPressureHigh**: fires at <10% available (default rules use different thresholds)
- **NodeDiskFull**: homelab disks are smaller; 15% free triggers a warning
- **EtcdHighCommitDuration**: homelab etcd runs on HDD rather than SSD — latency thresholds adjusted accordingly
- **PodCrashLooping**: 3+ restarts in 15 minutes (tighter than defaults)

## Grafana Access

With the NodePort configuration in `values.yaml`, Grafana is accessible at:

```
http://192.168.1.110:30300
```

Default credentials: `admin / changeme-on-first-login`

Change the password immediately:

```bash
kubectl exec -n monitoring deployment/monitoring-grafana -- \
  grafana-cli admin reset-admin-password <new-password>
```

## Upgrading

```bash
helm repo update
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f prometheus/values.yaml \
  --reuse-values
```

Always check the chart release notes for CRD updates before upgrading. The kube-prometheus-stack CRDs must be updated manually when they change:

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/crds/crd-prometheusrules.yaml
```

## Lessons Learned

**kube-state-metrics OOMed on first install.** The default values allocate 128Mi which was insufficient for a cluster with many objects. Increased to 256Mi. If you have many namespaces, deployments, or pods, check kube-state-metrics memory usage first.

**Prometheus did not scrape etcd initially.** The `endpoints` field under `kubeEtcd` must match the actual IP of the control plane, and the TLS secret must exist in the monitoring namespace before the Prometheus pod starts. Order of operations matters.

**Grafana dashboards are ephemeral without persistence.** Without a PVC, any dashboard created manually disappears on pod restart. Use the provisioning mechanism (ConfigMaps or the `grafana.ini` sidecar) to persist dashboards as code.

**NodePort is fine for homelab but know the tradeoff.** Using NodePort means Grafana is accessible on port 30300 on any cluster node. In a production environment you would put this behind an Ingress with TLS. For a homelab behind a router with no external exposure, NodePort is acceptable.

## Related Repositories

- [k8s-production-patterns](https://github.com/Rekt-Dev/k8s-production-patterns) — workloads being monitored
- [k8s-security-hardening](https://github.com/Rekt-Dev/k8s-security-hardening) — security policies, some alerting covers security events
- [gitops-argocd-homelab](https://github.com/Rekt-Dev/gitops-argocd-homelab) — deploys this stack via ArgoCD
