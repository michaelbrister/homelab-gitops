# Grafana Alloy Kubernetes Log Collection with Loki

This tutorial deploys Grafana Alloy to discover Kubernetes pods, collect their container logs through the Kubernetes API, add useful labels, and send the logs to Loki.

The completed configuration uses:

```text
Alloy chart: 1.11.1
Chart repository: https://grafana.github.io/helm-charts
Namespace: monitoring
Controller type: Deployment
Replicas: 1
Collection: loki.source.kubernetes
Destination: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
```

## 1. Prerequisites

Complete:

- `Kube-prometheus-stack-tutorial.md`
- `Loki-tutorial.md`

Verify both Applications and the Loki gateway:

```bash
kubectl get applications -n argo-cd \
  kube-prometheus-stack loki

kubectl get pods,services,pvc -n monitoring | grep -E 'grafana|loki'
```

In Grafana, confirm the provisioned Loki datasource exists.

## 2. Add Grafana's Helm repository

Alloy is published in Grafana's main Helm repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo grafana/alloy --versions | head
```

The completed deployment pinned chart `1.11.1`.

## 3. Create the repository files

```bash
mkdir -p apps/alloy
touch apps/alloy/values.yaml
touch argocd/applications/alloy.yaml
```

## 4. Configure Alloy

Create `apps/alloy/values.yaml`:

```yaml
controller:
  type: deployment
  replicas: 1

rbac:
  create: true

alloy:
  configMap:
    create: true
    content: |-
      discovery.kubernetes "pods" {
        role = "pod"
      }

      discovery.relabel "pod_logs" {
        targets = discovery.kubernetes.pods.targets

        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }

        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label  = "pod"
        }

        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }

        rule {
          source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
          target_label  = "app"
        }
      }

      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pod_logs.output
        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
        endpoint {
          url = "http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push"
        }
      }
```

## 5. Understand the Alloy pipeline

The configuration executes in this order:

```text
discovery.kubernetes "pods"
        |
        | discovers pod targets and metadata
        v
discovery.relabel "pod_logs"
        |
        | creates namespace, pod, container, and app labels
        v
loki.source.kubernetes "pods"
        |
        | reads the pod container logs
        v
loki.write "default"
        |
        v
loki-gateway -> Loki
```

The labels make basic LogQL queries readable without copying every Kubernetes label into Loki.

`rbac.create: true` lets the chart create the ServiceAccount and permissions Alloy needs for Kubernetes discovery and log access.

## 6. Create the Argo CD Application

Create `argocd/applications/alloy.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: alloy
  namespace: argo-cd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  sources:
    - repoURL: https://grafana.github.io/helm-charts
      chart: alloy
      targetRevision: 1.11.1
      helm:
        valueFiles:
          - $values/apps/alloy/values.yaml

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

## 7. Render locally

```bash
helm template alloy grafana/alloy \
  --namespace monitoring \
  --version 1.11.1 \
  -f apps/alloy/values.yaml \
  > /tmp/alloy.yaml
```

Verify:

```bash
wc -l /tmp/alloy.yaml
grep '^kind:' /tmp/alloy.yaml
```

The rendered resources should include a Deployment, ServiceAccount, RBAC, ConfigMap, and Service.

Inspect the rendered Alloy configuration:

```bash
grep -A80 'discovery.kubernetes' /tmp/alloy.yaml
```

Confirm the push URL points at `loki-gateway.monitoring.svc.cluster.local`.

## 8. Commit and deploy

```bash
git add apps/alloy/values.yaml argocd/applications/alloy.yaml
git commit -m "add Alloy log collection"
git push
```

Watch Argo CD:

```bash
kubectl get applications -n argo-cd -w
```

Expected:

```text
alloy   Synced   Healthy
```

## 9. Validate the Alloy workload

```bash
kubectl get pods -n monitoring \
  -l app.kubernetes.io/name=alloy

kubectl get deployment,serviceaccount,clusterrole,clusterrolebinding \
  -A | grep -i alloy
```

The Alloy pod should be `Running` and ready.

## 10. Validate Alloy logs

```bash
kubectl logs -n monitoring \
  -l app.kubernetes.io/name=alloy \
  --tail=100
```

A healthy collector starts its components, discovers Kubernetes targets, and writes batches to Loki without repeated permission or push errors.

## 11. Generate a known test log

Create a temporary pod that writes a distinctive line:

```bash
kubectl run alloy-log-test \
  -n default \
  --image=busybox:1.37 \
  --restart=Never \
  -- sh -c 'echo ALLOY_TUTORIAL_LOG_TEST; sleep 60'
```

Confirm the source log:

```bash
kubectl logs alloy-log-test -n default
```

Expected:

```text
ALLOY_TUTORIAL_LOG_TEST
```

Allow Alloy a short reconciliation interval before querying Loki.

## 12. Query logs in Grafana

Open Grafana, then **Explore → Loki**.

Query the temporary pod:

```logql
{namespace="default", pod="alloy-log-test"}
```

The result should contain:

```text
ALLOY_TUTORIAL_LOG_TEST
```

Query monitoring logs:

```logql
{namespace="monitoring"}
```

Query Gitea logs:

```logql
{namespace="gitea"}
```

These queries prove the complete data path, not just Alloy pod health.

## 13. Inspect available labels

In Grafana's Loki query builder, inspect label values for:

```text
namespace
pod
container
app
```

The values should correspond to Kubernetes metadata. Avoid adding every pod label automatically: highly variable labels create excessive Loki stream cardinality.

## 14. Cleanup the test pod

```bash
kubectl delete pod alloy-log-test -n default
```

The already-ingested log remains queryable until Loki retention removes it.

## 15. Restart validation

Delete the Alloy pod:

```bash
kubectl delete pod -n monitoring \
  -l app.kubernetes.io/name=alloy

kubectl get pods -n monitoring -w
```

After the replacement is ready, generate another test log or query recent monitoring logs. Alloy should resume discovery and delivery automatically.

## 16. Argo CD self-heal validation

Patch the live Deployment replica count:

```bash
kubectl scale deployment -n monitoring \
  -l app.kubernetes.io/name=alloy \
  --replicas=2
```

Argo CD should restore the Git-managed count to one. Verify:

```bash
kubectl get deployment -n monitoring \
  -l app.kubernetes.io/name=alloy

kubectl get application alloy -n argo-cd
```

## 17. Ownership and rollback

```text
argocd/applications/alloy.yaml
  -> chart, version, namespace, reconciliation

apps/alloy/values.yaml
  -> Alloy controller, RBAC, discovery, relabeling, and Loki output

apps/loki/values.yaml
  -> log storage and retention

apps/kube-prometheus-stack/values.yaml
  -> Grafana Loki datasource
```

To roll back an Alloy configuration change, revert its Git commit and push. Argo CD will restore the previous ConfigMap and Deployment specification.

If Alloy is unavailable, applications continue running and Loki retains previously ingested logs; only new log collection is interrupted.

## 18. Final architecture

```text
Kubernetes pods
      |
      | API-based log access
      v
Grafana Alloy
      |
      | namespace, pod, container, app labels
      v
loki-gateway
      |
      v
Monolithic Loki -> 10Gi PVC
      |
      v
Grafana Explore and LogQL
```
