# Argo CD GitOps Bootstrap with Helmfile and App-of-Apps

This tutorial bootstraps Argo CD on the MicroK8s foundation and then transfers workload deployment to a root App-of-Apps Application.

The completed design uses:

```text
Argo CD Helm chart: 10.3.0
Argo CD app version: v3.5.0
Bootstrap tool: Helmfile
Namespace: argo-cd
Ingress controller: Traefik
Repository: https://github.com/michaelbrister/homelab-gitops.git
Root Application: homelab
```

Argo CD cannot initially install itself from an Argo CD Application. Helmfile owns the bootstrap release; after that, Argo CD owns the root Application and every child workload.

## 1. Prerequisites

Complete `MicroK8s-foundation-tutorial.md` first. Verify:

```bash
kubectl get nodes
kubectl get storageclass
kubectl get ingressclass traefik
dig +short argocd.home.mikebrister.com
```

Install Helm and Helmfile on the management workstation, and create the public Git repository:

```text
https://github.com/michaelbrister/homelab-gitops.git
```

## 2. Create the repository structure

```bash
mkdir -p homelab-gitops/bootstrap/argocd
mkdir -p homelab-gitops/argocd/applications
mkdir -p homelab-gitops/apps
cd homelab-gitops
git init
```

The initial structure is:

```text
homelab-gitops/
├── bootstrap/
│   └── argocd/
│       ├── helmfile.yaml
│       └── values.yaml
├── argocd/
│   ├── root-app.yaml
│   └── applications/
│       └── .gitkeep
└── apps/
```

The `.gitkeep` matters because Git does not track empty directories. The root Application needs `argocd/applications` to exist in the repository before it can reconcile that path.

## 3. Define the Helmfile bootstrap

Create `bootstrap/argocd/helmfile.yaml`:

```yaml
repositories:
  - name: argo
    url: https://argoproj.github.io/argo-helm

releases:
  - name: argo-cd
    namespace: argo-cd
    createNamespace: true
    chart: argo/argo-cd
    version: 10.3.0
    values:
      - values.yaml
```

Create `bootstrap/argocd/values.yaml` for the initial HTTP bootstrap:

```yaml
configs:
  cm:
    url: http://argocd.home.mikebrister.com

  params:
    server.insecure: "true"

server:
  ingress:
    enabled: true
    ingressClassName: traefik
    hostname: argocd.home.mikebrister.com
    pathType: Prefix
    paths:
      - /
    tls: false
```

`server.insecure=true` means the Argo CD server speaks HTTP behind Traefik. Later, cert-manager will add HTTPS at the ingress without requiring Argo CD itself to terminate TLS.

## 4. Render and review Argo CD

```bash
cd bootstrap/argocd
helmfile template > /tmp/argocd.yaml
```

Inspect the rendered ingress:

```bash
grep -A40 'kind: Ingress' /tmp/argocd.yaml
```

Verify:

```text
ingressClassName: traefik
host: argocd.home.mikebrister.com
backend service port: 80
```

Review the actual changes Helmfile will make:

```bash
helmfile diff
```

## 5. Install Argo CD

```bash
helmfile apply
```

Wait for its workloads:

```bash
kubectl get pods -n argo-cd -w
```

Verify deployments and services:

```bash
kubectl get deployments,statefulsets,services,ingress -n argo-cd
```

Test the initial ingress:

```bash
curl -I http://argocd.home.mikebrister.com
```

## 6. Retrieve the initial administrator password

Argo CD stores the bootstrap administrator password in a Secret:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argo-cd \
  -o jsonpath='{.data.password}' \
  | base64 --decode
echo
```

Use username `admin` to sign in. Do not commit this password.

## 7. Create the root Application

Create `argocd/root-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: homelab
  namespace: argo-cd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/michaelbrister/homelab-gitops.git
    targetRevision: main
    path: argocd/applications
    directory:
      recurse: true

  destination:
    server: https://kubernetes.default.svc
    namespace: argo-cd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The root Application watches only child `Application` manifests. Each child then renders and manages its own workload.

## 8. Commit the bootstrap repository

Create the tracked placeholder:

```bash
touch argocd/applications/.gitkeep
```

Commit and push:

```bash
git add bootstrap argocd apps
git commit -m "bootstrap Argo CD GitOps repository"
git branch -M main
git remote add origin https://github.com/michaelbrister/homelab-gitops.git
git push -u origin main
```

## 9. Apply the one-time root Application

The root Application must be created once outside itself:

```bash
kubectl apply -f argocd/root-app.yaml
```

Force an initial refresh:

```bash
kubectl annotate application homelab \
  -n argo-cd \
  argocd.argoproj.io/refresh=hard \
  --overwrite
```

Verify:

```bash
kubectl get applications -n argo-cd
```

Expected:

```text
NAME      SYNC STATUS   HEALTH STATUS
homelab   Synced        Healthy
```

## 10. Understand the App-of-Apps flow

From this point onward, do not manually apply child Application files. Commit them beneath `argocd/applications`:

```text
Git push
   |
   v
homelab root Application
   |
   v
argocd/applications/*.yaml
   |
   v
child Argo CD Applications
   |
   v
Helm charts and Kubernetes workloads
```

Each child Application can use two sources:

1. an upstream Helm chart repository;
2. `homelab-gitops` as the source of its values file.

That avoids copying third-party charts into the GitOps repository.

## 11. Validate child-Application discovery

Create `argocd/applications/namespace-test.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: namespace-test
  namespace: argo-cd
spec:
  project: default
  source:
    repoURL: https://github.com/michaelbrister/homelab-gitops.git
    targetRevision: main
    path: apps/namespace-test
  destination:
    server: https://kubernetes.default.svc
    namespace: namespace-test
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Create `apps/namespace-test/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - configmap.yaml
```

Create `apps/namespace-test/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitops-test
data:
  owner: argo-cd
```

Commit and push:

```bash
git add argocd/applications/namespace-test.yaml apps/namespace-test
git commit -m "validate App-of-Apps discovery"
git push
```

Watch Argo CD discover and deploy it:

```bash
kubectl get applications -n argo-cd -w
kubectl get configmap gitops-test -n namespace-test -o yaml
```

## 12. Validate self-healing

Change the live ConfigMap:

```bash
kubectl patch configmap gitops-test \
  -n namespace-test \
  --type merge \
  -p '{"data":{"owner":"manual"}}'
```

Argo CD should restore `owner: argo-cd`:

```bash
kubectl get configmap gitops-test \
  -n namespace-test \
  -o jsonpath='{.data.owner}'
echo
```

This validates automated sync and self-heal using a field explicitly managed by Git.

## 13. Remove the test Application

Delete its files from Git:

```bash
git rm argocd/applications/namespace-test.yaml
git rm -r apps/namespace-test
git commit -m "remove App-of-Apps test"
git push
```

Because pruning is enabled, Argo CD removes the child Application and its resources.

## 14. Final validation

```bash
helmfile -f bootstrap/argocd/helmfile.yaml list
kubectl get pods -n argo-cd
kubectl get ingress -n argo-cd
kubectl get application homelab -n argo-cd
curl -I http://argocd.home.mikebrister.com
```

The GitOps bootstrap is complete when:

- Argo CD pods are healthy;
- its UI is reachable;
- the root `homelab` Application is `Synced` and `Healthy`;
- child Applications appear after a Git push;
- Argo CD restores drift in Git-managed fields.

## 15. Ownership and recovery

```text
Helmfile owns:
  Argo CD installation

kubectl bootstrap owns:
  initial creation of argocd/root-app.yaml

Argo CD owns:
  root Application reconciliation
  child Applications
  application workloads
```

If Argo CD must be rebuilt, run `helmfile apply` again, then reapply `argocd/root-app.yaml`. The root Application reconstructs the child Applications from Git.

The later cert-manager tutorial updates `bootstrap/argocd/values.yaml` to serve trusted HTTPS at `https://argocd.home.mikebrister.com`.
