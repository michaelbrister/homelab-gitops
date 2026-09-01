# Loki on MicroK8s with Argo CD and Grafana

This tutorial adds persistent Kubernetes log storage to the monitoring namespace. Loki runs in monolithic mode with one replica and filesystem storage, which fits the single-node homelab.

The completed configuration uses:

```text
Loki chart: 18.7.6
Deployment mode: Monolithic
Replication factor: 1
Schema: TSDB v13
Storage: filesystem
Retention: 168h (7 days)
PVC: 10Gi microk8s-hostpath
Namespace: monitoring
Gateway: loki-gateway.monitoring.svc.cluster.local
```

## 1. Prerequisites

Complete `Kube-prometheus-stack-tutorial.md` first.

Verify:

```bash
kubectl get application kube-prometheus-stack -n argo-cd
kubectl get pods,pvc -n monitoring
```

Grafana and Prometheus should be healthy before Loki is added.

## 2. Add the Grafana Community chart repository

```bash
helm repo add grafana-community \
  https://grafana-community.github.io/helm-charts

helm repo update
helm search repo grafana-community/loki --versions | head
```

The completed deployment pinned chart `18.7.6`.

## 3. Create the repository files

```bash
mkdir -p apps/loki
touch apps/loki/values.yaml
touch argocd/applications/loki.yaml
```

## 4. Configure monolithic Loki

Create `apps/loki/values.yaml`:

```yaml
deploymentMode: Monolithic

loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 1

  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

  storage:
    type: filesystem
    bucketNames:
      chunks: chunks
      ruler: ruler
      admin: admin

  storage_config:
    filesystem:
      directory: /var/loki/chunks

  limits_config:
    retention_period: 168h

singleBinary:
  replicas: 1

  persistence:
    enabled: true
    storageClass: microk8s-hostpath
    size: 10Gi

backend:
  replicas: 0

read:
  replicas: 0

write:
  replicas: 0

ingester:
  replicas: 0

querier:
  replicas: 0

queryFrontend:
  replicas: 0

queryScheduler:
  replicas: 0

minio:
  enabled: false
```

### Why these settings are used

`deploymentMode: Monolithic` and `singleBinary.replicas: 1` run all Loki functions in one process.

`replication_factor: 1` matches the one-replica topology.

TSDB with schema `v13` stores the log index and chunks using Loki's current storage model.

The `bucketNames` values satisfy the chart's storage model, but `storage.type: filesystem` and `object_store: filesystem` keep the actual data on the PVC rather than in object storage.

All distributed component replica counts are zero because they are not used in monolithic mode. MinIO is also disabled.

## 5. Create the Argo CD Application

Create `argocd/applications/loki.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: loki
  namespace: argo-cd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  sources:
    - repoURL: https://grafana-community.github.io/helm-charts
      chart: loki
      targetRevision: 18.7.6
      helm:
        valueFiles:
          - $values/apps/loki/values.yaml

    - repoURL: https://github.com/michaelbrister/homelab-gitops.git
      targetRevision: main
      ref: values

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

Loki shares the `monitoring` namespace with Prometheus and Grafana.

## 6. Render locally

```bash
helm template loki grafana-community/loki \
  --namespace monitoring \
  --version 18.7.6 \
  -f apps/loki/values.yaml \
  > /tmp/loki.yaml
```

Verify success and inspect the resource types:

```bash
wc -l /tmp/loki.yaml
grep '^kind:' /tmp/loki.yaml
```

Confirm the rendered workload has one monolithic replica and a persistent volume claim.

## 7. Commit and deploy

```bash
git add apps/loki/values.yaml argocd/applications/loki.yaml
git commit -m "add Loki"
git push
```

Watch Argo CD:

```bash
kubectl get applications -n argo-cd -w
```

Expected:

```text
loki   Synced   Healthy
```

## 8. Validate Loki resources

```bash
kubectl get pods -n monitoring | grep loki
kubectl get pvc -n monitoring | grep loki
kubectl get service -n monitoring | grep loki
```

Confirm:

- the monolithic Loki pod is running;
- its 10Gi PVC is `Bound`;
- the `loki` and `loki-gateway` services exist.

## 9. Validate Loki readiness

Port-forward the Loki process service:

```bash
kubectl port-forward -n monitoring svc/loki 3100:3100
```

In another terminal:

```bash
curl http://localhost:3100/ready
```

Expected:

```text
ready
```

This endpoint checks the Loki process directly.

## 10. Validate the gateway API

Stop the previous port-forward, then forward the gateway:

```bash
kubectl port-forward -n monitoring svc/loki-gateway 3100:80
```

Query a Loki API endpoint:

```bash
curl http://localhost:3100/loki/api/v1/labels
```

Before Alloy is installed, the label list may be empty. A valid Loki JSON response confirms the gateway routes Loki API paths correctly.

The gateway is the stable endpoint used by Grafana and Alloy:

```text
http://loki-gateway.monitoring.svc.cluster.local
```

## 11. Provision the Loki datasource in Grafana

Update the existing `grafana:` section in `apps/kube-prometheus-stack/values.yaml` by adding:

```yaml
grafana:
  additionalDataSources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki-gateway.monitoring.svc.cluster.local
      isDefault: false
```

Merge this with the existing Grafana ingress and persistence configuration rather than creating a second `grafana:` key.

Render kube-prometheus-stack again with its pinned version:

```bash
helm template kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version "$KPS_CHART_VERSION" \
  -f apps/kube-prometheus-stack/values.yaml \
  > /tmp/kube-prometheus-stack-with-loki.yaml
```

Commit and push:

```bash
git add apps/kube-prometheus-stack/values.yaml
git commit -m "add Loki Grafana datasource"
git push
```

Wait for Argo CD:

```bash
kubectl get applications -n argo-cd \
  kube-prometheus-stack loki
```

Both should be `Synced` and `Healthy`.

## 12. Validate the Grafana datasource

In Grafana:

1. Open **Connections → Data sources**.
2. Select **Loki**.
3. Confirm the URL is the in-cluster gateway address.
4. Run **Save & test** if the UI offers it.

At this stage the datasource is connected, but there may be no Kubernetes pod logs yet. `Grafana-Alloy-tutorial.md` installs the collector that supplies them.

## 13. Validate persistence

Record the Loki pod name and PVC:

```bash
kubectl get pods,pvc -n monitoring | grep loki
```

Delete the Loki pod:

```bash
kubectl delete pod -n monitoring \
  -l app.kubernetes.io/name=loki

kubectl get pods -n monitoring -w
```

After the replacement is ready:

```bash
kubectl port-forward -n monitoring svc/loki 3100:3100
curl http://localhost:3100/ready
```

The same PVC should remain bound and the process should return `ready`.

## 14. Argo CD ownership

```bash
kubectl get application loki -n argo-cd
```

The ownership boundary is:

```text
argocd/applications/loki.yaml
  -> chart source, version, namespace, sync policy

apps/loki/values.yaml
  -> Loki topology, retention, and storage

apps/kube-prometheus-stack/values.yaml
  -> Grafana's Loki datasource
```

Make operational changes through Git and let Argo CD reconcile them.

## 15. Rollback and storage notes

To roll back a configuration change, revert its Git commit and push. Verify both the Loki and kube-prometheus-stack Applications afterward.

The 10Gi filesystem PVC is suitable for this low-volume, single-node lab. It is not distributed object storage and does not survive loss of the node or disk.

Retention limits old data but does not replace capacity monitoring. Watch the PVC's filesystem usage in Prometheus/Grafana as log volume grows.

## 16. Final architecture

```text
Grafana
   |
   | Loki datasource
   v
loki-gateway
   |
   v
Monolithic Loki
   |
   v
10Gi microk8s-hostpath PVC

Alloy is added next:
Kubernetes pod logs -> Alloy -> loki-gateway
```
