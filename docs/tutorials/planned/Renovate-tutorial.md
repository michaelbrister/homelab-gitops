# Self-Hosted Renovate for Gitea and Argo CD Manifests

> Status: implementation candidate. Renovate was not installed in the referenced homelab session. Start with one repository and no automerge; mark this guide validated only after a real update pull request is reviewed, merged, and reconciled safely.

This guide runs Renovate as a Kubernetes CronJob. Its Gitea token lives in OpenBao, ESO creates the runtime Secret, and repository-owned configuration controls update behavior.

## 1. Target outcome

- A dedicated Gitea bot opens dependency pull requests.
- Its token is stored in OpenBao and synced by ESO.
- Renovate scans only `michaelbrister/homelab-gitops` initially.
- Argo CD, Helmfile, Helm values, Kubernetes images, and supported files are discovered.
- Automerge is disabled.
- One update passes render checks, review, merge, and Argo CD reconciliation.

## 2. Create the bot and token

In Gitea, create a dedicated `renovate` user with full name and email configured. Grant only the repository permissions required to read code, push branches, and create/update pull requests and issues. Generate a personal access token following the permission matrix for the installed Gitea version.

Store the token in OpenBao without placing it in shell history:

```bash
read -s RENOVATE_GITEA_TOKEN
echo
kubectl exec -i -n openbao openbao-0 -- env BAO_TOKEN="$BAO_TOKEN" \
  bao kv put secret/homelab/renovate gitea_token="$RENOVATE_GITEA_TOKEN"
unset RENOVATE_GITEA_TOKEN
```

Use the established OpenBao administrative session method from `Openbao-tutorial.md`; never commit `BAO_TOKEN` or the Gitea token.

Extend the ESO policy so the Renovate role can read only `secret/data/homelab/renovate`, then update the OpenBao role to permit the `renovate` ServiceAccount in namespace `renovate`.

## 3. Create the ServiceAccount and ExternalSecret

Create `apps/renovate/serviceaccount.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: renovate
  namespace: renovate
```

Create `apps/renovate/external-secret.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: renovate
  namespace: renovate
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: openbao
  target:
    name: renovate
    creationPolicy: Owner
  data:
    - secretKey: RENOVATE_TOKEN
      remoteRef:
        key: homelab/renovate
        property: gitea_token
```

Confirm the live ESO CRD API version and adjust only if the installed ESO release serves a different version.

## 4. Select and pin Renovate

Use the official `ghcr.io/renovatebot/renovate` image. Select a fixed release tag from the official release stream; never use `latest`.

```bash
export RENOVATE_VERSION="REPLACE_WITH_TESTED_VERSION"
```

The standard image is sufficient for native Renovate managers. Choose the full image only if a validated manager requires external package-manager binaries.

## 5. Create global configuration

Create `apps/renovate/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: renovate-config
  namespace: renovate
data:
  config.json: |
    {
      "platform": "gitea",
      "endpoint": "https://gitea.home.mikebrister.com/api/v1/",
      "repositories": ["michaelbrister/homelab-gitops"],
      "onboarding": true,
      "requireConfig": "required",
      "dependencyDashboard": true,
      "automerge": false,
      "prConcurrentLimit": 3,
      "prHourlyLimit": 2,
      "logLevel": "info"
    }
```

Explicit repository selection is safer than autodiscovery for the first rollout.

## 6. Create the CronJob

Create `apps/renovate/cronjob.yaml` and replace the image tag:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: renovate
  namespace: renovate
spec:
  schedule: "17 4 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 1
      template:
        spec:
          serviceAccountName: renovate
          restartPolicy: Never
          containers:
            - name: renovate
              image: ghcr.io/renovatebot/renovate:REPLACE_WITH_TESTED_VERSION
              args:
                - --config-file=/etc/renovate/config.json
              envFrom:
                - secretRef:
                    name: renovate
              volumeMounts:
                - name: config
                  mountPath: /etc/renovate
                  readOnly: true
          volumes:
            - name: config
              configMap:
                name: renovate-config
```

## 7. Create Kustomize and Argo CD ownership

Create `apps/renovate/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - serviceaccount.yaml
  - external-secret.yaml
  - configmap.yaml
  - cronjob.yaml
```

Create `argocd/applications/renovate.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: renovate
  namespace: argo-cd
spec:
  project: default
  source:
    repoURL: https://github.com/michaelbrister/homelab-gitops.git
    targetRevision: main
    path: apps/renovate
  destination:
    server: https://kubernetes.default.svc
    namespace: renovate
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl kustomize apps/renovate > /tmp/renovate.yaml
grep -n 'REPLACE_WITH_TESTED_VERSION' /tmp/renovate.yaml
```

Replace the version, ensure no token is rendered, commit, and push.

## 8. Add repository configuration

Create `renovate.json` at the GitOps repository root:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "enabledManagers": ["argocd", "helmfile", "helm-values", "kubernetes"],
  "argocd": {
    "managerFilePatterns": ["/argocd/applications/.+\\.ya?ml$/"]
  },
  "kubernetes": {
    "managerFilePatterns": ["/apps/.+\\.ya?ml$/"]
  },
  "schedule": ["before 6am on saturday"],
  "timezone": "America/New_York",
  "rangeStrategy": "pin",
  "packageRules": [
    {
      "matchUpdateTypes": ["major"],
      "dependencyDashboardApproval": true
    },
    {
      "matchManagers": ["argocd"],
      "groupName": "Argo CD Helm chart updates"
    }
  ]
}
```

The Argo CD manager has no default filename pattern, so the explicit pattern is required. Keep patterns narrow to prevent false matches.

## 9. Dry run and first live run

Create a one-time Job from the CronJob and add Renovate's dry-run argument to that Job only:

```bash
kubectl create job -n renovate --from=cronjob/renovate renovate-dry-run
kubectl edit job -n renovate renovate-dry-run
```

Append `--dry-run=full`, then inspect logs:

```bash
kubectl logs -n renovate job/renovate-dry-run -f
```

Confirm the endpoint, repository, managers, and dependencies are correct and that no secret appears in logs.

Run a live one-time Job:

```bash
kubectl create job -n renovate --from=cronjob/renovate renovate-first-run
kubectl logs -n renovate job/renovate-first-run -f
```

Approve the onboarding pull request. The next run should create the dependency dashboard and controlled update PRs.

## 10. Validate one update end to end

Choose a patch or minor update. Review its release notes and rendered manifests. Run the same Helm/Kustomize checks documented by the affected tutorial. Merge it, then confirm Argo CD reaches `Synced` and `Healthy` and the application passes its functional test.

Do not enable automerge as part of this validation.

## 11. Rotation, restart, and rollback

Rotate the Gitea token, update the OpenBao value, force an ESO refresh, and run a new Job. Confirm authentication works before revoking the old token.

Rollback by suspending the CronJob in Git:

```yaml
spec:
  suspend: true
```

Existing Renovate branches and pull requests remain ordinary Gitea objects and can be closed without deleting the controller.

## 12. Final checkpoint

Mark the tutorial validated after ESO sync, dry-run discovery, onboarding, dependency dashboard, one reviewed update PR, successful render checks, merge, Argo reconciliation, token rotation, concurrency prevention, and suspended rollback all pass.

## Official references

- [Renovate Gitea platform](https://docs.renovatebot.com/modules/platform/gitea/)
- [Renovate managers](https://docs.renovatebot.com/modules/manager/)
- [Renovate Argo CD manager](https://docs.renovatebot.com/modules/manager/argocd/)
