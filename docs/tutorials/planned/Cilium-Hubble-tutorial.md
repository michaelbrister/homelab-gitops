# Migrating MicroK8s Networking to Cilium with Hubble

> Status: implementation candidate and high risk. Cilium was not installed in the referenced homelab session. A CNI migration can disconnect every pod and the Kubernetes API. Rehearse this exact procedure on a clone, use console access, and adapt it to the live MicroK8s CNI configuration before execution.

This guide follows Cilium's official migration model: install Cilium initially without taking CNI ownership, select nodes with `CiliumNodeConfig`, recreate pods on the migrated node, remove the old CNI only after connectivity succeeds, and then enable policy enforcement and Hubble.

## 1. Target outcome

- Cilium is the only active pod CNI.
- kube-proxy remains enabled for the first migration.
- MetalLB continues to own `192.168.4.240`.
- Traefik and every private HTTPS endpoint remain reachable.
- Hubble Relay/UI and Prometheus metrics are enabled.
- A test policy produces visible allowed and denied flows.

Keeping kube-proxy and MetalLB for the first migration minimizes simultaneous changes. Kube-proxy replacement or Cilium L2 announcements should be separate, later tutorials.

## 2. Maintenance and recovery prerequisites

- Use physical or out-of-band console access; do not rely only on SSH through the cluster network.
- Take MicroK8s and application backups.
- Record the command that restores the current CNI configuration.
- Rehearse on a comparable single-node MicroK8s cluster.
- Schedule downtime: draining the only node stops ordinary workloads.

Capture the baseline:

```bash
microk8s version
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get daemonset -A
kubectl get networkpolicy -A -o yaml > /tmp/networkpolicies-before-cilium.yaml
kubectl get service -A -o wide > /tmp/services-before-cilium.txt
kubectl get ingress -A -o wide > /tmp/ingresses-before-cilium.txt
kubectl cluster-info dump > /tmp/cluster-info-before-cilium.txt
sudo ls -la /var/snap/microk8s/current/args/cni-network
sudo cp -a /var/snap/microk8s/current/args/cni-network /var/snap/microk8s/current/args/cni-network.pre-cilium
```

Record pod CIDR, service CIDR, API endpoint, current Calico resources, kube-proxy state, MetalLB pool, and Traefik address:

```bash
kubectl get ippools.crd.projectcalico.org -o yaml
kubectl get configmap -n kube-system kube-proxy -o yaml
kubectl get ipaddresspool,l2advertisement -n metallb-system -o yaml
kubectl get service -A | grep 192.168.4.240
```

Checkpoint: backups and the old CNI directory are recoverable from the console.

## 3. Select compatible versions

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm search repo cilium/cilium --versions | head -10
export CILIUM_VERSION="REPLACE_WITH_TESTED_VERSION"
```

Install the matching Cilium CLI using the project's documented method. Confirm that the selected Cilium release supports the live Kubernetes and kernel versions.

## 4. Create initial migration values

Create `apps/cilium/values-migration.yaml`:

```yaml
operator:
  replicas: 1

kubeProxyReplacement: false

routingMode: tunnel
tunnelProtocol: vxlan

cni:
  customConf: true
  uninstall: false

policyEnforcementMode: never

bpf:
  hostLegacyRouting: true

hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
  metrics:
    enableOpenMetrics: true
    enabled:
      - dns
      - drop
      - tcp
      - flow
      - icmp
      - http

prometheus:
  enabled: true

dashboards:
  enabled: true
```

`customConf: true` prevents Cilium from immediately writing the primary CNI configuration. Policy enforcement is disabled during mixed-CNI operation. Review tunnel and CIDR settings against the captured cluster state.

Generate the rendered values and manifests with the Cilium CLI/Helm version selected for the migration:

```bash
cilium install "$CILIUM_VERSION" \
  --values apps/cilium/values-migration.yaml \
  --dry-run-helm-values > /tmp/cilium-values-resolved.yaml
helm template cilium cilium/cilium --version "$CILIUM_VERSION" \
  --namespace kube-system \
  --values /tmp/cilium-values-resolved.yaml > /tmp/cilium-migration.yaml
```

Review capabilities, host mounts, CNI paths, API endpoint, IPAM mode, routing mode, and kube-proxy setting.

## 5. Install the reference CNI plugins

Cilium's migration procedure requires the reference CNI plugins on each node. Download the manifest matching the selected Cilium tag, inspect it, and then apply it server-side. Do not apply a floating `main` or `latest` URL.

```bash
curl -fL "https://raw.githubusercontent.com/cilium/cilium/${CILIUM_VERSION}/examples/misc/migration/install-reference-cni-plugins.yaml" \
  -o /tmp/install-reference-cni-plugins.yaml
less /tmp/install-reference-cni-plugins.yaml
kubectl apply -n kube-system --server-side -f /tmp/install-reference-cni-plugins.yaml
```

Wait for its DaemonSet to complete successfully before continuing.

## 6. Install Cilium in secondary mode

The CNI bootstrap is intentionally performed with Helm, because Argo CD itself depends on working networking. Git still stores the pinned values.

```bash
helm upgrade --install cilium cilium/cilium \
  --version "$CILIUM_VERSION" \
  --namespace kube-system \
  --values /tmp/cilium-values-resolved.yaml
cilium status --wait
```

At this point Cilium should be healthy but manage zero existing pods. Do not remove Calico.

## 7. Create the selective takeover configuration

Create `apps/cilium/cilium-node-config.yaml`:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNodeConfig
metadata:
  name: cilium-default
  namespace: kube-system
spec:
  nodeSelector:
    matchLabels:
      io.cilium.migration/cilium-default: "true"
  defaults:
    write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
    custom-cni-conf: "false"
    cni-chaining-mode: "none"
    cni-exclusive: "true"
```

```bash
kubectl apply --server-side -f apps/cilium/cilium-node-config.yaml
```

Because the node lacks the selector label, Cilium still does not take CNI ownership.

## 8. Migrate the single node

Record the node name:

```bash
export CILIUM_NODE="$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"
kubectl cordon "$CILIUM_NODE"
kubectl drain "$CILIUM_NODE" --ignore-daemonsets --delete-emptydir-data --force
kubectl label node "$CILIUM_NODE" io.cilium.migration/cilium-default=true
```

Restart the node from its console so kubelet and CNI state begin cleanly:

```bash
sudo reboot
```

After the node returns:

```bash
microk8s status --wait-ready
cilium status --wait
kubectl uncordon "$CILIUM_NODE"
kubectl get pods -A -o wide
```

Newly created pod sandboxes should now use Cilium. Wait for every workload to be recreated and ready.

## 9. Validate before removing Calico

```bash
cilium connectivity test
kubectl run dns-test --image=busybox:1.37 --restart=Never --rm -it -- nslookup kubernetes.default.svc.cluster.local
curl -I https://argocd.home.mikebrister.com
curl -I https://gitea.home.mikebrister.com
curl -I https://grafana.home.mikebrister.com
kubectl get service -A | grep 192.168.4.240
kubectl get externalsecret -A
```

Also validate Gitea push, Grafana metrics/logs, OpenBao leadership, ESO sync, cert-manager readiness, Longhorn volumes, and internet egress.

Checkpoint: stop and restore the saved CNI configuration if any critical path fails.

## 10. Remove the old CNI and finalize Cilium

MicroK8s packages and manages Calico details by release. Derive the exact removal commands from the captured live manifests and rehearse them on the clone. Remove only the old Calico controllers, DaemonSets, CNI configuration, and CRDs that are no longer referenced. Do not remove kube-proxy or MetalLB in this tutorial.

After removal, change `apps/cilium/values.yaml` to the final values:

```yaml
operator:
  replicas: 1
kubeProxyReplacement: false
routingMode: tunnel
tunnelProtocol: vxlan
cni:
  customConf: false
  uninstall: false
policyEnforcementMode: default
bpf:
  hostLegacyRouting: false
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
  metrics:
    enableOpenMetrics: true
    enabled: [dns, drop, tcp, flow, icmp, http]
prometheus:
  enabled: true
dashboards:
  enabled: true
```

Upgrade Helm using the final values and rerun connectivity tests. Do not change to kube-proxy replacement during this step.

## 11. Transfer ongoing ownership to Argo CD

Create `argocd/applications/cilium.yaml` with the same release name, namespace, pinned version, and final values source. Initially disable automated pruning. Sync manually, confirm Argo produces no destructive replacement, then enable self-heal. Enable prune only after a later controlled review.

This bootstrap boundary is intentional: Helm performs the network transition; Argo CD owns steady-state configuration afterward.

## 12. Hubble UI and monitoring

Create a private Ingress for the Hubble UI only after the final chart renders the service name. Use:

```text
hubble.home.mikebrister.com
issuer: letsencrypt-prod
secret: hubble-tls
```

Verify Hubble Relay and metrics:

```bash
cilium hubble port-forward &
hubble status
hubble observe --follow
kubectl get servicemonitor -n kube-system
```

## 13. NetworkPolicy proof

Deploy two disposable pods and a Service in `network-test`. Verify connectivity before policy. Add a CiliumNetworkPolicy that allows only the selected client identity, then prove the allowed request succeeds and an unmatched request is denied. Observe both verdicts with `hubble observe`.

Never begin policy testing in `kube-system`, `argo-cd`, `openbao`, or `longhorn-system`.

## 14. Restart and rollback

Reboot the node once more and repeat the full application checklist. Keep the saved Calico manifests and CNI directory through a rollback window.

Rollback requires a console maintenance window: stop workload recreation, remove Cilium CNI ownership using the selected release's uninstall guidance, restore the saved MicroK8s CNI directory and Calico resources, restart MicroK8s, recreate pods, and rerun the baseline tests. Rehearse this before the real cutover.

## 15. Final checkpoint

Mark this guide validated only after Cilium health, DNS, services, ingress, MetalLB, egress, every application, Hubble flows, policy verdicts, monitoring, restart, Argo ownership, and the rehearsed rollback all pass.

## Official references

- [Migrating an existing cluster to Cilium](https://docs.cilium.io/en/stable/installation/k8s-install-migration/)
- [Cilium Helm installation](https://docs.cilium.io/en/stable/installation/k8s-install-helm/)
