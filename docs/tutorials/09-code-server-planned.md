# Tutorial 9: Plan the code-server Deployment

> **Status: not implemented.** This repository has not selected a code-server
> chart or image and contains no code-server Argo CD Application.

code-server provides a browser-based development environment. A useful GitOps
deployment needs deliberate choices for packaging, persistence, identity,
secrets, and workload privileges before any manifest is added.

## Choose and pin the packaging

Select one maintained path:

- a reviewed Helm chart with a pinned version;
- a Deployment, Service, and PersistentVolumeClaim maintained in this
  repository; or
- an internally maintained image and chart.

Record the upstream source, selected version, image digest, exposed service
port, supported configuration mechanism, and upgrade notes. Do not build the
GitOps Application around an unpinned example chart.

If a Helm chart is selected, inspect and render it before committing values:

```bash
helm show values <repository>/<chart> --version <version>
helm template code-server <repository>/<chart> \
  --namespace code-server \
  --version <version> \
  --values apps/code-server/values.yaml
```

## Design persistence and access

Decide which paths must survive a pod replacement. At minimum, evaluate:

- the workspace directory;
- editor configuration and extensions;
- Git configuration; and
- SSH known-host data.

Use a PVC for durable data and document its backup and restore process. The
current single-node cluster can use `microk8s-hostpath`, but that storage ties
the volume to a node and is not a backup.

Keep first access private with port-forwarding. Before enabling ingress,
choose a real authentication strategy, issue TLS, and decide whether an
additional identity-aware proxy is required.

## Design the security boundary

A browser IDE can execute commands and read every mounted credential. Keep its
service account minimally privileged and do not mount a broad cluster-admin
kubeconfig, host filesystem path, or container runtime socket.

Deliver individual Git or registry credentials through External Secrets. Mount
only the credentials required by this instance, and prefer short-lived tokens
where the upstream service supports them.

Add resource requests and limits, a restrictive pod security context, and a
NetworkPolicy appropriate for the Git hosts, package registries, and internal
services the workspace must reach.

## Add the GitOps artifacts

The expected layout is:

```text
apps/code-server/values.yaml
apps/code-server-config/kustomization.yaml
apps/code-server-config/external-secret.yaml
argocd/applications/code-server.yaml
argocd/applications/code-server-config.yaml
```

If raw manifests are chosen instead of Helm, place the Deployment, Service,
PVC, and NetworkPolicy under `apps/code-server/` with a Kustomization and use a
single-source Argo CD Application.

The values or manifests should define:

- a pinned image;
- a non-root security context when supported;
- persistent storage and a known mount path;
- CPU and memory requests and limits;
- a ClusterIP Service;
- ingress disabled for the first deployment; and
- Secret references without secret values.

## Verify before exposing it

After committing and syncing the GitOps artifacts:

```bash
kubectl get application code-server --namespace argo-cd
kubectl get pods,services,persistentvolumeclaims --namespace code-server
kubectl get events --namespace code-server --sort-by=.lastTimestamp
```

Discover the actual service port, then port-forward it:

```bash
kubectl get service code-server --namespace code-server
kubectl port-forward service/code-server --namespace code-server 8080:<service-port>
```

Create a test file, restart the pod through the owning controller, and confirm
the file and editor configuration persist. Also confirm the pod cannot perform
unintended Kubernetes operations:

```bash
kubectl auth can-i --list \
  --as=system:serviceaccount:code-server:<service-account> \
  --namespace code-server
```

Only after those checks should ingress be enabled with the Traefik class,
cert-manager TLS, authentication, and a hostname approved for the homelab.

## Definition of done

code-server is implemented only when the package and image are pinned, data
survives a pod replacement, backups are documented, secrets are narrowly
scoped, the pod is not privileged, and authenticated TLS access has been
verified.
