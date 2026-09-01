# Tutorial 8: Plan the Permission Manager Deployment

> **Status: not implemented.** There is no Permission Manager chart,
> configuration, or Argo CD Application in this repository.

Permission Manager is an administrative interface for Kubernetes users and
RBAC. Because it can grant cluster access, its packaging, credentials, ingress,
and permissions require a security review before deployment.

## Resolve the chart source

The earlier MicroK8s runbook used a local chart outside this repository. Argo
CD cannot reproduce that installation until the chart has an immutable source.
Choose one of these approaches:

1. publish the reviewed chart to a versioned Helm or OCI registry; or
2. vendor the reviewed chart under `charts/permission-manager/` in this
   repository.

A published, immutable chart is easier to reuse and upgrade. Vendoring can be
appropriate when this homelab owns the chart, but it also makes this repository
responsible for maintaining every template.

Do not point Argo CD at an unversioned chart directory in another workstation
checkout.

## Review the chart before adoption

Before adding it to GitOps, confirm that the chart:

- uses APIs supported by the cluster's Kubernetes version;
- does not create PodSecurityPolicy resources;
- uses `autoscaling/v2` if it creates an HPA;
- supports `spec.ingressClassName`;
- scopes Roles and ClusterRoles to the permissions the product actually needs;
- supports a Secret reference instead of placing passwords in Helm values; and
- pins an image by a reviewed version or digest.

Render and inspect the candidate chart:

```bash
helm lint <chart-reference> --values <reviewed-values-file>
helm template permission-manager <chart-reference> \
  --namespace permission-manager \
  --values <reviewed-values-file>
```

Do not proceed if placeholder cluster addresses, passwords, or hostnames remain
in the rendered output.

## Add the GitOps artifacts

After the chart source is settled, add:

```text
apps/permission-manager/values.yaml
apps/permission-manager-config/kustomization.yaml
apps/permission-manager-config/external-secret.yaml
argocd/applications/permission-manager.yaml
argocd/applications/permission-manager-config.yaml
```

The exact config Application is optional, but credentials must be delivered by
the OpenBao and External Secrets workflow from
[`06-openbao-and-external-secrets.md`](06-openbao-and-external-secrets.md), not
stored in `values.yaml`.

At minimum, the values must set the real cluster name, Kubernetes API address,
Traefik ingress class, hostname, TLS Secret, and non-secret application
settings. Use cert-manager for TLS only after staging issuance succeeds.

## Reconcile in a safe order

1. Create the OpenBao path and least-privilege policy for the application.
2. Sync the ExternalSecret and verify the target Secret exists without printing
   its contents.
3. Sync the Permission Manager Application.
4. Keep ingress disabled and test through port-forwarding first.
5. Review the application's effective RBAC before enabling ingress.

Inspect the effective resources with:

```bash
kubectl get serviceaccounts,roles,rolebindings \
  --namespace permission-manager
kubectl get clusterroles,clusterrolebindings | grep permission-manager
kubectl auth can-i --list \
  --as=system:serviceaccount:permission-manager:<service-account> \
  --namespace permission-manager
```

## Verify the application

```bash
kubectl get application permission-manager --namespace argo-cd
kubectl get pods,services,ingresses --namespace permission-manager
kubectl get events --namespace permission-manager --sort-by=.lastTimestamp
```

Use a service port-forward for the first login. Confirm credential rotation and
session behavior before exposing the UI through ingress.

## Definition of done

Permission Manager is implemented only when its chart source is immutable, the
rendered manifests pass review, credentials come from OpenBao, effective RBAC
is documented, TLS works, and an administrator can recover or disable access
without deleting the cluster-wide RBAC blindly.
