# Tutorial 7: Plan the Linkerd Deployment

> **Status: not implemented.** This repository does not currently contain
> Linkerd values or Argo CD Applications. The commands in this tutorial are
> validation and implementation guidance for a future change.

Linkerd adds a service-mesh data plane, workload identity, and service-level
telemetry. It should be introduced only after the base cluster and existing
observability stack are healthy.

## Decide the deployment shape

Before adding manifests, decide and record:

- the Linkerd CLI and control-plane version to pin;
- how the identity trust anchor and issuer certificate will be generated,
  stored, and rotated;
- which namespaces will be meshed first;
- whether the Viz extension is required; and
- whether the Viz dashboard will remain port-forward-only or receive a
  protected ingress.

Do not store identity private keys in this repository. If cert-manager or
OpenBao will manage them, test the complete issuance and rotation path before
meshing an application.

## Validate the cluster first

Install a Linkerd CLI version that matches the version you intend to deploy,
then run:

```bash
linkerd version --client
linkerd check --pre
```

Resolve every failed preflight check before creating an Argo CD Application.
The preflight check is diagnostic and does not install Linkerd.

## Add the GitOps artifacts

Prefer Linkerd's Helm charts over piping CLI-generated manifests directly into
the cluster. The intended repository shape is:

```text
apps/linkerd-crds/values.yaml
apps/linkerd-control-plane/values.yaml
apps/linkerd-viz/values.yaml               # only if Viz is selected
argocd/applications/linkerd-crds.yaml
argocd/applications/linkerd-control-plane.yaml
argocd/applications/linkerd-viz.yaml       # only if Viz is selected
```

Keep CRDs, the control plane, and Viz in separate Applications so dependencies
and failures are visible. Pin tested chart versions rather than tracking a
moving channel.

Before writing values, inspect the charts for the selected release:

```bash
helm repo add linkerd https://helm.linkerd.io/stable
helm repo update
helm search repo linkerd --versions
helm show values linkerd/linkerd-control-plane --version <version>
```

Render every chart locally before committing it:

```bash
helm template linkerd-crds linkerd/linkerd-crds \
  --namespace linkerd \
  --version <version> \
  --values apps/linkerd-crds/values.yaml

helm template linkerd-control-plane linkerd/linkerd-control-plane \
  --namespace linkerd \
  --version <version> \
  --values apps/linkerd-control-plane/values.yaml
```

The Argo CD Applications should follow
[`03-add-an-application.md`](03-add-an-application.md), use the selected chart
version, and create the `linkerd` namespace. Do not add automated namespace
injection during the initial control-plane deployment.

## Verify the control plane

After the GitOps Applications have synced:

```bash
kubectl get applications --namespace argo-cd | grep linkerd
kubectl get pods --namespace linkerd
linkerd check
```

If Viz was selected, validate it separately:

```bash
kubectl get pods --namespace linkerd-viz
linkerd viz check
linkerd viz dashboard
```

Port-forwarding is the default dashboard access path. If a stable ingress is
later required, protect it with TLS and authentication and limit it to trusted
networks.

## Mesh one workload first

Choose a stateless, non-critical namespace as the first canary. Add the
injection annotation through Git, sync it, and restart only that workload.
Then verify:

```bash
linkerd check --proxy
linkerd stat deployments --namespace <canary-namespace>
kubectl get pods --namespace <canary-namespace>
```

Do not mesh OpenBao, the ingress controller, or the entire cluster as the first
test. Expand namespace by namespace after observing normal traffic and restart
behavior.

## Definition of done

Linkerd is implemented only when the pinned charts and values are in Git, the
identity material has a tested rotation path, `linkerd check` passes, a canary
workload is healthy, and removal steps have been tested or documented.
