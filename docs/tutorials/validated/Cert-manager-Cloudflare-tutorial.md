# cert-manager, Cloudflare DNS-01, and Private HTTPS

This tutorial installs cert-manager through Argo CD, configures Let's Encrypt with Cloudflare DNS-01 validation, proves issuance with staging, enables production issuance, and adds HTTPS to Grafana, Argo CD, and Gitea. It contains only the completed working path; run the sections in order.

## 1. Completed outcome

At the end:

- cert-manager `v1.21.1` is managed by Argo CD.
- Its CRDs and ServiceMonitor are installed.
- Cloudflare DNS-01 validation issues certificates without exposing applications publicly.
- Staging issuance is proven before production is enabled.
- Grafana, Argo CD, and Gitea use separate, namespace-local TLS Secrets.
- Traefik terminates TLS; the applications continue using HTTP inside Kubernetes.

```text
LAN client --HTTPS--> Traefik at 192.168.4.240 --HTTP--> application Service

cert-manager --> Cloudflare DNS API --> _acme-challenge TXT record
       |
       +--> Let's Encrypt validates DNS and issues the certificate
```

## 2. Versions and final names

| Item | Value |
|---|---|
| cert-manager chart | `v1.21.1` |
| Chart repository | `https://charts.jetstack.io` |
| Ingress class | `traefik` |
| Staging issuer | `letsencrypt-staging` |
| Production issuer | `letsencrypt-prod` |
| Cloudflare Secret | `cloudflare-api-token` |
| Secret key | `api-token` |
| Private DNS suffix | `home.mikebrister.com` |
| Traefik LAN address | `192.168.4.240` |

## 3. Prerequisites

Complete the MicroK8s, Argo CD, Gitea, and kube-prometheus-stack tutorials first. You also need:

- control of `mikebrister.com` in Cloudflare;
- a Cloudflare API token permitted to edit DNS for that zone;
- LAN DNS entries for `argocd.home.mikebrister.com`, `gitea.home.mikebrister.com`, and `grafana.home.mikebrister.com`, all pointing to `192.168.4.240`;
- an email address for Let's Encrypt notices;
- the GitOps repository watched by the `homelab` root Application.

```bash
microk8s kubectl get ingressclass traefik
microk8s kubectl get application -n argo-cd homelab
```

Checkpoint: Traefik exists and `homelab` is `Synced` and `Healthy`.

## 4. Repository layout

```text
apps/
├── cert-manager/
│   └── values.yaml
└── cert-manager-config/
    ├── kustomization.yaml
    ├── clusterissuer-staging.yaml
    ├── clusterissuer-prod.yaml
    └── test-certificate.yaml

argocd/applications/
├── cert-manager.yaml
└── cert-manager-config.yaml
```

The first Application installs controllers and CRDs. The second creates resources that depend on those CRDs.

## 5. Install cert-manager

Create `apps/cert-manager/values.yaml`:

```yaml
crds:
  enabled: true

prometheus:
  enabled: true
  servicemonitor:
    enabled: true
```

Create `argocd/applications/cert-manager.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argo-cd
spec:
  project: default
  sources:
    - repoURL: https://charts.jetstack.io
      chart: cert-manager
      targetRevision: v1.21.1
      helm:
        releaseName: cert-manager
        valueFiles:
          - $values/apps/cert-manager/values.yaml
    - repoURL: https://github.com/michaelbrister/homelab-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

The second source points to the homelab GitOps repository and exposes its files as `$values`. Server-side apply handles the large CRDs reliably.

Render the pinned chart:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm template cert-manager jetstack/cert-manager \
  --version v1.21.1 \
  --namespace cert-manager \
  --values apps/cert-manager/values.yaml \
  > /tmp/cert-manager-rendered.yaml
grep '^kind: CustomResourceDefinition' /tmp/cert-manager-rendered.yaml | head
grep '^kind: Deployment' /tmp/cert-manager-rendered.yaml
```

Commit and push:

```bash
git add apps/cert-manager/values.yaml argocd/applications/cert-manager.yaml
git commit -m "Install cert-manager"
git push
```

Wait for and validate the installation:

```bash
microk8s kubectl rollout status deployment/cert-manager -n cert-manager
microk8s kubectl rollout status deployment/cert-manager-cainjector -n cert-manager
microk8s kubectl rollout status deployment/cert-manager-webhook -n cert-manager
microk8s kubectl get crd certificates.cert-manager.io \
  certificaterequests.cert-manager.io \
  clusterissuers.cert-manager.io \
  challenges.acme.cert-manager.io \
  orders.acme.cert-manager.io
microk8s kubectl get application -n argo-cd cert-manager
```

Checkpoint: all controllers are available, the CRDs exist, and the Application is healthy.

## 6. Bootstrap the Cloudflare token

Create a narrowly scoped Cloudflare token with DNS-edit permission for the required zone. Never put it in Git.

```bash
read -s CLOUDFLARE_TOKEN
echo
microk8s kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token="$CLOUDFLARE_TOKEN"
unset CLOUDFLARE_TOKEN
```

Verify the object and key without printing the value:

```bash
microk8s kubectl get secret cloudflare-api-token \
  -n cert-manager \
  -o jsonpath='{.data.api-token}' | wc -c
```

This one manual bootstrap Secret is replaced by an ESO-managed Secret in `Openbao-tutorial.md`.

## 7. Configure and prove staging issuance

Create `apps/cert-manager-config/clusterissuer-staging.yaml`. Replace `YOUR_EMAIL_ADDRESS` before committing.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: YOUR_EMAIL_ADDRESS
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

Create `apps/cert-manager-config/test-certificate.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-home-mikebrister-com
  namespace: cert-manager
spec:
  secretName: test-home-mikebrister-com-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
    - test.home.mikebrister.com
```

The staging certificate proves the whole ACME path without consuming production rate limits. Browsers will not trust it, which is expected.

Create `apps/cert-manager-config/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - clusterissuer-staging.yaml
  - test-certificate.yaml
```

Create `argocd/applications/cert-manager-config.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager-config
  namespace: argo-cd
spec:
  project: default
  source:
    repoURL: https://github.com/michaelbrister/homelab-gitops.git
    targetRevision: main
    path: apps/cert-manager-config
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
```

Render, commit, and push:

```bash
microk8s kubectl kustomize apps/cert-manager-config > /tmp/cert-manager-config.yaml
grep '^kind:' /tmp/cert-manager-config.yaml
git add apps/cert-manager-config argocd/applications/cert-manager-config.yaml
git commit -m "Configure staging ACME DNS validation"
git push
```

Wait for the successful ACME flow:

```bash
microk8s kubectl wait clusterissuer/letsencrypt-staging \
  --for=condition=Ready --timeout=180s
microk8s kubectl wait certificate/test-home-mikebrister-com \
  -n cert-manager --for=condition=Ready --timeout=300s
microk8s kubectl get certificaterequest,order,challenge -n cert-manager
microk8s kubectl get secret test-home-mikebrister-com-tls -n cert-manager
```

Checkpoint: the issuer and CertificateRequest are ready, the Order is valid, the Certificate is ready, and the TLS Secret exists. Do not proceed to production before this checkpoint passes.

## 8. Add the production issuer

Create `apps/cert-manager-config/clusterissuer-prod.yaml`, replacing the email address:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: YOUR_EMAIL_ADDRESS
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

Add it to the Kustomization:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - clusterissuer-staging.yaml
  - clusterissuer-prod.yaml
  - test-certificate.yaml
```

```bash
microk8s kubectl kustomize apps/cert-manager-config > /tmp/cert-manager-config.yaml
git add apps/cert-manager-config
git commit -m "Add production ACME issuer"
git push
microk8s kubectl wait clusterissuer/letsencrypt-prod \
  --for=condition=Ready --timeout=180s
```

Checkpoint: both ClusterIssuers report `Ready=True`.

## 9. Enable HTTPS for Grafana

Update the existing kube-prometheus-stack values so its `grafana` section includes the following. Preserve its persistence and other settings.

```yaml
grafana:
  ingress:
    enabled: true
    ingressClassName: traefik
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - grafana.home.mikebrister.com
    tls:
      - secretName: grafana-tls
        hosts:
          - grafana.home.mikebrister.com
```

Render using the pinned version from the monitoring tutorial, then commit:

```bash
helm template kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version "$KPS_CHART_VERSION" --namespace monitoring \
  --values apps/kube-prometheus-stack/values.yaml \
  > /tmp/kube-prometheus-stack-rendered.yaml
git add apps/kube-prometheus-stack/values.yaml
git commit -m "Enable HTTPS for Grafana"
git push
microk8s kubectl wait certificate/grafana-tls -n monitoring \
  --for=condition=Ready --timeout=300s
microk8s kubectl get secret grafana-tls -n monitoring
```

## 10. Enable HTTPS for Argo CD

Update `bootstrap/argocd/values.yaml`:

```yaml
configs:
  cm:
    url: https://argocd.home.mikebrister.com
  params:
    server.insecure: "true"

server:
  ingress:
    enabled: true
    ingressClassName: traefik
    hostname: argocd.home.mikebrister.com
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    pathType: Prefix
    paths:
      - /
    tls: true
```

`server.insecure: "true"` is correct because Traefik owns external TLS. This chart expects the TLS Secret name `argocd-server-tls`.

```bash
helmfile --file bootstrap/argocd/helmfile.yaml template > /tmp/argocd-rendered.yaml
helmfile --file bootstrap/argocd/helmfile.yaml diff
git add bootstrap/argocd/values.yaml
git commit -m "Enable HTTPS for Argo CD"
git push
helmfile --file bootstrap/argocd/helmfile.yaml apply
microk8s kubectl wait certificate/argocd-server-tls -n argo-cd \
  --for=condition=Ready --timeout=300s
microk8s kubectl get secret argocd-server-tls -n argo-cd
```

Argo CD remains a Helmfile-owned bootstrap component; it owns its child Applications.

## 11. Enable HTTPS for Gitea

Update the existing Gitea values, preserving its PostgreSQL and persistence settings:

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
      ROOT_URL: https://gitea.home.mikebrister.com/
      DOMAIN: gitea.home.mikebrister.com
      PROTOCOL: http
```

`PROTOCOL: http` matches TLS termination at Traefik. `ROOT_URL` remains HTTPS so redirects and clone links use the public address.

```bash
helm template gitea gitea-charts/gitea \
  --version "$GITEA_CHART_VERSION" --namespace gitea \
  --values apps/gitea/values.yaml > /tmp/gitea-rendered.yaml
git add apps/gitea/values.yaml
git commit -m "Enable HTTPS for Gitea"
git push
microk8s kubectl wait certificate/gitea-tls -n gitea \
  --for=condition=Ready --timeout=300s
microk8s kubectl get secret gitea-tls -n gitea
```

## 12. Validate HTTPS end to end

```bash
getent hosts argocd.home.mikebrister.com
getent hosts gitea.home.mikebrister.com
getent hosts grafana.home.mikebrister.com
curl -I https://argocd.home.mikebrister.com
curl -I https://gitea.home.mikebrister.com
curl -I https://grafana.home.mikebrister.com
```

Each name should resolve to `192.168.4.240`. Inspect a certificate's SAN:

```bash
openssl s_client -connect gitea.home.mikebrister.com:443 \
  -servername gitea.home.mikebrister.com </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer -subject -ext subjectAltName
```

Checkpoint: all three URLs complete a trusted TLS handshake, contain the expected hostname, load correctly, and remain healthy in Argo CD.

## 13. Validate self-healing

Add a harmless live annotation:

```bash
microk8s kubectl annotate ingress -n monitoring \
  kube-prometheus-stack-grafana homelab.example/self-heal-test=true
```

Argo CD removes the uncommitted drift. Verify that this returns an empty value:

```bash
microk8s kubectl get ingress -n monitoring \
  kube-prometheus-stack-grafana \
  -o jsonpath='{.metadata.annotations.homelab\.example/self-heal-test}'
```

## 14. Security, ownership, and rollback

- Keep the Cloudflare token out of Git, shell history, screenshots, and rendered manifests.
- Limit it to DNS edits for the required zone.
- DNS-01 needs no inbound internet path to Traefik; Let's Encrypt validates a temporary public TXT record.
- Each application has a separate TLS Secret in its own namespace.
- cert-manager owns CertificateRequests, Orders, Challenges, and generated TLS Secrets.
- Argo CD owns declarative Issuers, Certificates, Ingress settings, and the cert-manager release.

After production works, remove the staging proof by deleting `test-certificate.yaml` from the Kustomization and Git. Argo CD prunes the Certificate and its generated Secret.

To return one application to HTTP, revert only its TLS annotation, TLS block, and HTTPS public URL in Git. Do not remove the production ClusterIssuer while other Certificates reference it.

Before uninstalling cert-manager, first remove or replace every dependent Ingress and Certificate, then remove `cert-manager-config`, and remove the controller Application last.

## 15. Final checkpoint

The tutorial is complete when both issuers are ready; production certificates exist in `argo-cd`, `gitea`, and `monitoring`; all three private hostnames use trusted HTTPS; no application is publicly exposed; and the Cloudflare credential is absent from Git.
