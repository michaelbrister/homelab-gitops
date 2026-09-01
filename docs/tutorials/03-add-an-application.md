# Tutorial 3: Add an Application

This tutorial adds a Helm-based component to the existing app-of-apps layout.
Use the same shape as the current Applications so chart configuration remains
in Git while chart source code stays upstream.

The example uses placeholders. Replace every value in angle brackets before
committing it.

## Create the values file

Create `apps/<app-name>/values.yaml` with only the settings you intend to
override:

```yaml
replicaCount: 1
```

Consult the selected chart's documentation and pin a tested chart version.
Avoid copying the chart's entire default values file; small override files are
easier to review during upgrades.

## Create the child Application

Create `argocd/applications/<app-name>.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argo-cd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: <helm-repository-url>
      chart: <chart-name>
      targetRevision: <pinned-chart-version>
      helm:
        valueFiles:
          - $values/apps/<app-name>/values.yaml
    - repoURL: https://github.com/michaelbrister/homelab-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: <app-name>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Use a released chart version for `targetRevision`, not `latest`. Confirm that
the chart's repository URL and chart name match its upstream documentation.

## Validate before pushing

At minimum, parse the YAML and inspect the diff:

```bash
kubectl apply --dry-run=client --filename argocd/applications/<app-name>.yaml
git diff --check
git diff -- apps/<app-name> argocd/applications/<app-name>.yaml
```

If the Helm repository is configured locally, render the chart as an additional
check:

```bash
helm template <app-name> <repository-alias>/<chart-name> \
  --namespace <app-name> \
  --version <pinned-chart-version> \
  --values apps/<app-name>/values.yaml
```

Rendering catches chart/value mismatches before Argo CD attempts the sync.

## Reconcile and verify

Commit and push both files. The root Application automatically discovers the
new child:

```bash
kubectl get application <app-name> --namespace argo-cd --watch
```

Then inspect the workload:

```bash
kubectl get all --namespace <app-name>
kubectl get events --namespace <app-name> --sort-by=.lastTimestamp
```

## Add raw Kubernetes resources instead

For configuration that does not come from a Helm chart, create an
`apps/<app-name>/kustomization.yaml` and point a single-source Application at
that directory. Use `apps/cert-manager-config` and
`argocd/applications/cert-manager-config.yaml` as the working examples.

Keep controllers and their custom resources in separate Applications when
ordering matters. A controller must install its CRDs before Argo CD can apply
resources of those kinds.
