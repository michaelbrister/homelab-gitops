# Gitea on MicroK8s with Argo CD and PostgreSQL

This tutorial deploys Gitea as the first real workload managed by the Argo CD App-of-Apps structure. It uses one Gitea instance, one PostgreSQL instance, persistent MicroK8s hostpath volumes, and Traefik ingress.

The completed application design is:

```text
Argo CD multi-source Application
        |
        |-- official Gitea Helm chart
        `-- apps/gitea/values.yaml
                    |
                    v
              Gitea + PostgreSQL
                    |
                    v
           microk8s-hostpath PVCs
```

This tutorial establishes Gitea over LAN HTTP. `Cert-manager-Cloudflare-tutorial.md` later changes the hostname to trusted HTTPS without changing Gitea's internal HTTP protocol.

## 1. Prerequisites

Complete:

- `MicroK8s-foundation-tutorial.md`
- `ArgoCD-gitops-bootstrap-tutorial.md`

Verify:

```bash
kubectl get application homelab -n argo-cd
kubectl get storageclass microk8s-hostpath
kubectl get ingressclass traefik
dig +short gitea.home.mikebrister.com
```

The root Application must be `Synced` and `Healthy`.

## 2. Select and pin the Gitea chart

Add the official repository:

```bash
helm repo add gitea https://dl.gitea.com/charts/
helm repo update
helm search repo gitea/gitea --versions | head -10
```

Choose the chart version used for the deployment and save it for the following commands:

```bash
GITEA_CHART_VERSION="REPLACE_WITH_SELECTED_VERSION"
```

Use that exact version both when rendering locally and in the Argo CD Application. Do not track `latest`.

## 3. Create the repository files

From the `homelab-gitops` repository root:

```bash
mkdir -p apps/gitea
touch apps/gitea/values.yaml
touch argocd/applications/gitea.yaml
```

The relevant layout is:

```text
argocd/applications/gitea.yaml
apps/gitea/values.yaml
```

## 4. Configure Gitea and PostgreSQL

Create `apps/gitea/values.yaml`:

```yaml
ingress:
  enabled: true
  ingressClassName: traefik
  hosts:
    - host: gitea.home.mikebrister.com
      paths:
        - path: /
          pathType: Prefix

service:
  http:
    type: ClusterIP
    port: 3000

persistence:
  enabled: true
  storageClass: microk8s-hostpath
  size: 10Gi

gitea:
  config:
    repository:
      ENABLE_PUSH_CREATE_USER: true

    server:
      DOMAIN: gitea.home.mikebrister.com
      ROOT_URL: http://gitea.home.mikebrister.com/
      PROTOCOL: http

    database:
      DB_TYPE: postgres

postgresql:
  enabled: true

  primary:
    persistence:
      enabled: true
      storageClass: microk8s-hostpath
      size: 10Gi

postgresql-ha:
  enabled: false

redis:
  enabled: false

redis-cluster:
  enabled: false
```

The important choices are:

- `ingressClassName: traefik` uses the field expected by the working chart configuration.
- Gitea listens over HTTP because Traefik is the reverse proxy.
- Gitea data and PostgreSQL data use separate 10Gi PVCs.
- PostgreSQL HA and Redis are unnecessary for this single-node lab.
- push-to-create lets a user create a repository by pushing to a new repository URL.

Do not put an administrator password in this values file.

## 5. Create the Argo CD Application

Create `argocd/applications/gitea.yaml`, replacing the chart version with the value selected earlier:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitea
  namespace: argo-cd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  sources:
    - repoURL: https://dl.gitea.com/charts/
      chart: gitea
      targetRevision: REPLACE_WITH_SELECTED_VERSION
      helm:
        valueFiles:
          - $values/apps/gitea/values.yaml

    - repoURL: https://github.com/michaelbrister/homelab-gitops.git
      targetRevision: main
      ref: values

  destination:
    server: https://kubernetes.default.svc
    namespace: gitea

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

The chart comes from the official Gitea repository; only the homelab-specific values live in Git.

## 6. Render locally

```bash
helm template gitea gitea/gitea \
  --namespace gitea \
  --version "$GITEA_CHART_VERSION" \
  -f apps/gitea/values.yaml \
  > /tmp/gitea.yaml
```

Inspect the rendered resource types:

```bash
grep '^kind:' /tmp/gitea.yaml
```

Inspect the Gitea ingress:

```bash
grep -A45 'kind: Ingress' /tmp/gitea.yaml
```

Confirm it contains:

```text
ingressClassName: traefik
host: gitea.home.mikebrister.com
```

Also verify that Gitea and PostgreSQL both render persistent volume claims.

## 7. Commit and deploy through the root Application

```bash
git add apps/gitea/values.yaml argocd/applications/gitea.yaml
git commit -m "deploy Gitea"
git push
```

Do not manually apply `gitea.yaml`. The `homelab` root Application discovers it.

Watch:

```bash
kubectl get applications -n argo-cd -w
```

Expected:

```text
gitea   Synced   Healthy
```

## 8. Validate workloads and persistence

```bash
kubectl get pods -n gitea -w
kubectl get pvc -n gitea
kubectl get services,ingress -n gitea
```

Confirm:

- the Gitea pod is running;
- the PostgreSQL pod is running;
- the Gitea PVC is `Bound`;
- the PostgreSQL PVC is `Bound`;
- the Ingress uses `gitea.home.mikebrister.com`.

Because the StorageClass uses `WaitForFirstConsumer`, a PVC can remain `Pending` until its pod is scheduled.

## 9. Open Gitea

Verify DNS and ingress:

```bash
dig +short gitea.home.mikebrister.com
curl -I http://gitea.home.mikebrister.com
```

Open:

```text
http://gitea.home.mikebrister.com
```

## 10. Create the initial administrator

Find the Gitea pod:

```bash
kubectl get pods -n gitea \
  -l app.kubernetes.io/name=gitea
```

Set the pod name and read the password without echoing it:

```bash
GITEA_POD="REPLACE_WITH_GITEA_POD_NAME"
read -s GITEA_ADMIN_PASSWORD
```

Create the account:

```bash
kubectl exec -n gitea "$GITEA_POD" -c gitea -- \
  gitea admin user create \
  --username mbrister \
  --password "$GITEA_ADMIN_PASSWORD" \
  --email mike@home.mikebrister.com \
  --admin

unset GITEA_ADMIN_PASSWORD
```

This is a one-time bootstrap operation. The credential is not stored in Git.

Sign in through the web interface and verify the account has administrator access.

## 11. Validate repository creation and Git push

Create a temporary local repository:

```bash
mkdir -p /tmp/gitea-test
cd /tmp/gitea-test

git init
git branch -M main
echo "# Gitea test" > README.md
git add README.md
git commit -m "initial commit"
```

Add the Gitea remote:

```bash
git remote add origin \
  http://gitea.home.mikebrister.com/mbrister/test.git
```

Push:

```bash
git push -u origin main
```

Authenticate with the administrator account. With `ENABLE_PUSH_CREATE_USER=true`, pushing to this new path creates the repository.

Open the repository in Gitea and confirm the commit and README are visible.

## 12. Validate restart persistence

Record the repository and account state, then delete only the Gitea pod:

```bash
kubectl delete pod -n gitea \
  -l app.kubernetes.io/name=gitea

kubectl get pods -n gitea -w
```

After the replacement is ready:

- sign in again;
- open the test repository;
- confirm its commit remains;
- verify both PVCs remain `Bound`.

This confirms application and database state survive pod replacement.

## 13. Validate Argo CD ownership

Change a Git-managed setting on the live Ingress:

```bash
kubectl annotate ingress -n gitea \
  -l app.kubernetes.io/name=gitea \
  tutorial-test=manual \
  --overwrite
```

For the strongest self-heal test, patch a field defined in Git, such as the host, then confirm Argo CD restores `gitea.home.mikebrister.com`. Do not leave live-only changes as part of normal operation.

Check:

```bash
kubectl get application gitea -n argo-cd
kubectl get ingress -n gitea -o yaml
```

## 14. HTTPS handoff

Do not enable ingress TLS until cert-manager and its production ClusterIssuer exist. `Cert-manager-Cloudflare-tutorial.md` later changes the relevant final values to:

```yaml
ingress:
  enabled: true
  ingressClassName: traefik
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: gitea.home.mikebrister.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: gitea-tls
      hosts:
        - gitea.home.mikebrister.com

gitea:
  config:
    server:
      DOMAIN: gitea.home.mikebrister.com
      ROOT_URL: https://gitea.home.mikebrister.com/
      PROTOCOL: http
```

The final traffic path remains:

```text
Browser --HTTPS--> Traefik --HTTP--> Gitea:3000
```

## 15. Backup and rollback notes

The important data lives in two PVCs:

- Gitea repositories and application data;
- PostgreSQL database data.

Back up both before chart upgrades. A database-consistent backup should include a PostgreSQL dump rather than only copying live volume files.

To roll back a configuration change:

1. revert the Git commit;
2. push the revert;
3. allow Argo CD to reconcile;
4. verify the Application, pods, PVCs, and repository access.

Do not delete the Gitea Application with its resource finalizer unless deletion of its managed resources is intended.

## 16. Final architecture

```text
GitHub homelab-gitops
        |
        v
Argo CD root Application
        |
        v
Gitea child Application
   |                 |
   v                 v
Gitea Helm chart   Git-managed values
        |
        v
Gitea Service <- Traefik <- LAN DNS
        |
        +-> Gitea 10Gi PVC
        `-> PostgreSQL -> PostgreSQL 10Gi PVC
```
