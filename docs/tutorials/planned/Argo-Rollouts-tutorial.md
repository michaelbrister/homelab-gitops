# Argo Rollouts with Traefik and Prometheus

> Status: implementation candidate. Argo Rollouts was not installed in the referenced homelab session. Use a disposable demo application and mark the guide validated only after promotion, automated analysis, abort, and rollback all pass.

This tutorial installs the controller and demonstrates a weighted Traefik canary while keeping stable and canary Services explicit.

## 1. Select and pin the release

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo/argo-rollouts --versions | head -10
export ROLLOUTS_CHART_VERSION="REPLACE_WITH_TESTED_VERSION"
```

Install the matching `kubectl-argo-rollouts` CLI from the project's release artifacts and verify its version.

## 2. Install through Argo CD

Create `apps/argo-rollouts/values.yaml`:

```yaml
controller:
  replicas: 1
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true

dashboard:
  enabled: true
  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - rollouts.home.mikebrister.com
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    tls:
      - secretName: argo-rollouts-dashboard-tls
        hosts:
          - rollouts.home.mikebrister.com
```

Render the selected chart and adjust keys to its actual values schema.

Create `argocd/applications/argo-rollouts.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-rollouts
  namespace: argo-cd
spec:
  project: default
  sources:
    - repoURL: https://argoproj.github.io/argo-helm
      chart: argo-rollouts
      targetRevision: REPLACE_WITH_TESTED_VERSION
      helm:
        releaseName: argo-rollouts
        valueFiles:
          - $values/apps/argo-rollouts/values.yaml
    - repoURL: https://github.com/michaelbrister/homelab-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: argo-rollouts
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

```bash
helm template argo-rollouts argo/argo-rollouts \
  --version "$ROLLOUTS_CHART_VERSION" --namespace argo-rollouts \
  --values apps/argo-rollouts/values.yaml > /tmp/argo-rollouts.yaml
git add apps/argo-rollouts argocd/applications/argo-rollouts.yaml
git commit -m "Install Argo Rollouts"
git push
kubectl get pods -n argo-rollouts
kubectl get crd rollouts.argoproj.io analysistemplates.argoproj.io
kubectl get servicemonitor -n argo-rollouts
```

## 3. Create the demo application

Use namespace `rollouts-demo`. Create two Services with identical selectors except the controller-managed hash is omitted:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rollouts-demo-stable
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: rollouts-demo
---
apiVersion: v1
kind: Service
metadata:
  name: rollouts-demo-canary
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: rollouts-demo
```

Create a TraefikService without weights; Rollouts owns the weights dynamically:

```yaml
apiVersion: traefik.io/v1alpha1
kind: TraefikService
metadata:
  name: rollouts-demo
spec:
  weighted:
    services:
      - name: rollouts-demo-stable
        port: 80
      - name: rollouts-demo-canary
        port: 80
```

Create an IngressRoute pointing to that TraefikService, with cert-manager TLS configured using the tested Traefik CRD pattern. Use `rollouts-demo.home.mikebrister.com` and `rollouts-demo-tls`.

## 4. Create the Rollout

Create `apps/rollouts-demo/rollout.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: rollouts-demo
spec:
  replicas: 5
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: rollouts-demo
  template:
    metadata:
      labels:
        app: rollouts-demo
    spec:
      containers:
        - name: demo
          image: argoproj/rollouts-demo:blue
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
  strategy:
    canary:
      stableService: rollouts-demo-stable
      canaryService: rollouts-demo-canary
      trafficRouting:
        traefik:
          weightedTraefikServiceName: rollouts-demo
      steps:
        - setWeight: 10
        - pause: {duration: 2m}
        - setWeight: 25
        - pause: {}
        - setWeight: 50
        - pause: {duration: 2m}
        - setWeight: 100
```

Create a normal Kustomize-based child Application for `apps/rollouts-demo`, render it, commit, and sync. Confirm blue is stable:

```bash
kubectl argo rollouts get rollout rollouts-demo -n rollouts-demo --watch
curl https://rollouts-demo.home.mikebrister.com
```

## 5. Prove manual canary progression

Change the image in Git to `argoproj/rollouts-demo:yellow`, commit, and push. Observe the 10% and 25% stages in the CLI and browser. At the indefinite pause:

```bash
kubectl argo rollouts promote rollouts-demo -n rollouts-demo
```

Confirm the rollout completes and yellow becomes stable.

Important GitOps rule: do not declare `weight` fields in the Git-managed TraefikService. Argo Rollouts must be allowed to change them without Argo CD fighting the controller.

## 6. Add Prometheus analysis

Create an AnalysisTemplate that queries a demo-specific Prometheus metric. The exact PromQL must be tested against the labels in the live Prometheus data; do not copy a generic query that always returns an empty vector.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: rollout-success-rate
spec:
  metrics:
    - name: success-rate
      interval: 30s
      count: 5
      successCondition: result[0] >= 0.99
      failureLimit: 2
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
          query: REPLACE_WITH_VALIDATED_PROMQL
```

Reference it after an early canary step:

```yaml
- analysis:
    templates:
      - templateName: rollout-success-rate
```

First prove a successful analysis. Then use a controlled query threshold or failing demo version to prove the Rollout aborts while the stable Service remains healthy.

## 7. Rollback and self-heal

Use Git to restore the last good image. The emergency command:

```bash
kubectl argo rollouts abort rollouts-demo -n rollouts-demo
kubectl argo rollouts undo rollouts-demo -n rollouts-demo
```

is operational recovery, but commit the desired stable version to Git afterward so Argo CD and Rollouts agree.

Delete the controller pod, verify it resumes reconciliation, and confirm dashboard HTTPS, metrics, Services, and traffic weights.

## 8. Final checkpoint

Mark this tutorial validated only after initial deployment, weighted canary, manual promotion, successful analysis, controlled failed analysis, abort, Git rollback, controller restart, dashboard TLS, and Argo CD health all pass.

## Official references

- [Argo Rollouts traffic management](https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/)
- [Traefik integration](https://argo-rollouts.readthedocs.io/en/latest/features/traffic-management/traefik/)
