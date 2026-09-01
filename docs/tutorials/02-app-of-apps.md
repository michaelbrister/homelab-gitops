# Tutorial 2: Understand the App-of-Apps Layout

This repository uses Argo CD's app-of-apps pattern. One root Application owns
a directory of child Applications, and each child installs one component.

## Follow the reconciliation path

The path from Git to a running workload is:

```text
argocd/root-app.yaml
  -> argocd/applications/*.yaml
     -> Helm chart plus apps/<name>/values.yaml
        or apps/<name>/ Kubernetes resources
```

For example, `argocd/applications/gitea.yaml` has two sources:

1. the upstream Gitea Helm chart; and
2. this repository, exposed to Argo CD as the `$values` source.

The chart reads `$values/apps/gitea/values.yaml`. This keeps the upstream chart
separate from the configuration owned by the homelab.

Configuration-only Applications work differently. For example,
`argocd/applications/cert-manager-config.yaml` points directly to
`apps/cert-manager-config`, where Kustomize discovers `kustomization.yaml`.

## Inspect the live tree

```bash
kubectl get application homelab --namespace argo-cd
kubectl get applications --namespace argo-cd \
  --output custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status
```

Inspect one child in detail:

```bash
kubectl describe application gitea --namespace argo-cd
```

To see which live resources it tracks:

```bash
kubectl get application gitea --namespace argo-cd \
  --output jsonpath='{range .status.resources[*]}{.kind}{"\t"}{.namespace}{"\t"}{.name}{"\n"}{end}'
```

## Understand automated sync

The Applications enable:

- `prune`, which removes resources deleted from Git; and
- `selfHeal`, which reverts drift made directly in the cluster.

That means Git is the control surface. Edit a file, review the diff, commit it,
and push it to `main`. A direct `kubectl edit` may be overwritten on the next
reconciliation.

Use direct cluster changes only for diagnosis or for bootstrap secrets that
are deliberately excluded from Git.

## Safely change a value

As a low-risk exercise, render or inspect a values file locally, then change it
through Git:

```bash
git diff -- apps/gitea/values.yaml
git status --short
```

After the commit reaches `main`, watch the corresponding Application and
workload:

```bash
kubectl get application gitea --namespace argo-cd --watch
kubectl get pods --namespace gitea --watch
```

Press Ctrl-C after reconciliation completes.

## Diagnose sync failures

Start with the Application conditions and Kubernetes events:

```bash
kubectl get application gitea --namespace argo-cd --output yaml
kubectl get events --namespace gitea --sort-by=.lastTimestamp
```

Common causes include an unavailable chart repository, an invalid values key,
a missing storage class, a missing CRD, or a resource that depends on a secret
not created yet.
