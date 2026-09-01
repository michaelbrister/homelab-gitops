# Tutorial 1: Bootstrap Argo CD

This tutorial installs Argo CD with Helmfile and connects it to the root
Application in this repository.

## Prerequisites

You need:

- a running Kubernetes cluster and a working `kubectl` context;
- Helm and Helmfile;
- cluster-admin access for the initial installation;
- DNS and Traefik if you want to use the configured ingress immediately; and
- a clone of this repository.

The persistent workloads in later tutorials expect the
`microk8s-hostpath` storage class. Check the cluster before continuing:

```bash
kubectl cluster-info
kubectl get nodes
kubectl get storageclass
kubectl get ingressclass
```

If your ingress class, storage class, domain, or repository URL differs, update
the manifests before applying the root Application.

## Install Argo CD

From the repository root, render the release first:

```bash
helmfile -f bootstrap/argocd/helmfile.yaml template
```

Review the output, then install it:

```bash
helmfile -f bootstrap/argocd/helmfile.yaml apply
kubectl wait --for=condition=Available deployment --all \
  --namespace argo-cd --timeout=5m
kubectl get pods --namespace argo-cd
```

The release uses the chart version pinned in
`bootstrap/argocd/helmfile.yaml` and the ingress settings in
`bootstrap/argocd/values.yaml`.

## Access Argo CD before DNS is ready

Use a port-forward for the initial login:

```bash
kubectl port-forward --namespace argo-cd service/argo-cd-argocd-server 8080:443
```

In another terminal, retrieve the generated password:

```bash
kubectl get secret --namespace argo-cd argocd-initial-admin-secret \
  --output jsonpath='{.data.password}' | base64 --decode
echo
```

Open `https://127.0.0.1:8080` and sign in as `admin`. The browser may warn
about the locally presented certificate during this bootstrap-only step.

If the service name differs after a chart upgrade, find it with:

```bash
kubectl get services --namespace argo-cd
```

## Create the root Application

Apply the only resource that is intentionally bootstrapped outside Argo CD:

```bash
kubectl apply --filename argocd/root-app.yaml
kubectl get applications --namespace argo-cd
kubectl describe application homelab --namespace argo-cd
```

The root Application watches `argocd/applications/`. Argo CD will discover
the child Applications there and begin reconciling them automatically.

Some child Applications require secrets or cluster preparation described in
later tutorials. A temporary `Degraded` or `Progressing` state is therefore
expected on a first installation; inspect the affected Application rather than
repeatedly reapplying the root manifest.

## Verify the bootstrap

```bash
kubectl get applications --namespace argo-cd
kubectl get pods --all-namespaces
```

The bootstrap is complete when the `homelab` Application is present and its
child Applications have been created. Continue with
[Tutorial 2](02-app-of-apps.md) to understand how reconciliation works.

## Recovery

If a values change breaks Argo CD itself, fix the values locally and run the
Helmfile apply command again. Do not delete the `argo-cd` namespace as a first
response: doing so removes Application state and the initial admin secret.
