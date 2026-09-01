# Atlantis with Gitea, OpenBao, and Remote Terraform State

> Status: implementation candidate. Atlantis was not installed in the referenced homelab session. Validate with disposable Terraform infrastructure and remote state; never make the first apply against irreplaceable homelab resources.

This guide deploys Atlantis through Argo CD, receives signed Gitea webhooks, stores credentials in OpenBao, and enforces approval, mergeability, and undiverged requirements before apply.

## 1. Target outcome

- Atlantis is privately available at `https://atlantis.home.mikebrister.com`.
- Gitea sends signed events to `/events`.
- VCS and backend credentials come from OpenBao through ESO.
- Only explicitly allowlisted repositories can run plans.
- Terraform state is remote and locked.
- A disposable pull request proves plan, approval, apply, locking, and cleanup.

Security warning: Terraform code and providers can execute commands. Treat repository access to Atlantis as infrastructure execution authority.

## 2. Prepare a disposable Terraform target

Create a separate Gitea repository containing a harmless resource and a remote backend with locking. Confirm command-line `terraform init`, `validate`, `plan`, `apply`, and `destroy` work before adding Atlantis.

Add `atlantis.yaml`:

```yaml
version: 3
projects:
  - name: homelab-test
    dir: .
    workspace: default
    autoplan:
      enabled: true
      when_modified:
        - "*.tf"
        - ".terraform.lock.hcl"
    plan_requirements:
      - undiverged
    apply_requirements:
      - approved
      - mergeable
      - undiverged
```

Protect `main` in Gitea and require a reviewer other than the pull-request author.

## 3. Create the Atlantis bot and webhook secret

Create a dedicated Gitea user named `atlantis`. Grant repository read/write, issue read/write, and user read permissions as required by the installed Gitea version. Generate its access token.

Generate a strong webhook secret locally:

```bash
openssl rand -hex 32
```

Store the Gitea token, webhook secret, and only the credentials needed for the disposable Terraform backend in OpenBao under `secret/homelab/atlantis`. Use interactive or file-based input so values do not enter Git or shell history.

Extend the ESO policy with read access only to `secret/data/homelab/atlantis`, and bind the OpenBao role to ServiceAccount `atlantis` in namespace `atlantis`.

## 4. Select and pin Atlantis

```bash
helm repo add runatlantis https://runatlantis.github.io/helm-charts
helm repo update
helm search repo runatlantis/atlantis --versions | head -10
export ATLANTIS_CHART_VERSION="REPLACE_WITH_TESTED_VERSION"
```

Confirm the chart application version is at least new enough to include native Gitea settings, including webhook-secret validation.

## 5. Create the ESO Secret

Create `apps/atlantis/external-secret.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: atlantis
  namespace: atlantis
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: openbao
  target:
    name: atlantis
    creationPolicy: Owner
  data:
    - secretKey: ATLANTIS_GITEA_TOKEN
      remoteRef:
        key: homelab/atlantis
        property: gitea_token
    - secretKey: ATLANTIS_GITEA_WEBHOOK_SECRET
      remoteRef:
        key: homelab/atlantis
        property: webhook_secret
    - secretKey: TF_BACKEND_CREDENTIAL
      remoteRef:
        key: homelab/atlantis
        property: backend_credential
```

Add environment keys required by the chosen backend without granting unrelated cloud or cluster privileges.

## 6. Create restricted server-side repository configuration

Create `apps/atlantis/repos.yaml`:

```yaml
repos:
  - id: gitea.home.mikebrister.com/michaelbrister/terraform-atlantis-test
    branch: /^main$/
    plan_requirements:
      - undiverged
    apply_requirements:
      - approved
      - mergeable
      - undiverged
    import_requirements:
      - approved
      - mergeable
      - undiverged
    allowed_overrides: []
    allow_custom_workflows: false
```

An exact allowlist is safer than a whole organization during validation. Repository code cannot weaken requirements or define arbitrary workflows.

## 7. Create chart values

Generate the selected chart's default values first:

```bash
helm show values runatlantis/atlantis --version "$ATLANTIS_CHART_VERSION" > /tmp/atlantis-default-values.yaml
```

Create `apps/atlantis/values.yaml`, adapting key names to that exact chart:

```yaml
orgAllowlist: gitea.home.mikebrister.com/michaelbrister/terraform-atlantis-test

gitea:
  user: atlantis
  baseUrl: https://gitea.home.mikebrister.com

environment:
  ATLANTIS_GITEA_USER: atlantis
  ATLANTIS_GITEA_BASE_URL: https://gitea.home.mikebrister.com
  ATLANTIS_REPO_ALLOWLIST: gitea.home.mikebrister.com/michaelbrister/terraform-atlantis-test

environmentSecrets:
  - name: ATLANTIS_GITEA_TOKEN
    secretKeyRef:
      name: atlantis
      key: ATLANTIS_GITEA_TOKEN
  - name: ATLANTIS_GITEA_WEBHOOK_SECRET
    secretKeyRef:
      name: atlantis
      key: ATLANTIS_GITEA_WEBHOOK_SECRET
  - name: TF_BACKEND_CREDENTIAL
    secretKeyRef:
      name: atlantis
      key: TF_BACKEND_CREDENTIAL

repoConfig: |
  repos:
    - id: gitea.home.mikebrister.com/michaelbrister/terraform-atlantis-test
      branch: /^main$/
      plan_requirements: [undiverged]
      apply_requirements: [approved, mergeable, undiverged]
      import_requirements: [approved, mergeable, undiverged]
      allowed_overrides: []
      allow_custom_workflows: false

dataStorage: 5Gi
storageClassName: longhorn

ingress:
  enabled: true
  ingressClassName: traefik
  host: atlantis.home.mikebrister.com
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  tls:
    - secretName: atlantis-tls
      hosts:
        - atlantis.home.mikebrister.com

serviceMonitor:
  enabled: true
```

Chart keys vary. The rendered StatefulSet—not this template—is authoritative. Confirm every `ATLANTIS_*` variable, Secret reference, PVC, Service, Ingress, and config mount.

## 8. Create the Argo CD Application

Create `argocd/applications/atlantis.yaml` as a multi-source Application using chart `atlantis` from `https://runatlantis.github.io/helm-charts`, the pinned version, and `$values/apps/atlantis/values.yaml`. Add `apps/atlantis/external-secret.yaml` through a Kustomize source or a separate configuration Application so the Secret exists before the StatefulSet starts.

Render first:

```bash
helm template atlantis runatlantis/atlantis \
  --version "$ATLANTIS_CHART_VERSION" --namespace atlantis \
  --values apps/atlantis/values.yaml > /tmp/atlantis.yaml
grep '^kind:' /tmp/atlantis.yaml | sort | uniq -c
grep -n 'ATLANTIS_GITEA\|ATLANTIS_REPO_ALLOWLIST' /tmp/atlantis.yaml
```

Ensure no secret values appear. Commit, push, and validate:

```bash
kubectl get externalsecret -n atlantis
kubectl get secret -n atlantis atlantis
kubectl get pods,pvc,service,ingress -n atlantis
kubectl wait certificate/atlantis-tls -n atlantis --for=condition=Ready --timeout=300s
curl -I https://atlantis.home.mikebrister.com
```

## 9. Configure the Gitea webhook

In the test repository, add a Gitea webhook:

```text
Target URL: https://atlantis.home.mikebrister.com/events
Secret: the OpenBao-stored webhook secret
Events: pull request, issue comment, and push events required by Atlantis
SSL verification: enabled
```

Send a test delivery and verify a successful response in Gitea and Atlantis logs. A missing webhook secret is a security failure; do not proceed.

## 10. Prove plan, locks, and apply

1. Create a branch that changes the disposable Terraform resource.
2. Open a pull request.
3. Confirm Atlantis posts a plan.
4. Open a second conflicting pull request and confirm the project lock prevents a competing plan/apply.
5. Attempt `atlantis apply` before approval and confirm it is rejected.
6. Have another authorized user approve the pull request.
7. Confirm it is mergeable and undiverged.
8. Comment `atlantis apply`.
9. Verify remote state and the disposable target.
10. Merge and confirm the lock clears.

Review logs and plan comments to ensure sensitive backend values were not printed.

## 11. Persistence, restart, and rotation

Create a new plan, then delete the Atlantis pod. Confirm its PVC reattaches and the stored plan remains usable. If the plan is lost, rerun it rather than bypassing controls.

Rotate the Gitea token and webhook secret through OpenBao and ESO. Update the Gitea webhook secret in the same controlled window, restart Atlantis if its selected release does not reload secrets, and prove a new delivery.

## 12. Rollback and cleanup

Suspend external access by removing the Ingress in Git or disabling the webhook in Gitea. Finish or discard active plans and locks before uninstalling. Destroy only the disposable test resource through the reviewed workflow. Retain remote state according to its recovery policy.

Never delete the Atlantis PVC while an approved plan is expected to be applied.

## 13. Final checkpoint

Mark this tutorial validated only after signed webhook delivery, allowlist enforcement, plan comment, competing lock, rejected unauthorized apply, approved apply, remote-state verification, pod restart, secret rotation, metrics, HTTPS, and Argo CD self-heal all pass.

## Official references

- [Atlantis deployment](https://www.runatlantis.io/docs/deployment.html)
- [Gitea webhook configuration](https://www.runatlantis.io/docs/configuring-webhooks.html)
- [Atlantis server configuration](https://www.runatlantis.io/docs/server-configuration.html)
- [Server-side repository configuration](https://www.runatlantis.io/docs/server-side-repo-config)
