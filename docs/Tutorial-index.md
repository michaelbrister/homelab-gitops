# MicroK8s Homelab Tutorial Series

Follow these tutorials in order. Each guide starts from the validated result of the preceding guides and contains the complete working path for one installed part of the homelab.

## 1. MicroK8s foundation — validated

File: [MicroK8s-foundation-tutorial.md](tutorials/validated/MicroK8s-foundation-tutorial.md)

Installs the single-node Kubernetes base, DNS, hostpath storage, MetalLB, Traefik, and private LAN DNS. Ends with a real Ingress and persistence test.

## 2. Argo CD GitOps bootstrap — validated

File: [ArgoCD-gitops-bootstrap-tutorial.md](tutorials/validated/ArgoCD-gitops-bootstrap-tutorial.md)

Bootstraps Argo CD with Helmfile, creates the App-of-Apps repository structure, and proves automated sync, pruning, and self-healing.

## 3. Gitea and PostgreSQL — validated

File: [Gitea-tutorial.md](tutorials/validated/Gitea-tutorial.md)

Deploys Gitea and PostgreSQL with persistent volumes, exposes Gitea through Traefik, creates the administrator safely, and validates a real Git push.

## 4. kube-prometheus-stack — validated

File: [Kube-prometheus-stack-tutorial.md](tutorials/validated/Kube-prometheus-stack-tutorial.md)

Installs Prometheus Operator, Prometheus, Alertmanager, Grafana, kube-state-metrics, and node-exporter with persistent storage and functional metrics checks.

## 5. Loki — validated

File: [Loki-tutorial.md](tutorials/validated/Loki-tutorial.md)

Deploys a single-node monolithic Loki instance with TSDB, filesystem persistence, seven-day retention, and a Grafana data source.

## 6. Grafana Alloy — validated

File: [Grafana-Alloy-tutorial.md](tutorials/validated/Grafana-Alloy-tutorial.md)

Deploys Alloy with Kubernetes discovery and RBAC, forwards pod logs to Loki, and validates log queries in Grafana.

## 7. cert-manager, Cloudflare DNS-01, and HTTPS — validated

File: [Cert-manager-Cloudflare-tutorial.md](tutorials/validated/Cert-manager-Cloudflare-tutorial.md)

Installs cert-manager, proves Cloudflare DNS-01 with Let's Encrypt staging, enables production issuance, and adds private trusted HTTPS to Grafana, Argo CD, and Gitea.

## 8. External Secrets Operator — validated

File: [External-Secrets-Operator-tutorial.md](tutorials/validated/External-Secrets-Operator-tutorial.md)

Installs ESO `2.8.0`, validates its three controllers and CRDs, enables monitoring, and establishes the dependency boundary for OpenBao.

## 9. OpenBao, ESO integration, and Raft — validated

File: [Openbao-tutorial.md](tutorials/validated/Openbao-tutorial.md)

Installs OpenBao, configures static auto-unseal, Kubernetes authentication, ESO policy and role, migrates the Cloudflare token, proves GitOps self-healing, and migrates file storage to single-node integrated Raft.

## Implementation candidates

Tutorials 10–15 are detailed implementation candidates. They were not completed in the referenced homelab session, so each includes version-selection gates, safety checks, and validation criteria. Do not treat one as validated until its final checkpoint passes.

## 10. Longhorn — implementation candidate, not validated

File: [Longhorn-tutorial.md](tutorials/planned/Longhorn-tutorial.md)

Proves Longhorn on a disposable volume, then defines application-aware backup, restore, migration, restart, and rollback procedures for existing hostpath PVCs.

## 11. Cilium and Hubble — implementation candidate, not validated

File: [Cilium-Hubble-tutorial.md](tutorials/planned/Cilium-Hubble-tutorial.md)

Defines a rehearsed migration from the existing MicroK8s CNI to Cilium while initially retaining kube-proxy and MetalLB, then validates Hubble and network policy.

## 12. Kyverno — implementation candidate, not validated

File: [Kyverno-tutorial.md](tutorials/planned/Kyverno-tutorial.md)

Installs Kyverno, starts validation in Audit mode, reviews PolicyReports, and promotes one clean rule to enforcement.

## 13. Argo Rollouts — implementation candidate, not validated

File: [Argo-Rollouts-tutorial.md](tutorials/planned/Argo-Rollouts-tutorial.md)

Demonstrates weighted Traefik canaries, manual promotion, Prometheus analysis, controlled abort, and Git rollback with a disposable application.

## 14. Renovate — implementation candidate, not validated

File: [Renovate-tutorial.md](tutorials/planned/Renovate-tutorial.md)

Runs Renovate against Gitea, sources its token from OpenBao through ESO, and validates one reviewed update without automerge.

## 15. Atlantis — implementation candidate, not validated

File: [Atlantis-tutorial.md](tutorials/planned/Atlantis-tutorial.md)

Connects Atlantis to signed Gitea webhooks and remote Terraform state, then validates planning, locking, approval, apply, restart, and token rotation on disposable infrastructure.

## Dependency flow

```text
MicroK8s
   |
   v
Argo CD
   |
   +--> Gitea
   |
   +--> kube-prometheus-stack --> Loki --> Alloy
   |
   +--> cert-manager --> HTTPS for Argo CD, Gitea, and Grafana
   |
   +--> External Secrets Operator --> OpenBao --> Raft
                                             |
                                             v
Longhorn --> Cilium/Hubble --> Kyverno --> Argo Rollouts --> Renovate --> Atlantis
```

The sequence avoids circular dependencies: Argo CD is bootstrapped before it manages workloads; cert-manager exists before TLS is enabled; ESO exists before the OpenBao store is declared; and OpenBao's data is validated before storage is migrated to Raft. The implementation candidates then add storage and networking foundations before policy, delivery, and repository automation.
