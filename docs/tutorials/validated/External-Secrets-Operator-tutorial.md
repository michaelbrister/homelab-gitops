# External Secrets Operator on MicroK8s with Argo CD

This tutorial installs External Secrets Operator (ESO) through the Argo CD App-of-Apps structure. It stops after validating the operator and APIs; `Openbao-tutorial.md` configures the provider and migrates the Cloudflare token. Only the completed working path is included.

## 1. Completed outcome

At the end:

- ESO chart `2.8.0` runs in `external-secrets`.
- Argo CD owns the release and reconciles drift.
- ESO's CRDs are installed.
- The operator, webhook, and certificate controller are healthy.
- Prometheus can discover ESO metrics through a ServiceMonitor.
- The cluster is ready for the OpenBao ClusterSecretStore in the next tutorial.

```text
Git --> Argo CD root Application --> external-secrets Application
                                      |
                                      +--> CRDs, controllers, ServiceMonitor
```

## 2. Version and prerequisites

| Item | Value |
|---|---|
| ESO chart | `2.8.0` |
| Chart repository | `https://charts.external-secrets.io` |
| Namespace | `external-secrets` |
| Replicas | `1` |
| CRDs | installed by chart |
| ServiceMonitor | enabled |

Complete the MicroK8s, Argo CD, and kube-prometheus-stack tutorials first. OpenBao is not required yet.

```bash
microk8s kubectl get application -n argo-cd homelab
microk8s kubectl get crd servicemonitors.monitoring.coreos.com
```

Checkpoint: `homelab` is healthy and the ServiceMonitor CRD exists.

## 3. Repository layout

```text
apps/external-secrets/
└── values.yaml

argocd/applications/
└── external-secrets.yaml
```

The official repository supplies the chart; your Git repository supplies its values.

## 4. Create the values

Create `apps/external-secrets/values.yaml`:

```yaml
installCRDs: true

replicaCount: 1

serviceMonitor:
  enabled: true
```

`installCRDs` supplies the ExternalSecret, SecretStore, and ClusterSecretStore APIs. One replica matches the single-node homelab. The ServiceMonitor integrates ESO with Prometheus.

## 5. Create the Argo CD Application

Create `argocd/applications/external-secrets.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argo-cd
spec:
  project: default
  sources:
    - repoURL: https://charts.external-secrets.io
      chart: external-secrets
      targetRevision: 2.8.0
      helm:
        releaseName: external-secrets
        valueFiles:
          - $values/apps/external-secrets/values.yaml
    - repoURL: https://github.com/michaelbrister/homelab-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

The second source exposes the homelab repository as `$values`. `CreateNamespace=true` creates the namespace, and server-side apply supports the CRDs.

## 6. Render before deployment

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm template external-secrets external-secrets/external-secrets \
  --version 2.8.0 \
  --namespace external-secrets \
  --values apps/external-secrets/values.yaml \
  > /tmp/external-secrets-rendered.yaml
grep '^kind: CustomResourceDefinition' /tmp/external-secrets-rendered.yaml | head
grep '^kind: Deployment' /tmp/external-secrets-rendered.yaml
grep '^kind: ServiceMonitor' /tmp/external-secrets-rendered.yaml
```

Checkpoint: rendering succeeds and includes CRDs, the controller Deployments, and a ServiceMonitor.

## 7. Commit and install through Argo CD

```bash
git add apps/external-secrets/values.yaml argocd/applications/external-secrets.yaml
git commit -m "Install External Secrets Operator"
git push
```

The root Application discovers the child Application and installs the chart.

```bash
microk8s kubectl get application -n argo-cd external-secrets -w
```

Stop watching once it reports `Synced` and `Healthy`.

## 8. Validate all controllers

```bash
microk8s kubectl rollout status deployment/external-secrets \
  -n external-secrets
microk8s kubectl rollout status deployment/external-secrets-webhook \
  -n external-secrets
microk8s kubectl rollout status deployment/external-secrets-cert-controller \
  -n external-secrets
microk8s kubectl get pods -n external-secrets -o wide
```

The completed installation has one ready pod for each component:

- `external-secrets` reconciles stores and ExternalSecrets;
- `external-secrets-webhook` validates and converts custom resources;
- `external-secrets-cert-controller` manages the webhook's internal TLS.

## 9. Validate the APIs

```bash
microk8s kubectl get crd externalsecrets.external-secrets.io
microk8s kubectl get crd secretstores.external-secrets.io
microk8s kubectl get crd clustersecretstores.external-secrets.io
microk8s kubectl api-resources --api-group=external-secrets.io
```

Checkpoint: ExternalSecret, SecretStore, and ClusterSecretStore appear in API discovery.

## 10. Validate monitoring

```bash
microk8s kubectl get servicemonitor -n external-secrets
microk8s kubectl get service -n external-secrets
```

When the existing Prometheus configuration accepts ServiceMonitors across namespaces, inspect target discovery through a local tunnel:

```bash
microk8s kubectl port-forward -n monitoring \
  service/kube-prometheus-stack-prometheus 9090:9090
```

Open `http://127.0.0.1:9090/targets` and confirm the ESO target is healthy.

## 11. Validate restart behavior

Delete the main operator pod and allow its Deployment to replace it:

```bash
microk8s kubectl delete pod -n external-secrets \
  -l app.kubernetes.io/name=external-secrets
microk8s kubectl rollout status deployment/external-secrets \
  -n external-secrets
microk8s kubectl get pods -n external-secrets
```

Checkpoint: the replacement becomes ready without manual recovery.

## 12. Validate Argo CD self-healing

Temporarily change the live replica count:

```bash
microk8s kubectl scale deployment/external-secrets \
  -n external-secrets --replicas=2
```

Argo CD restores the committed value. Verify that it returns to `1`:

```bash
microk8s kubectl get deployment external-secrets \
  -n external-secrets -o jsonpath='{.spec.replicas}{"\n"}'
```

Checkpoint: the Application is again `Synced` and `Healthy`.

## 13. Continue with OpenBao

Do not create a placeholder store here. A functional store requires the provider, authentication method, policy, role, ServiceAccount, and secret path to agree.

`Openbao-tutorial.md` completes the chain in this order:

1. install and initialize OpenBao;
2. configure static auto-unseal;
3. enable Kubernetes authentication;
4. create the ESO policy and role;
5. create the ESO ServiceAccount;
6. create the ClusterSecretStore;
7. create an ExternalSecret for the Cloudflare token;
8. verify the generated Kubernetes Secret;
9. migrate OpenBao storage to integrated Raft.

For ESO `2.8.0`, the completed integration uses ESO's Vault-compatible provider against OpenBao because that tested path supports Kubernetes authentication.

## 14. Security and ownership

- ESO reads authoritative secret values from a provider; those values do not belong in Git.
- Give its provider identity only the paths and operations it needs.
- A ClusterSecretStore is cluster-scoped, so restrict its authentication role deliberately.
- Generated Secrets still live in Kubernetes and remain subject to Kubernetes RBAC.
- Argo CD owns the operator and declarative ESO resources. ESO owns the generated target Secrets.
- Never commit provider tokens, OpenBao root tokens, seal material, or Secret values.

## 15. Cleanup and rollback

Before provider resources exist, remove ESO by deleting its child Application manifest from Git, committing, and letting the root Application prune it. Remove CRDs only after confirming none of their resources are needed.

After stores and ExternalSecrets exist, remove them before the operator. Generated Kubernetes Secrets may remain depending on creation and deletion policies, so review them explicitly.

To roll back a chart change, restore the last validated `targetRevision` and matching values in Git, then let Argo CD reconcile.

## 16. Final checkpoint

The installation is complete when the Application is healthy; all three controllers are ready; the core CRDs exist; the ServiceMonitor exists; a deleted pod is recreated; self-heal restores one replica; and no provider credential is stored in Git.

Continue with `Openbao-tutorial.md` to connect ESO to OpenBao and migrate the Cloudflare token.
