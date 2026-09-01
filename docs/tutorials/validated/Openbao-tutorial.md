# OpenBao on MicroK8s with Argo CD, Static Auto-Unseal, ESO, and Raft

This tutorial reproduces the completed homelab setup in the order it was successfully implemented:

1. Install OpenBao through Helm and Argo CD with persistent file storage.
2. Initialize it with Shamir shares.
3. Configure a static seal and migrate Shamir to static auto-unseal.
4. Configure KV v2 and Kubernetes authentication for External Secrets Operator (ESO).
5. Move the Cloudflare API token into OpenBao.
6. Transfer the configuration to Argo CD ownership and validate self-healing.
7. Migrate the deprecated file backend to integrated Raft.
8. Cut over to single-node HA/Raft mode and validate the complete system.

The completed environment used:

- MicroK8s with `microk8s-hostpath`
- OpenBao Helm chart `0.29.3`
- OpenBao `2.6.2`
- External Secrets Operator `2.8.0`
- one OpenBao Raft voter
- Argo CD automated sync, pruning, and self-healing

> **Homelab warning:** The static seal key is stored in the same Kubernetes cluster as the encrypted OpenBao data. This enables unattended restarts but is not a strong independent root of trust. Use a KMS, HSM, or separately secured seal mechanism for production or valuable data.

> **Never commit secrets:** Keep the static seal key, recovery shares, root token, and Cloudflare token out of Git. Maintain secure copies outside the cluster.

## 1. Repository layout

The permanent GitOps files use this layout:

```text
apps/
├── openbao/
│   └── values.yaml
├── openbao-config/
│   ├── kustomization.yaml
│   ├── tokenreview-rbac.yaml
│   └── eso-serviceaccount.yaml
└── external-secrets-config/
    ├── kustomization.yaml
    ├── openbao-store.yaml
    └── cloudflare-token.yaml

argocd/applications/
├── openbao.yaml
├── openbao-config.yaml
└── external-secrets-config.yaml
```

The later storage migration temporarily uses:

```text
apps/openbao/migrate.hcl
apps/openbao/migration-pod.yaml
apps/openbao/raft-migration-pvc.yaml
```

These migration resources are applied manually and are not added to an Argo CD Application or Kustomization.

## 2. Install OpenBao through Argo CD

### 2.1 Create the initial Helm values

Create `apps/openbao/values.yaml`:

```yaml
server:
  standalone:
    enabled: true

    config: |
      ui = true

      listener "tcp" {
        address         = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_disable     = 1
      }

      storage "file" {
        path = "/openbao/data"
      }

  dataStorage:
    enabled: true
    storageClass: microk8s-hostpath
    size: 10Gi

  readinessProbe:
    enabled: true
    path: /v1/sys/health?uninitcode=204&sealedcode=204

ui:
  enabled: true

injector:
  enabled: false
```

ESO will deliver secrets, so the OpenBao Agent Injector is disabled.

### 2.2 Create the Argo CD Application

Create `argocd/applications/openbao.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: openbao
  namespace: argo-cd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  sources:
    - repoURL: https://openbao.github.io/openbao-helm
      chart: openbao
      targetRevision: 0.29.3
      helm:
        valueFiles:
          - $values/apps/openbao/values.yaml

    - repoURL: https://github.com/michaelbrister/homelab-gitops.git
      targetRevision: main
      ref: values

  destination:
    server: https://kubernetes.default.svc
    namespace: openbao

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 2.3 Render, commit, and deploy

```bash
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update

helm template openbao openbao/openbao \
  --namespace openbao \
  --version 0.29.3 \
  -f apps/openbao/values.yaml \
  > /tmp/openbao.yaml
```

Commit and push:

```bash
git add apps/openbao/values.yaml argocd/applications/openbao.yaml
git commit -m "add OpenBao"
git push
```

Watch Argo CD and the OpenBao pod:

```bash
kubectl get applications -n argo-cd
kubectl get pods -n openbao -w
```

### Checkpoint: uninitialized OpenBao

```bash
kubectl exec -n openbao openbao-0 -- bao status
```

Expected state:

```text
Seal Type       shamir
Initialized     false
Sealed          true
Storage Type    file
HA Enabled      false
```

## 3. Initialize OpenBao with Shamir

Initialize with five shares and a threshold of three:

```bash
kubectl exec -it -n openbao openbao-0 -- \
  bao operator init \
  -key-shares=5 \
  -key-threshold=3
```

Store all five shares and the initial root token outside Kubernetes and outside Git.

Unseal OpenBao by running this command three times and entering a different share at each hidden prompt:

```bash
kubectl exec -it -n openbao openbao-0 -- bao operator unseal
```

Verify:

```bash
kubectl exec -n openbao openbao-0 -- bao status
```

Expected:

```text
Initialized     true
Sealed          false
Storage Type    file
```

## 4. Configure static auto-unseal

### 4.1 Create the static key Secret

Generate exactly 32 raw bytes:

```bash
openssl rand -out /tmp/openbao-unseal.key 32
```

Back up this file securely outside the cluster. Then create the Kubernetes Secret:

```bash
kubectl create secret generic openbao-unseal \
  -n openbao \
  --from-file=unseal.key=/tmp/openbao-unseal.key
```

Verify the stored key length without printing it:

```bash
kubectl get secret openbao-unseal -n openbao \
  -o go-template='{{index .data "unseal.key"}}' \
  | base64 --decode \
  | wc -c
```

The result must be `32`. After confirming the external backup exists, remove the temporary copy:

```bash
rm /tmp/openbao-unseal.key
```

### 4.2 Add the Secret mount and static seal

Update `apps/openbao/values.yaml`:

```yaml
server:
  standalone:
    enabled: true

    config: |
      ui = true

      listener "tcp" {
        address         = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_disable     = 1
      }

      storage "file" {
        path = "/openbao/data"
      }

      seal "static" {
        current_key_id = "homelab-1"
        current_key    = "file:///openbao/secrets/unseal.key"
      }

  dataStorage:
    enabled: true
    storageClass: microk8s-hostpath
    size: 10Gi

  volumes:
    - name: openbao-unseal
      secret:
        secretName: openbao-unseal
        defaultMode: 0400

  volumeMounts:
    - name: openbao-unseal
      mountPath: /openbao/secrets
      readOnly: true

  readinessProbe:
    enabled: true
    path: /v1/sys/health?uninitcode=204&sealedcode=204

ui:
  enabled: true

injector:
  enabled: false
```

Render and verify the mount and seal configuration:

```bash
helm template openbao openbao/openbao \
  --namespace openbao \
  --version 0.29.3 \
  -f apps/openbao/values.yaml \
  > /tmp/openbao-static.yaml

grep -A15 -B5 'openbao-unseal' /tmp/openbao-static.yaml
grep -A5 -B2 'seal "static"' /tmp/openbao-static.yaml
```

Commit and push:

```bash
git add apps/openbao/values.yaml
git commit -m "configure OpenBao static auto-unseal"
git push
```

Wait for Argo CD to sync. The StatefulSet uses an `OnDelete` update strategy, so recreate the pod:

```bash
kubectl delete pod openbao-0 -n openbao
kubectl get pods -n openbao -w
```

Verify the mounted key:

```bash
kubectl exec -n openbao openbao-0 -- \
  wc -c /openbao/secrets/unseal.key
```

Expected result:

```text
32 /openbao/secrets/unseal.key
```

### 4.3 Migrate Shamir to the static seal

Submit the original Shamir shares with `-migrate`. The flag must be present for every share:

```bash
kubectl exec -it -n openbao openbao-0 -- \
  bao operator unseal -migrate
```

Run the command three times, entering a different original share each time.

Verify:

```bash
kubectl exec -n openbao openbao-0 -- bao status
```

Expected:

```text
Seal Type             static
Recovery Seal Type    shamir
Initialized           true
Sealed                false
Storage Type          file
```

The former Shamir shares now serve as recovery shares.

### Checkpoint: automatic unseal

```bash
kubectl delete pod openbao-0 -n openbao
kubectl get pods -n openbao -w
kubectl exec -n openbao openbao-0 -- bao status
```

The replacement must return as `Initialized=true`, `Sealed=false`, and `Seal Type=static` without manual unseal commands.

## 5. Configure KV v2 and Kubernetes authentication

Log in:

```bash
kubectl exec -it -n openbao openbao-0 -- bao login
```

Enable KV v2 at `secret/`:

```bash
kubectl exec -it -n openbao openbao-0 -- \
  bao secrets enable -path=secret kv-v2

kubectl exec -n openbao openbao-0 -- bao secrets list
```

Enable Kubernetes authentication:

```bash
kubectl exec -it -n openbao openbao-0 -- \
  bao auth enable kubernetes
```

Configure it from inside the pod:

```bash
kubectl exec -it -n openbao openbao-0 -- /bin/sh

bao login

bao write auth/kubernetes/config \
  kubernetes_host="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"

exit
```

OpenBao uses its in-cluster service-account token and CA when validating Kubernetes service-account JWTs.

## 6. Create TokenReview RBAC and an ESO service account

Create `apps/openbao-config/tokenreview-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openbao-tokenreview
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: openbao
    namespace: openbao
```

Create `apps/openbao-config/eso-serviceaccount.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-openbao
  namespace: external-secrets
```

Create `apps/openbao-config/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - tokenreview-rbac.yaml
  - eso-serviceaccount.yaml
```

Apply the bootstrap configuration:

```bash
kubectl apply -k apps/openbao-config

kubectl get clusterrolebinding openbao-tokenreview
kubectl get serviceaccount external-secrets-openbao \
  -n external-secrets
```

## 7. Create the ESO policy and OpenBao role

Enter the OpenBao pod and log in:

```bash
kubectl exec -it -n openbao openbao-0 -- /bin/sh
bao login
```

Create a read-only KV v2 policy:

```bash
cat >/tmp/external-secrets.hcl <<'EOF'
path "secret/data/*" {
  capabilities = ["read"]
}

path "secret/metadata/*" {
  capabilities = ["read", "list"]
}
EOF

bao policy write external-secrets /tmp/external-secrets.hcl
bao policy read external-secrets
```

Bind that policy to the dedicated service account:

```bash
bao write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets-openbao \
  bound_service_account_namespaces=external-secrets \
  policies=external-secrets \
  ttl=1h

bao read auth/kubernetes/role/external-secrets
```

Create a test secret:

```bash
printf 'Test secret password: '
read -r -s OPENBAO_TEST_PASSWORD
printf '\n'

bao kv put secret/homelab/test \
  username=testuser \
  password="$OPENBAO_TEST_PASSWORD"
unset OPENBAO_TEST_PASSWORD

bao kv get secret/homelab/test
exit
```

The authentication chain is now:

```text
external-secrets/external-secrets-openbao
                  |
                  v
auth/kubernetes/role/external-secrets
                  |
                  v
policy external-secrets
                  |
                  v
read secret/data/* and list secret/metadata/*
```

## 8. Configure ESO 2.8.0 for OpenBao

ESO 2.8.0 uses its Vault-compatible provider for this Kubernetes-authentication setup. OpenBao accepts the compatible API calls, while the Vault provider supplies the required `auth.kubernetes` fields.

Create `apps/external-secrets-config/openbao-store.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: openbao
spec:
  provider:
    vault:
      server: http://openbao.openbao.svc.cluster.local:8200
      path: secret
      version: v2

      auth:
        kubernetes:
          mountPath: kubernetes
          role: external-secrets
          serviceAccountRef:
            name: external-secrets-openbao
            namespace: external-secrets
```

Create `apps/external-secrets-config/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - openbao-store.yaml
```

Apply it:

```bash
kubectl apply -k apps/external-secrets-config
kubectl get clustersecretstore openbao
```

Expected state:

```text
NAME      STATUS   READY
openbao   Valid    True
```

## 9. Validate ESO with a temporary ExternalSecret

Create `apps/external-secrets-config/test-secret.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: openbao-test
  namespace: default
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: openbao
    kind: ClusterSecretStore

  target:
    name: openbao-test
    creationPolicy: Owner

  data:
    - secretKey: username
      remoteRef:
        key: homelab/test
        property: username

    - secretKey: password
      remoteRef:
        key: homelab/test
        property: password
```

Temporarily add it to the Kustomization:

```yaml
resources:
  - openbao-store.yaml
  - test-secret.yaml
```

Apply and verify:

```bash
kubectl apply -k apps/external-secrets-config

kubectl get externalsecret openbao-test -n default
kubectl get secret openbao-test -n default

kubectl get secret openbao-test -n default \
  -o jsonpath='{.data.username}' \
  | base64 --decode
echo
```

The decoded value should be `testuser`.

Remove `test-secret.yaml` from the Kustomization and delete the temporary ExternalSecret:

```bash
kubectl delete externalsecret openbao-test -n default
```

## 10. Move the Cloudflare token into OpenBao

### 10.1 Store the token in OpenBao

```bash
kubectl exec -it -n openbao openbao-0 -- /bin/sh
bao login
```

Read the token without echoing it and write it to KV v2:

```bash
read -s CLOUDFLARE_TOKEN
bao kv put secret/homelab/cloudflare api-token="$CLOUDFLARE_TOKEN"
unset CLOUDFLARE_TOKEN

bao kv metadata get secret/homelab/cloudflare
exit
```

### 10.2 Create the permanent ExternalSecret

Create `apps/external-secrets-config/cloudflare-token.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: openbao
    kind: ClusterSecretStore

  target:
    name: cloudflare-api-token
    creationPolicy: Owner

  data:
    - secretKey: api-token
      remoteRef:
        key: homelab/cloudflare
        property: api-token
```

Set the permanent Kustomization:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - openbao-store.yaml
  - cloudflare-token.yaml
```

### 10.3 Transfer Secret ownership to ESO

Delete the manually created Kubernetes Secret, then immediately apply the ESO configuration:

```bash
kubectl delete secret cloudflare-api-token -n cert-manager
kubectl apply -k apps/external-secrets-config
```

Verify:

```bash
kubectl get externalsecret cloudflare-api-token -n cert-manager
kubectl get secret cloudflare-api-token -n cert-manager
kubectl get clusterissuer
```

The `ExternalSecret` should report `SecretSynced=True`. cert-manager continues to consume the same Secret name and `api-token` key.

## 11. Transfer configuration ownership to Argo CD

### 11.1 Create the configuration Applications

Create `argocd/applications/openbao-config.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: openbao-config
  namespace: argo-cd
spec:
  project: default
  source:
    repoURL: https://github.com/michaelbrister/homelab-gitops.git
    targetRevision: main
    path: apps/openbao-config
  destination:
    server: https://kubernetes.default.svc
    namespace: openbao
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

Create `argocd/applications/external-secrets-config.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets-config
  namespace: argo-cd
spec:
  project: default
  source:
    repoURL: https://github.com/michaelbrister/homelab-gitops.git
    targetRevision: main
    path: apps/external-secrets-config
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

### 11.2 Render, review, and commit

```bash
kubectl kustomize apps/openbao-config
kubectl kustomize apps/external-secrets-config

git diff
git status
```

Confirm that the diff contains secret references only, never secret values.

```bash
git add \
  argocd/applications/openbao-config.yaml \
  argocd/applications/external-secrets-config.yaml \
  apps/openbao-config \
  apps/external-secrets-config

git commit -m "manage OpenBao and external secrets config with Argo CD"
git push
```

Wait for both Applications:

```bash
kubectl get applications -n argo-cd \
  openbao-config external-secrets-config
```

Both should become `Synced` and `Healthy`.

### 11.3 Validate Argo CD self-healing

Change a Git-managed live field:

```bash
kubectl patch externalsecret cloudflare-api-token \
  -n cert-manager \
  --type merge \
  -p '{"spec":{"refreshInterval":"5m"}}'
```

Argo CD should restore the Git value of `1h`:

```bash
kubectl get externalsecret cloudflare-api-token \
  -n cert-manager \
  -o jsonpath='{.spec.refreshInterval}'
echo
```

### 11.4 Validate ESO recovery

Delete only the generated Kubernetes Secret:

```bash
kubectl delete secret cloudflare-api-token -n cert-manager
kubectl get secret cloudflare-api-token -n cert-manager -w
```

ESO should recreate it from OpenBao.

The final ownership model is:

```text
Argo CD -> ExternalSecret and ClusterSecretStore declarations
ESO     -> generated Kubernetes Secrets
OpenBao -> sensitive values
```

## 12. Migrate file storage to integrated Raft

This is an offline migration. OpenBao must remain stopped while `bao operator migrate` copies the storage keys.

### 12.1 Verify the current system

```bash
kubectl exec -n openbao openbao-0 -- bao status
kubectl get clustersecretstore openbao
kubectl get externalsecret cloudflare-api-token -n cert-manager

kubectl get pod openbao-0 -n openbao \
  -o jsonpath='{range .spec.containers[0].volumeMounts[*]}{.name}{" => "}{.mountPath}{"\n"}{end}'

kubectl get pod openbao-0 -n openbao \
  -o jsonpath='{range .spec.volumes[*]}{.name}{" => "}{.persistentVolumeClaim.claimName}{"\n"}{end}'

kubectl get pvc -n openbao
```

The completed environment used:

```text
data-openbao-0 -> /openbao/data
10Gi RWO, microk8s-hostpath
```

### 12.2 Pause OpenBao reconciliation and stop the server

Explicitly disable automated sync for the OpenBao Application:

```bash
kubectl patch application openbao \
  -n argo-cd \
  --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"enabled":false,"prune":true,"selfHeal":true}}}}'
```

Verify:

```bash
kubectl get application openbao -n argo-cd \
  -o jsonpath='{.spec.syncPolicy.automated}'
echo
```

Scale OpenBao to zero:

```bash
kubectl scale statefulset openbao -n openbao --replicas=0
kubectl get pods -n openbao -w
```

Confirm the server is fully stopped:

```bash
kubectl get statefulset openbao -n openbao \
  -o jsonpath='{.spec.replicas}'
echo

kubectl get pods -n openbao
```

The replica count must be `0`, with no `openbao-0` pod.

### 12.3 Create the temporary Raft PVC

Create `apps/openbao/raft-migration-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openbao-raft-migration
  namespace: openbao
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: microk8s-hostpath
  resources:
    requests:
      storage: 10Gi
```

Apply it:

```bash
kubectl apply -f apps/openbao/raft-migration-pvc.yaml
```

### 12.4 Create the migration configuration

Create `apps/openbao/migrate.hcl`:

```hcl
storage_source "file" {
  path = "/source"
}

storage_destination "raft" {
  path    = "/destination"
  node_id = "openbao-0"
}

cluster_addr = "https://openbao-0.openbao-internal:8201"
```

Create the ConfigMap:

```bash
kubectl create configmap openbao-migration-config \
  -n openbao \
  --from-file=migrate.hcl=apps/openbao/migrate.hcl \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

Verify it:

```bash
kubectl get configmap openbao-migration-config \
  -n openbao \
  -o jsonpath='{.data.migrate\.hcl}'
echo
```

### 12.5 Create the migration pod

Create `apps/openbao/migration-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: openbao-migrate
  namespace: openbao
spec:
  restartPolicy: Never

  containers:
    - name: migrate
      image: quay.io/openbao/openbao:2.6.2
      command:
        - /bin/sh
        - -c
        - |
          echo "OpenBao migration pod ready"
          sleep infinity

      volumeMounts:
        - name: source
          mountPath: /source

        - name: destination
          mountPath: /destination

        - name: migration-config
          mountPath: /openbao-migration
          readOnly: true

  volumes:
    - name: source
      persistentVolumeClaim:
        claimName: data-openbao-0

    - name: destination
      persistentVolumeClaim:
        claimName: openbao-raft-migration

    - name: migration-config
      configMap:
        name: openbao-migration-config
```

The source mount is writable because the migration command creates a temporary migration lock in the source backend.

Apply the pod and wait for it:

```bash
kubectl apply -f apps/openbao/migration-pod.yaml

kubectl wait \
  --for=condition=Ready \
  pod/openbao-migrate \
  -n openbao \
  --timeout=60s

kubectl get pvc -n openbao
```

Both PVCs should now be `Bound`.

### 12.6 Verify the offline migration environment

```bash
kubectl exec -n openbao openbao-migrate -- ls -la /source
kubectl exec -n openbao openbao-migrate -- ls -la /destination

kubectl exec -n openbao openbao-migrate -- \
  cat /openbao-migration/migrate.hcl

kubectl exec -n openbao openbao-migrate -- \
  sh -c 'test -w /source/core && echo "source core is writable"'
```

The source should contain the existing backend directories:

```text
auth
core
file
logical
logs
sys
```

The destination should be empty, and the source should be writable.

### 12.7 Run the file-to-Raft migration

```bash
kubectl exec -it -n openbao openbao-migrate -- \
  bao operator migrate \
  -config=/openbao-migration/migrate.hcl
```

Wait for:

```text
Success! All of the keys have been migrated.
```

### 12.8 Copy the migrated Raft data to the permanent PVC

Create a dedicated Raft directory on `data-openbao-0` and copy the completed migration output into it:

```bash
kubectl exec -n openbao openbao-migrate -- mkdir -p /source/raft

kubectl exec -n openbao openbao-migrate -- \
  sh -c 'cp -R /destination/. /source/raft/'
```

The permanent PVC now has this layout:

```text
data-openbao-0
├── auth/                 # retained file backend
├── core/
├── logical/
├── sys/
└── raft/                 # integrated Raft backend
    ├── vault.db
    └── raft/
        ├── raft.db
        └── snapshots/
```

Verify the copy:

```bash
kubectl exec -n openbao openbao-migrate -- \
  sh -c 'echo "=== destination ==="; ls -lahR /destination; echo "=== permanent copy ==="; ls -lahR /source/raft'
```

Compare SHA-256 hashes:

```bash
kubectl exec -n openbao openbao-migrate -- \
  sh -c '
    sha256sum /destination/vault.db /source/raft/vault.db
    sha256sum /destination/raft/raft.db /source/raft/raft/raft.db
  '
```

Each pair must match exactly.

Delete the migration pod, but keep both PVCs:

```bash
kubectl delete pod openbao-migrate -n openbao
kubectl get pvc -n openbao
```

## 13. Cut over Helm to single-node HA and Raft

### 13.1 Replace the runtime values

Replace `apps/openbao/values.yaml` with:

```yaml
server:
  standalone:
    enabled: false

  ha:
    enabled: true
    replicas: 1

    raft:
      enabled: true
      setNodeId: true

      config: |
        ui = true

        listener "tcp" {
          address         = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_disable     = 1
        }

        storage "raft" {
          path    = "/openbao/data/raft"
          node_id = "openbao-0"
        }

        cluster_addr = "https://openbao-0.openbao-internal:8201"

        seal "static" {
          current_key_id = "homelab-1"
          current_key    = "file:///openbao/secrets/unseal.key"
        }

  dataStorage:
    enabled: true
    storageClass: microk8s-hostpath
    size: 10Gi

  volumes:
    - name: openbao-unseal
      secret:
        secretName: openbao-unseal
        defaultMode: 0400

  volumeMounts:
    - name: openbao-unseal
      mountPath: /openbao/secrets
      readOnly: true

  readinessProbe:
    enabled: true
    path: /v1/sys/health?uninitcode=204&sealedcode=204

ui:
  enabled: true

injector:
  enabled: false
```

The critical changes are:

```text
standalone.enabled: true -> false
ha.enabled: false        -> true
ha.replicas              -> 1
storage "file"           -> storage "raft"
Raft path                -> /openbao/data/raft
```

### 13.2 Render and verify

```bash
helm template openbao openbao/openbao \
  --namespace openbao \
  --version 0.29.3 \
  -f apps/openbao/values.yaml \
  > /tmp/openbao-raft.yaml

grep -n 'storage "' apps/openbao/values.yaml
grep -n 'storage "' /tmp/openbao-raft.yaml
grep -A15 -B5 'openbao-unseal' /tmp/openbao-raft.yaml
```

Only `storage "raft"` should appear in the OpenBao server configuration, and the static key mount must remain present.

### 13.3 Commit, refresh, and resume Argo CD

```bash
git add apps/openbao/values.yaml
git commit -m "migrate OpenBao storage to raft"
git push origin main
```

Force a repository refresh:

```bash
kubectl annotate application openbao \
  -n argo-cd \
  argocd.argoproj.io/refresh=hard \
  --overwrite
```

Re-enable automated sync:

```bash
kubectl patch application openbao \
  -n argo-cd \
  --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"enabled":true,"prune":true,"selfHeal":true}}}}'
```

Watch the Application and pod:

```bash
kubectl get application openbao -n argo-cd -w
kubectl get pods -n openbao -w
```

Verify the live OpenBao configuration:

```bash
kubectl get configmap openbao-config \
  -n openbao \
  -o jsonpath='{.data.extraconfig-from-values\.hcl}'
echo
```

It should contain:

```hcl
storage "raft" {
  path    = "/openbao/data/raft"
  node_id = "openbao-0"
}
```

## 14. Validate the final system

### 14.1 Validate OpenBao and auto-unseal

```bash
kubectl exec -n openbao openbao-0 -- bao status
```

Expected fields:

```text
Seal Type              static
Recovery Seal Type     shamir
Initialized            true
Sealed                 false
Storage Type           raft
HA Enabled             true
Raft Committed Index   <number>
Raft Applied Index     <same number>
```

### 14.2 Validate Raft leadership

Authenticate, then list the peers:

```bash
kubectl exec -it -n openbao openbao-0 -- bao login

kubectl exec -n openbao openbao-0 -- \
  bao operator raft list-peers
```

Expected:

```text
Node        Address                             State     Voter
openbao-0   openbao-0.openbao-internal:8201     leader    true
```

### 14.3 Validate migrated data

```bash
kubectl exec -it -n openbao openbao-0 -- \
  bao kv get secret/homelab/test

kubectl exec -it -n openbao openbao-0 -- \
  bao kv metadata get secret/homelab/cloudflare
```

Using metadata for the Cloudflare path avoids printing the token.

### 14.4 Validate ESO

```bash
kubectl get clustersecretstore openbao
kubectl get externalsecret cloudflare-api-token -n cert-manager
```

Expected:

```text
ClusterSecretStore: Valid, Ready=True
ExternalSecret:     SecretSynced, Ready=True
```

### 14.5 Validate a complete restart

```bash
kubectl delete pod openbao-0 -n openbao
kubectl get pods -n openbao -w

kubectl exec -n openbao openbao-0 -- bao status
kubectl exec -it -n openbao openbao-0 -- bao login
kubectl exec -n openbao openbao-0 -- bao operator raft list-peers
kubectl get clustersecretstore openbao
kubectl get externalsecret cloudflare-api-token -n cert-manager
```

The final result should confirm:

- OpenBao restarted automatically.
- The static seal unsealed it without manual shares.
- Storage remained `raft`.
- `openbao-0` returned as the Raft leader and voter.
- ESO remained authenticated and continued syncing the Cloudflare token.

## 15. Cleanup and rollback notes

### Cleanup

Keep the temporary PVC and old file-backend directories through an observation window and at least one successful restart.

Afterward:

```bash
kubectl delete configmap openbao-migration-config -n openbao
kubectl delete pvc openbao-raft-migration -n openbao
```

Remove or archive the temporary repository files:

```text
apps/openbao/migrate.hcl
apps/openbao/migration-pod.yaml
apps/openbao/raft-migration-pvc.yaml
```

Retain the old file-backend directories on `data-openbao-0` longer than the temporary PVC. Do not delete directories from the permanent PVC until you have a separate backup and are certain they are no longer required.

### Rollback

During the observation window:

1. Disable automated sync for the OpenBao Application.
2. Scale the OpenBao StatefulSet to zero.
3. Restore the previous standalone/file-backed `values.yaml` in Git.
4. Sync the previous configuration.
5. Confirm the live ConfigMap uses `storage "file"` at `/openbao/data`.
6. Start OpenBao and validate it before restoring automated reconciliation.

> The retained file backend is a point-in-time rollback copy. Changes written after the Raft cutover will not exist there. Once Raft has accepted new data, use a Raft snapshot and tested restore process rather than relying on the old file backend.

## 16. Final architecture

```text
GitHub
   |
   v
Argo CD
   |-- OpenBao Helm release
   |-- TokenReview RBAC and ESO service account
   `-- ClusterSecretStore and ExternalSecret declarations

MicroK8s
   |
   |-- OpenBao 2.6.2
   |     |-- static key from openbao/openbao-unseal
   |     |-- integrated Raft at /openbao/data/raft
   |     |-- Kubernetes auth role external-secrets
   |     `-- KV v2 at secret/
   |
   |-- External Secrets Operator 2.8.0
   |     `-- Vault-compatible provider -> OpenBao
   |
   `-- cert-manager
         `-- consumes cert-manager/cloudflare-api-token
```

After a homelab restart, MicroK8s starts OpenBao, the static key auto-unseals it, Raft becomes active, ESO authenticates through Kubernetes, and the declared Kubernetes Secrets continue to reconcile.
