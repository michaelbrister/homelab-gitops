# kube-prometheus-stack on MicroK8s with Argo CD

This tutorial installs the homelab metrics and alerting platform: Prometheus Operator, Prometheus, Alertmanager, Grafana, kube-state-metrics, and node-exporter.

The completed storage configuration is:

```text
Namespace: monitoring
Grafana PVC: 5Gi
Prometheus PVC: 10Gi
Alertmanager PVC: 2Gi
StorageClass: microk8s-hostpath
Prometheus retention: 7d
Ingress: Traefik
```

The exact chart version selected during the completed deployment was not retained in the conversation export. This tutorial therefore makes version discovery and pinning an explicit step; use the same pinned version in both Helm rendering and Argo CD.

## 1. Prerequisites

Complete:

- `MicroK8s-foundation-tutorial.md`
- `ArgoCD-gitops-bootstrap-tutorial.md`

Verify:

```bash
kubectl get application homelab -n argo-cd
kubectl get storageclass microk8s-hostpath
kubectl get ingressclass traefik
```

## 2. Select and pin the chart

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update

helm search repo prometheus-community/kube-prometheus-stack \
  --versions | head -10
```

Select a chart version and save it:

```bash
KPS_CHART_VERSION="REPLACE_WITH_SELECTED_VERSION"
```

Do not use an unpinned `latest` value. CRD changes can make chart upgrades operationally significant.

## 3. Create the repository files

```bash
mkdir -p apps/kube-prometheus-stack
touch apps/kube-prometheus-stack/values.yaml
touch argocd/applications/kube-prometheus-stack.yaml
```

## 4. Configure the monitoring stack

Create `apps/kube-prometheus-stack/values.yaml`:

```yaml
grafana:
  enabled: true

  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - grafana.home.mikebrister.com

  persistence:
    enabled: true
    storageClassName: microk8s-hostpath
    size: 5Gi

prometheus:
  enabled: true

  prometheusSpec:
    retention: 7d

    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: microk8s-hostpath
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi

alertmanager:
  enabled: true

  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: microk8s-hostpath
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 2Gi
```

These values deliberately start close to the chart defaults. The main homelab-specific choices are ingress, retention, and persistent storage.

## 5. Create the Argo CD Application

Create `argocd/applications/kube-prometheus-stack.yaml`, inserting the selected version:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argo-cd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: REPLACE_WITH_SELECTED_VERSION
      helm:
        valueFiles:
          - $values/apps/kube-prometheus-stack/values.yaml

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
      - ServerSideApply=true
```

`ServerSideApply=true` is important because this chart includes large Prometheus Operator CRDs that can exceed client-side apply annotation limits.

## 6. Render locally

```bash
helm template kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version "$KPS_CHART_VERSION" \
  -f apps/kube-prometheus-stack/values.yaml \
  > /tmp/kube-prometheus-stack.yaml
```

Verify a substantial manifest was produced:

```bash
wc -l /tmp/kube-prometheus-stack.yaml
grep '^kind: CustomResourceDefinition' \
  /tmp/kube-prometheus-stack.yaml | head
```

Inspect the Grafana ingress:

```bash
grep -A45 'kind: Ingress' /tmp/kube-prometheus-stack.yaml
```

## 7. Commit and deploy

```bash
git add \
  apps/kube-prometheus-stack/values.yaml \
  argocd/applications/kube-prometheus-stack.yaml

git commit -m "add kube-prometheus-stack"
git push
```

Watch Argo CD:

```bash
kubectl get applications -n argo-cd -w
```

Expected:

```text
kube-prometheus-stack   Synced   Healthy
```

## 8. Validate the installed components

```bash
kubectl get pods -n monitoring
```

The stack includes components corresponding to:

```text
Prometheus Operator
Prometheus
Alertmanager
Grafana
kube-state-metrics
node-exporter
```

Verify deployments and StatefulSets:

```bash
kubectl get deployments,statefulsets,daemonsets -n monitoring
```

All desired replicas should be available.

## 9. Validate persistent storage

```bash
kubectl get pvc -n monitoring
```

Confirm the Grafana, Prometheus, and Alertmanager claims are `Bound` with the expected sizes.

The relationship is:

```text
Grafana       -> 5Gi hostpath PVC
Prometheus    -> 10Gi hostpath PVC, 7-day retention
Alertmanager  -> 2Gi hostpath PVC
```

## 10. Retrieve the Grafana administrator password

The chart generates the password in a Secret:

```bash
kubectl get secret -n monitoring \
  kube-prometheus-stack-grafana \
  -o jsonpath='{.data.admin-password}' \
  | base64 --decode
echo
```

The username is:

```text
admin
```

Do not add this password to Git.

## 11. Open Grafana

Verify DNS and ingress:

```bash
dig +short grafana.home.mikebrister.com
kubectl get ingress -n monitoring
curl -I http://grafana.home.mikebrister.com
```

Open:

```text
http://grafana.home.mikebrister.com
```

Trusted HTTPS is added later in `Cert-manager-Cloudflare-tutorial.md`.

## 12. Validate Prometheus in Grafana

In Grafana:

1. Open **Connections → Data sources**.
2. Confirm Prometheus is provisioned.
3. Open a Kubernetes node dashboard.
4. Confirm current CPU, memory, filesystem, and network data appears for the MicroK8s node.
5. Open namespace and pod dashboards and confirm workload metrics appear.

This proves the stack is collecting useful cluster data rather than merely running.

## 13. Inspect Prometheus Operator APIs

```bash
kubectl get crd | grep monitoring.coreos.com
```

Important resources include:

```text
alertmanagerconfigs.monitoring.coreos.com
podmonitors.monitoring.coreos.com
prometheusrules.monitoring.coreos.com
servicemonitors.monitoring.coreos.com
```

List the ServiceMonitors installed by the chart:

```bash
kubectl get servicemonitors -A
```

The monitoring discovery model is:

```text
Application metrics endpoint
        |
        v
Kubernetes Service
        |
        v
ServiceMonitor
        |
        v
Prometheus
        |
        v
Grafana dashboard
```

## 14. Validate Prometheus targets

Port-forward Prometheus:

```bash
kubectl port-forward -n monitoring \
  service/kube-prometheus-stack-prometheus \
  9090:9090
```

Open:

```text
http://localhost:9090/targets
```

Confirm the principal Kubernetes and stack targets are up. Then query:

```promql
up
```

An `up` value of `1` indicates a successfully scraped target.

## 15. Validate Alertmanager

Port-forward Alertmanager:

```bash
kubectl port-forward -n monitoring \
  service/kube-prometheus-stack-alertmanager \
  9093:9093
```

Open:

```text
http://localhost:9093
```

Confirm the UI loads and the Alertmanager pod uses its persistent claim.

## 16. Restart and persistence validation

Delete the Grafana pod and allow its controller to recreate it:

```bash
kubectl delete pod -n monitoring \
  -l app.kubernetes.io/name=grafana

kubectl get pods -n monitoring -w
```

Sign in again and verify dashboards and configuration remain available.

Repeat only if desired for Prometheus and Alertmanager, ensuring one component is tested at a time.

## 17. Argo CD ownership

```bash
kubectl get application kube-prometheus-stack -n argo-cd
```

All durable configuration changes should be made in:

```text
apps/kube-prometheus-stack/values.yaml
```

After a Git push, Argo CD renders the pinned chart and reconciles the stack. Avoid manual edits to generated Deployments, StatefulSets, Services, or Ingresses.

## 18. Later integrations

The following changes belong to later tutorials:

- `Loki-tutorial.md` provisions a Loki datasource through this values file.
- `Cert-manager-Cloudflare-tutorial.md` adds trusted TLS to the Grafana ingress.
- `External-Secrets-Operator-tutorial.md` enables an ESO ServiceMonitor.

## 19. Rollback and storage notes

To roll back a values change:

1. revert the Git commit;
2. push the revert;
3. let Argo CD synchronize;
4. verify the Application, pods, targets, and dashboards.

The PVCs survive normal pod recreation. They do not protect against loss of the single MicroK8s node or its disk.

CRDs may outlive chart removal. Do not delete Prometheus Operator CRDs casually; doing so removes their custom resources across the cluster.

## 20. Final architecture

```text
MicroK8s targets
   |-- API server
   |-- node-exporter
   |-- kube-state-metrics
   `-- application ServiceMonitors
                 |
                 v
             Prometheus
                 |
          +------+------+
          |             |
          v             v
       Grafana      Alertmanager

Persistent data -> microk8s-hostpath PVCs
Git desired state -> Argo CD -> Helm chart
```
