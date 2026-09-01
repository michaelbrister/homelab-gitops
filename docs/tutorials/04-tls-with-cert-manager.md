# Tutorial 4: Issue TLS Certificates with cert-manager

The repository installs cert-manager and defines Let's Encrypt staging and
production `ClusterIssuer` resources using Cloudflare DNS-01 validation.

## Understand the dependency order

TLS requires these resources in order:

```text
cert-manager controller and CRDs
  -> cloudflare-api-token Secret in namespace cert-manager
     -> letsencrypt-staging / letsencrypt-prod ClusterIssuer
        -> Certificate or annotated Ingress
```

The Cloudflare token is delivered by External Secrets in the complete stack.
During initial bring-up, you can either complete
[Tutorial 6](06-openbao-and-external-secrets.md) first or create the Secret
manually as a temporary bootstrap step.

## Prepare the Cloudflare token

Create a narrowly scoped Cloudflare API token that can edit DNS records for the
zone used by the homelab. Do not put the token in a YAML file or commit it.

For a temporary bootstrap Secret, read the token without adding it to shell
history and create the Secret from standard input:

```bash
read -s CLOUDFLARE_API_TOKEN
echo
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token="$CLOUDFLARE_API_TOKEN"
unset CLOUDFLARE_API_TOKEN
```

The Secret name and key must match
`apps/cert-manager-config/clusterissuer-staging.yaml` and
`clusterissuer-prod.yaml`.

## Verify the issuers

Wait for cert-manager and the configuration Application to sync:

```bash
kubectl get application cert-manager --namespace argo-cd
kubectl get application cert-manager-config --namespace argo-cd
kubectl get clusterissuers
kubectl describe clusterissuer letsencrypt-staging
```

Do not test with the production issuer first. Let's Encrypt production has
stricter rate limits, while the staging issuer is intended for repeated setup
tests.

## Test certificate issuance

The repository includes `apps/cert-manager-config/test-certificate.yaml`.
Watch its status:

```bash
kubectl get certificate --namespace cert-manager --watch
```

In another terminal, inspect the ACME resources if issuance stalls:

```bash
kubectl get certificaterequests,orders,challenges --namespace cert-manager
kubectl describe certificate test-home-mikebrister-com --namespace cert-manager
kubectl logs --namespace cert-manager deployment/cert-manager --tail=100
```

The certificate is ready when this command prints `True`:

```bash
kubectl get certificate test-home-mikebrister-com \
  --namespace cert-manager \
  --output jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
echo
```

Staging certificates are intentionally not trusted by browsers.

## Add TLS to an application

For charts that support ingress annotations, follow the Gitea or Grafana
values already in this repository:

```yaml
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod
```

Set the ingress TLS host and a unique Secret name. cert-manager's ingress shim
will create a matching `Certificate`.

Before using production, confirm all of the following:

- the staging certificate became Ready;
- public DNS resolves the requested hostname correctly;
- the Cloudflare token is stored in OpenBao; and
- External Secrets owns the Kubernetes Secret instead of a manual bootstrap
  command.
