# Tutorial 5: Install and Verify Observability

The observability stack has three child Applications:

- `kube-prometheus-stack` collects metrics and runs Grafana;
- Loki stores logs in single-binary mode; and
- Alloy discovers pod logs and forwards them to Loki.

All three deploy into the `monitoring` namespace.

## Review storage before syncing

The current values target a single-node lab and request persistent volumes from
`microk8s-hostpath`:

- Prometheus: 10 GiB
- Alertmanager: 2 GiB
- Grafana: 5 GiB
- Loki: 10 GiB

Confirm that the storage class exists and that the node has enough free space:

```bash
kubectl get storageclass microk8s-hostpath
kubectl get nodes
```

For a multi-node cluster, use a storage backend with appropriate scheduling and
failure semantics instead of host-local volumes.

## Verify reconciliation

```bash
kubectl get applications --namespace argo-cd \
  kube-prometheus-stack loki alloy
kubectl get pods,persistentvolumeclaims --namespace monitoring
```

Wait for deployments and stateful workloads to settle. If a PVC is Pending,
describe it before troubleshooting the pod:

```bash
kubectl describe persistentvolumeclaim --namespace monitoring
kubectl get events --namespace monitoring --sort-by=.lastTimestamp
```

## Verify metrics

List the services to discover the chart-generated names:

```bash
kubectl get services --namespace monitoring
```

Grafana is configured for
`https://grafana.home.mikebrister.com`. If DNS or TLS is not ready, port-forward
the Grafana service shown by the previous command:

```bash
kubectl port-forward --namespace monitoring service/kube-prometheus-stack-grafana 3000:80
```

Open `http://127.0.0.1:3000`. Retrieve the generated admin password without
printing any other Secret fields:

```bash
kubectl get secret --namespace monitoring kube-prometheus-stack-grafana \
  --output jsonpath='{.data.admin-password}' | base64 --decode
echo
```

## Verify logs

Alloy discovers Kubernetes pods, attaches namespace, pod, container, and app
labels, then writes to Loki's in-cluster gateway.

Check both ends of the pipeline:

```bash
kubectl logs --namespace monitoring deployment/alloy --tail=100
kubectl get pods --namespace monitoring --selector=app.kubernetes.io/name=loki
```

In Grafana, open **Explore**, select the Loki data source, and start with:

```logql
{namespace="monitoring"}
```

Narrow the query with the `pod`, `container`, or `app` labels added by Alloy.

## Tune retention and capacity

Prometheus and Loki are both configured for seven days of retention. Change
retention and volume sizes together: a longer retention period without enough
disk space will eventually make the workloads unhealthy.

After any values change, watch Argo CD and confirm the PVC behavior. Most
storage classes can expand volumes but cannot shrink them.

## Troubleshooting sequence

1. Check the three Argo CD Application health states.
2. Check Pending PVCs and namespace events.
3. Check Alloy logs for discovery or write errors.
4. Confirm the `loki-gateway` service exists in `monitoring`.
5. Confirm Grafana's Loki data source points to the in-cluster gateway.
