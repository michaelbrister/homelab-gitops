# Tutorial 6: Deliver Secrets with OpenBao and External Secrets

OpenBao stores secret values; External Secrets Operator (ESO) authenticates to
OpenBao with a Kubernetes service account and materializes selected values as
Kubernetes Secrets.

This tutorial includes security-sensitive bootstrap commands. Run them from a
trusted workstation, keep generated key material out of the repository, and
save recovery material in an encrypted offline location.

## Understand the data path

```text
OpenBao KV v2 secret
  -> Kubernetes-auth role and policy
     -> ClusterSecretStore/openbao
        -> ExternalSecret
           -> namespaced Kubernetes Secret
```

The Argo CD Applications install the OpenBao and ESO controllers. The
`openbao-config` Application creates ESO's service account and the token-review
RBAC binding. The authentication backend, policy, role, and secret values are
intentionally bootstrapped outside Git.

## Create the static seal Secret

`apps/openbao/values.yaml` mounts a Secret named `openbao-unseal` and reads the
key from `unseal.key`. Create a 32-byte key encoded as base64 in a temporary
directory outside the repository:

```bash
OPENBAO_KEY_DIR="$(mktemp -d)"
openssl rand -base64 32 > "$OPENBAO_KEY_DIR/unseal.key"
chmod 600 "$OPENBAO_KEY_DIR/unseal.key"
kubectl create namespace openbao --dry-run=client --output yaml | kubectl apply --filename -
kubectl create secret generic openbao-unseal \
  --namespace openbao \
  --from-file=unseal.key="$OPENBAO_KEY_DIR/unseal.key"
```

Copy the key to encrypted offline storage, then securely remove the temporary
copy using the method appropriate for your workstation. Losing this key makes
the OpenBao data unreadable. Replacing it without a planned seal migration can
also make the data unreadable.

## Wait for the controllers

```bash
kubectl get applications --namespace argo-cd openbao external-secrets openbao-config
kubectl get pods --namespace openbao
kubectl get pods --namespace external-secrets
```

Initialize OpenBao once if the new instance reports that it is not initialized:

```bash
kubectl exec --namespace openbao openbao-0 -- \
  bao status -address=http://127.0.0.1:8200
kubectl exec --interactive --tty --namespace openbao openbao-0 -- \
  bao operator init -address=http://127.0.0.1:8200
```

Capture the initialization output directly into encrypted offline storage. Do
not paste the root token or recovery material into Git, a ticket, or shell
history. If the instance is already initialized, do not initialize it again.

## Configure Kubernetes authentication

Open a shell in the OpenBao pod:

```bash
kubectl exec --interactive --tty --namespace openbao openbao-0 -- sh
```

Inside the pod, set the local address and authenticate without placing the root
token in command history:

```bash
export BAO_ADDR=http://127.0.0.1:8200
bao login
```

Enable a KV v2 mount and Kubernetes authentication if they do not already
exist:

```bash
bao secrets enable -path=secret kv-v2
bao auth enable kubernetes
bao write auth/kubernetes/config \
  kubernetes_host=https://kubernetes.default.svc:443
```

Create the least-privilege ESO policy:

```bash
bao policy write external-secrets - <<'EOF'
path "secret/data/homelab/*" {
  capabilities = ["read"]
}

path "secret/metadata/homelab/*" {
  capabilities = ["read", "list"]
}
EOF
```

Bind that policy only to the service account created by
`apps/openbao-config/eso-serviceaccount.yaml`:

```bash
bao write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets-openbao \
  bound_service_account_namespaces=external-secrets \
  policies=external-secrets \
  ttl=1h
```

## Store the initial values

Still inside the OpenBao pod, write the test values:

```bash
bao kv put secret/homelab/test username=example password=change-me
```

Write the Cloudflare token without embedding it in a command or shell history:

```bash
read -s CLOUDFLARE_API_TOKEN
echo
bao kv put secret/homelab/cloudflare api-token="$CLOUDFLARE_API_TOKEN"
unset CLOUDFLARE_API_TOKEN
```

Exit the pod shell when finished:

```bash
exit
```

## Reconcile the store and ExternalSecrets

The `external-secrets-config` Application installs the `ClusterSecretStore`
and the example `ExternalSecret` resources:

```bash
kubectl get application external-secrets-config --namespace argo-cd
kubectl get clustersecretstore openbao
kubectl get externalsecrets --all-namespaces
```

Verify status without printing secret data:

```bash
kubectl describe clustersecretstore openbao
kubectl get secret openbao-test --namespace default
kubectl get secret cloudflare-api-token --namespace cert-manager
```

Do not use `kubectl get secret --output yaml` during routine verification; its
base64-encoded values are still secrets and may be captured in terminal logs.

## Add another managed secret

1. Write the value beneath an approved OpenBao path.
2. Add an `ExternalSecret` manifest under an application config directory.
3. Reference individual properties rather than syncing a broad secret tree.
4. Commit only the `ExternalSecret`, never the value.
5. Verify the `Ready` condition and the target Secret's existence.

If ESO reports `permission denied`, compare the KV path, policy path, auth role,
service account name, and namespace. If it reports connection errors, confirm
that `openbao.openbao.svc.cluster.local:8200` resolves from the ESO namespace.
