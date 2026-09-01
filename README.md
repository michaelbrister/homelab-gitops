# Homelab GitOps

This repository manages a single-node MicroK8s homelab with Argo CD. Argo CD
bootstraps itself from this repository and continuously reconciles the
applications under `argocd/applications/`.

## What is installed

- Argo CD for GitOps reconciliation
- Gitea for self-hosted Git
- cert-manager with Let's Encrypt DNS-01 issuers
- kube-prometheus-stack, Loki, and Alloy for metrics and logs
- OpenBao and External Secrets Operator for secret delivery

## Tutorials

See the [complete homelab tutorial index](docs/Tutorial-index.md) for the
validated guides and planned implementation candidates.

The tutorials build on one another. Start with the first chapter on a new
cluster, or jump to the component you are changing:

1. [Bootstrap Argo CD](docs/tutorials/01-bootstrap-argocd.md)
2. [Understand the app-of-apps layout](docs/tutorials/02-app-of-apps.md)
3. [Add an application](docs/tutorials/03-add-an-application.md)
4. [Issue TLS certificates with cert-manager](docs/tutorials/04-tls-with-cert-manager.md)
5. [Install and verify the observability stack](docs/tutorials/05-observability.md)
6. [Deliver secrets with OpenBao and External Secrets](docs/tutorials/06-openbao-and-external-secrets.md)

Planned components that do not yet have Argo CD Applications or values in this
repository are documented separately:

7. [Plan the Linkerd deployment](docs/tutorials/07-linkerd-planned.md)
8. [Plan the Permission Manager deployment](docs/tutorials/08-permission-manager-planned.md)
9. [Plan the code-server deployment](docs/tutorials/09-code-server-planned.md)

The planned tutorials are implementation guides, not claims that those
components are currently installed. Each one lists the decisions and GitOps
artifacts required to move it into the managed stack.

## Repository layout

```text
bootstrap/argocd/       One-time Argo CD Helm release
argocd/root-app.yaml    Root Application applied during bootstrap
argocd/applications/    Child Argo CD Applications
apps/                   Helm values and Kubernetes resources
docs/tutorials/         Guided setup and extension tutorials
```

## Important assumptions

The checked-in configuration is tailored to this homelab. Before reusing it,
review the hostnames, email address, storage class, ingress class, repository
URL, and secret paths. In particular, the manifests assume:

- a working MicroK8s cluster;
- the `traefik` ingress class;
- the `microk8s-hostpath` storage class;
- DNS for `*.home.mikebrister.com` points at the ingress controller; and
- the repository is reachable at
  `https://github.com/michaelbrister/homelab-gitops.git`.

Secrets and OpenBao bootstrap material must never be committed to this
repository.
