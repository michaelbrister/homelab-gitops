# Longhorn on MicroK8s: Installation, Validation, and Safe PVC Migration

> Status: implementation candidate. Longhorn was not installed in the referenced homelab session. Replace every version placeholder, rehearse the backup and restore process, and mark this guide validated only after all checkpoints pass.

This guide adds Longhorn without immediately replacing existing `microk8s-hostpath` volumes. It proves storage on a disposable workload first, then gives a safe migration pattern for one stateful application at a time.

## 1. Target outcome

- Longhorn is managed by Argo CD in `longhorn-system`.
- A single-node StorageClass uses one replica.
- The UI is private and protected by cert-manager.
- Prometheus collects Longhorn metrics.
- Provisioning, restart, snapshot, and restore tests pass.
- Existing hostpath PVCs remain untouched until each application-specific migration is proven.

Warning: one replica on one node improves storage management, snapshots, and portability, but does not protect against node or disk loss. Keep an out-of-cluster backup.

## 2. Prerequisites and inventory

Complete tutorials 1–9. Record the current storage state:

```bash
kubectl get storageclass
kubectl get pvc -A -o wide
kubectl get pv -o custom-columns=NAME:.metadata.name,SC:.spec.storageClassName,CLAIM:.spec.claimRef.namespace/.spec.claimRef.name,RECLAIM:.spec.persistentVolumeReclaimPolicy,STATUS:.status.phase
```

Take application-level backups of Gitea/PostgreSQL, Prometheus, Grafana, Loki, and OpenBao. Do not use a filesystem copy while an application is actively writing unless that application documents it as safe.

On the MicroK8s node, confirm Longhorn's required host facilities:

```bash
sudo systemctl status iscsid
command -v iscsiadm
findmnt --version
lsmod | grep iscsi_tcp
df -h
```

Install and enable missing host prerequisites using the operating system's package manager before proceeding. Reserve a dedicated path, for example `/var/lib/longhorn`, with enough free capacity.

Checkpoint: backups are restorable, the inventory is saved outside the node, iSCSI is active, and adequate disk space exists.

## 3. Select and pin the release

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm search repo longhorn/longhorn --versions | head -10
export LONGHORN_CHART_VERSION="REPLACE_WITH_TESTED_VERSION"
```

Confirm the selected release supports the cluster's Kubernetes version in the Longhorn compatibility documentation. Use the same version in Helm rendering and Argo CD.

## 4. Repository layout

```text
apps/longhorn/
├── values.yaml
└── test/
    ├── pvc.yaml
    └── pod.yaml
argocd/applications/longhorn.yaml
```

Create `apps/longhorn/values.yaml`:

```yaml
defaultSettings:
  defaultReplicaCount: 1
  defaultDataPath: /var/lib/longhorn
  storageMinimalAvailablePercentage: 15

persistence:
  defaultClass: false
  defaultClassReplicaCount: 1
  reclaimPolicy: Retain

longhornUI:
  replicas: 1

metrics:
  serviceMonitor:
    enabled: true

ingress:
  enabled: true
  ingressClassName: traefik
  host: longhorn.home.mikebrister.com
  tls: true
  tlsSecret: longhorn-tls
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

Before committing, render the selected version and verify the exact value names against its generated output. Chart keys can change between releases.

## 5. Create the Argo CD Application

Create `argocd/applications/longhorn.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: longhorn
  namespace: argo-cd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://charts.longhorn.io
      chart: longhorn
      targetRevision: REPLACE_WITH_TESTED_VERSION
      helm:
        releaseName: longhorn
        valueFiles:
          - $values/apps/longhorn/values.yaml
    - repoURL: https://github.com/michaelbrister/homelab-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: longhorn-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

## 6. Render and inspect

```bash
helm template longhorn longhorn/longhorn \
  --version "$LONGHORN_CHART_VERSION" \
  --namespace longhorn-system \
  --values apps/longhorn/values.yaml \
  > /tmp/longhorn.yaml
grep '^kind:' /tmp/longhorn.yaml | sort | uniq -c
grep -n 'privileged: true' /tmp/longhorn.yaml
```

Review RBAC, privileged containers, host mounts, the StorageClass, ingress host, replica count, and data path. Replace both version placeholders with the selected version.

## 7. Deploy and validate controllers

```bash
git add apps/longhorn argocd/applications/longhorn.yaml
git commit -m "Install Longhorn storage"
git push
kubectl get application -n argo-cd longhorn -w
```

Then validate:

```bash
kubectl get pods -n longhorn-system -o wide
kubectl get storageclass longhorn
kubectl get nodes.longhorn.io -n longhorn-system
kubectl get servicemonitor -n longhorn-system
kubectl wait certificate/longhorn-tls -n longhorn-system --for=condition=Ready --timeout=300s
curl -I https://longhorn.home.mikebrister.com
```

Checkpoint: Argo CD is healthy, all required Longhorn components are ready, the node is schedulable in Longhorn, and the private UI uses a trusted certificate.

## 8. Prove a disposable volume

Create `apps/longhorn/test/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test
  namespace: default
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

Create `apps/longhorn/test/pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: longhorn-test
  namespace: default
spec:
  containers:
    - name: test
      image: busybox:1.37
      command: ["sh", "-c", "test -f /data/proof || date -Iseconds > /data/proof; cat /data/proof; sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: longhorn-test
```

Apply the test resources directly because they are temporary validation objects:

```bash
kubectl apply -f apps/longhorn/test/pvc.yaml
kubectl apply -f apps/longhorn/test/pod.yaml
kubectl wait pod/longhorn-test --for=condition=Ready --timeout=180s
kubectl exec longhorn-test -- cat /data/proof
kubectl delete pod longhorn-test
kubectl apply -f apps/longhorn/test/pod.yaml
kubectl wait pod/longhorn-test --for=condition=Ready --timeout=180s
kubectl exec longhorn-test -- cat /data/proof
```

The timestamp must remain identical after pod recreation.

## 9. Prove snapshot and restore

Use the Longhorn UI to create a snapshot of `longhorn-test`. Change `/data/proof`, then restore the snapshot while the test pod is stopped. Reattach the volume and confirm the earlier value returns.

Configure an S3-compatible or NFS backup target before treating Longhorn snapshots as disaster recovery. A snapshot stored on the same node is not an off-node backup. Create a backup, restore it to a new test volume, mount it, and verify the proof file.

Checkpoint: both snapshot restore and off-node backup restore are demonstrated and recorded.

## 10. Migrate one workload safely

Kubernetes cannot change an existing PVC's StorageClass in place. For each workload:

1. choose a low-risk application;
2. record its StatefulSet/Deployment, PVC, mount path, UID/GID, and backup command;
3. stop or quiesce writes;
4. take and verify an application-aware backup;
5. create a new Longhorn PVC with a distinct name;
6. restore into the new PVC using a one-shot restore pod or the application's restore command;
7. update the Git-managed workload to reference the new claim;
8. sync and validate reads, writes, restart, metrics, and ingress;
9. rehearse reverting Git to the old claim;
10. retain the old PVC through the rollback window.

Do not migrate OpenBao by copying its live Raft files. Use a supported Raft snapshot and restore procedure. Use PostgreSQL dump/restore for Gitea's database, and use each observability component's supported backup or rebuild approach.

## 11. Default StorageClass decision

Keep `microk8s-hostpath` as default until all tests pass. If Longhorn should become the default, make the change deliberately and verify only one class has the default annotation:

```bash
kubectl patch storageclass microk8s-hostpath -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass longhorn -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get storageclass
```

Represent this choice in Git or the owning configuration so live drift is not later reversed.

## 12. Restart and self-heal validation

```bash
kubectl delete pod -n longhorn-system -l app=longhorn-manager
kubectl get pods -n longhorn-system -w
kubectl exec longhorn-test -- cat /data/proof
```

Also reboot the single node during a maintenance window, then verify Longhorn, the test volume, all migrated workloads, and Argo CD health.

## 13. Cleanup and rollback

Delete the disposable pod and PVC only after backup/restore validation:

```bash
kubectl delete -f apps/longhorn/test/pod.yaml
kubectl delete -f apps/longhorn/test/pvc.yaml
```

Rollback one migrated workload by stopping writes, restoring the last compatible data if needed, and reverting Git to its old hostpath claim. Never uninstall Longhorn while any bound PV still depends on it.

## 14. Final checkpoint

Mark this tutorial validated only after provisioning, remount, snapshot restore, off-node backup restore, node restart, monitoring, HTTPS, self-heal, one workload migration, and that workload's rollback rehearsal all pass.

## Official references

- [Longhorn Helm installation](https://longhorn.io/docs/1.12.1/deploy/install/install-with-helm/)
- [Longhorn best practices](https://longhorn.io/docs/1.12.1/best-practices/)
