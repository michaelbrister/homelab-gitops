# Kyverno on MicroK8s: Audit First, Then Deny

> Status: implementation candidate. Kyverno was not installed in the referenced homelab session. Select a release compatible with MicroK8s, test every policy in Audit mode, and mark this guide validated only after all checkpoints pass.

This guide installs Kyverno through Argo CD, separates controller ownership from policy ownership, reports current violations, and promotes one low-risk rule to enforcement.

## 1. Target outcome

- Kyverno's admission, background, cleanup, and reports controllers are healthy.
- Prometheus collects Kyverno metrics.
- Policies live in a separate Argo CD Application.
- Initial policies run in Audit mode and produce PolicyReports.
- Narrow exceptions are documented in Git.
- One tested policy is promoted from Audit to Deny without breaking platform reconciliation.

## 2. Inventory before admission control

```bash
kubectl get namespaces
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.hostNetwork}{"\n"}{end}'
kubectl get pods -A -o yaml > /tmp/pods-before-kyverno.yaml
kubectl get application -n argo-cd
```

Record platform namespaces that contain privileged workloads, hostPath mounts, or host networking. Do not broadly exempt every workload namespace.

## 3. Select and pin Kyverno

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm search repo kyverno/kyverno --versions | head -10
export KYVERNO_CHART_VERSION="REPLACE_WITH_TESTED_VERSION"
```

Check the selected release's Kubernetes compatibility matrix. Use the same version for rendering and Argo CD.

## 4. Install the controllers

Create `apps/kyverno/values.yaml`:

```yaml
admissionController:
  replicas: 1
  serviceMonitor:
    enabled: true

backgroundController:
  replicas: 1

cleanupController:
  replicas: 1

reportsController:
  replicas: 1

features:
  policyExceptions:
    enabled: true
    namespace: kyverno
```

Create `argocd/applications/kyverno.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno
  namespace: argo-cd
spec:
  project: default
  sources:
    - repoURL: https://kyverno.github.io/kyverno/
      chart: kyverno
      targetRevision: REPLACE_WITH_TESTED_VERSION
      helm:
        releaseName: kyverno
        valueFiles:
          - $values/apps/kyverno/values.yaml
    - repoURL: https://github.com/michaelbrister/homelab-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: kyverno
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Render and inspect webhooks, CRDs, RBAC, and replica counts:

```bash
helm template kyverno kyverno/kyverno --version "$KYVERNO_CHART_VERSION" \
  --namespace kyverno --values apps/kyverno/values.yaml > /tmp/kyverno.yaml
grep '^kind:' /tmp/kyverno.yaml | sort | uniq -c
```

Replace the version placeholders, commit, and validate:

```bash
git add apps/kyverno argocd/applications/kyverno.yaml
git commit -m "Install Kyverno"
git push
kubectl get application -n argo-cd kyverno -w
kubectl get pods -n kyverno
kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration | grep kyverno
kubectl get servicemonitor -n kyverno
```

Checkpoint: every controller is ready before policies are added.

## 5. Create the policy Application

```text
apps/kyverno-policies/
├── kustomization.yaml
└── require-managed-by-label.yaml
argocd/applications/kyverno-policies.yaml
```

Label only the namespaces included in the first policy:

```bash
kubectl label namespace gitea policy.homelab/enabled=true
kubectl label namespace monitoring policy.homelab/enabled=true
```

Represent these labels in the namespace-owning Git manifests before continuing.

Create `apps/kyverno-policies/require-managed-by-label.yaml` using the current CEL-based API:

```yaml
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: require-managed-by-label
spec:
  validationActions:
    - Audit
  matchConstraints:
    namespaceSelector:
      matchLabels:
        policy.homelab/enabled: "true"
    resourceRules:
      - apiGroups: [apps]
        apiVersions: [v1]
        operations: [CREATE, UPDATE]
        resources: [deployments, statefulsets]
  validations:
    - message: Workloads must declare app.kubernetes.io/managed-by.
      expression: >-
        has(object.metadata.labels) &&
        'app.kubernetes.io/managed-by' in object.metadata.labels &&
        object.metadata.labels['app.kubernetes.io/managed-by'] != ''
```

Audit allows resources while reporting violations. The namespace label makes expansion deliberate and Git-reviewable.

Create the Kustomization:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - require-managed-by-label.yaml
```

Create `argocd/applications/kyverno-policies.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno-policies
  namespace: argo-cd
spec:
  project: default
  source:
    repoURL: https://github.com/michaelbrister/homelab-gitops.git
    targetRevision: main
    path: apps/kyverno-policies
  destination:
    server: https://kubernetes.default.svc
    namespace: kyverno
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl kustomize apps/kyverno-policies > /tmp/kyverno-policies.yaml
git add apps/kyverno-policies argocd/applications/kyverno-policies.yaml
git commit -m "Add Kyverno audit policy"
git push
```

## 6. Review Audit results

```bash
kubectl get validatingpolicy require-managed-by-label
kubectl get policyreport -A
kubectl get clusterpolicyreport
kubectl describe validatingpolicy require-managed-by-label
```

Correct violations in the owning Helm values or manifests. Do not patch generated workloads directly. Recheck reports after Argo CD applies the fixes.

## 7. Validate admission behavior

Create a disposable namespace and compliant workload:

```bash
kubectl create namespace policy-test
kubectl create deployment accepted -n policy-test --image=nginx:alpine
kubectl label deployment accepted -n policy-test app.kubernetes.io/managed-by=kubectl
```

The initial policy intentionally scopes only namespaces carrying `policy.homelab/enabled=true`, demonstrating that selection is explicit.

Before expanding scope, render all affected repository manifests and evaluate them with the matching Kyverno CLI version. Add the opt-in label to `policy-test`, commit, and confirm an unlabeled test Deployment is reported but allowed.

## 8. Exceptions

Prefer correcting the workload. When an exception is truly required, enable the selected release's current PolicyException API and create the narrowest possible exception: one policy, one rule, one namespace, and selected resource names. Include an owner, reason, and review date in annotations. Keep exceptions in `apps/kyverno-policies`.

Never use a wildcard exception for all platform namespaces.

## 9. Promote one rule from Audit to Deny

Only after the scoped report contains no unexplained failures, change:

```yaml
validationActions:
  - Deny
```

Commit and push. Prove a compliant object is admitted and a deliberately noncompliant object is rejected. Then verify every Argo CD Application remains healthy.

```bash
kubectl get application -n argo-cd
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

## 10. Restart and self-heal

```bash
kubectl delete pod -n kyverno -l app.kubernetes.io/part-of=kyverno
kubectl get pods -n kyverno -w
kubectl get validatingpolicy require-managed-by-label
```

Modify a harmless live policy field and confirm Argo CD restores Git. Never use a denial test against a critical namespace.

## 11. Rollback

If enforcement blocks legitimate reconciliation, revert the policy to Audit in Git and sync `kyverno-policies`. If admission itself becomes unavailable, pause policy rollout and use the pre-tested controller recovery procedure. Remove policies before uninstalling Kyverno.

## 12. Final checkpoint

Mark this tutorial validated only after controller readiness, metrics, Audit reports, corrected violations, a narrow exception test, compliant admission, controlled denial, restart, and GitOps self-heal all pass without degrading existing applications.

## Official references

- [Kyverno installation](https://kyverno.io/docs/installation/installation/)
- [Kyverno Policy Reports](https://kyverno.io/docs/guides/reports/)
- [Applying Kyverno policies](https://kyverno.io/docs/guides/applying-policies/)
