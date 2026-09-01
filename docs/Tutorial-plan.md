# Homelab Tutorial Series Plan

Tutorials 1–9 cover components that were successfully installed and validated. Tutorials 10–15 now have implementation-candidate files, but remain planned extensions until their implementations pass every listed checkpoint. Each finished tutorial will use the same format as `Openbao-tutorial.md` and present only the validated working path, in dependency order, without troubleshooting history.

## Standard tutorial format

Every tutorial will contain:

1. Purpose and completed outcome
2. Tested component and chart versions
3. Prerequisites and dependencies
4. Final repository layout
5. Ordered implementation steps
6. Complete working manifests or Helm values
7. Local render or validation before deployment
8. Git commit and Argo CD reconciliation
9. Checkpoints after each major state change
10. Functional validation, not merely pod health
11. Restart, persistence, or self-heal validation where applicable
12. Security and storage warnings
13. Cleanup and rollback notes
14. Final architecture diagram

Common rules for the series:

- Include only steps that contributed to the successful final state.
- Do not include failed manifests, obsolete value keys, or debugging detours.
- Put commands in the exact order they should be run.
- Stop at explicit checkpoints before destructive or state-changing operations.
- Keep secret values out of Git and command examples.
- Pin Helm chart versions.
- Render Helm or Kustomize output before committing.
- Prefer Git and Argo CD reconciliation over direct live-cluster edits.
- Explain which controller owns each resource.
- Label unimplemented tutorials as planned; do not write their final instructions from assumptions.

Before drafting each tutorial, confirm any version not explicitly preserved in the conversation from the final Git manifest or the live Argo CD Application. Do not substitute a current/latest version for the version that was actually validated.

## Recommended tutorial order

```text
1. MicroK8s foundation
        |
        v
2. Argo CD GitOps bootstrap
        |
        v
3. Gitea and PostgreSQL
        |
        v
4. kube-prometheus-stack
        |
        v
5. Loki
        |
        v
6. Grafana Alloy
        |
        v
7. cert-manager, Cloudflare DNS-01, and HTTPS
        |
        v
8. External Secrets Operator
        |
        v
9. OpenBao, ESO integration, and Raft
        |
        v
10. Longhorn storage migration
        |
        v
11. Cilium networking and Hubble
        |
        v
12. Kyverno policy management
        |
        v
13. Argo Rollouts progressive delivery
        |
        v
14. Renovate dependency automation
        |
        v
15. Atlantis Terraform pull-request automation
```

Tutorials 1–9 are validated. Draft files for tutorials 10–15 exist and must retain their implementation-candidate warning until each installation is complete and validated.

## Tutorial 1: MicroK8s foundation

Proposed filename:

```text
MicroK8s-foundation-tutorial.md
```

### Purpose

Document the base single-node Kubernetes environment required by every later tutorial.

### Scope

- Install and verify MicroK8s.
- Configure local client access to the cluster.
- Enable or install cluster DNS.
- Enable `microk8s-hostpath` storage.
- Configure MetalLB for LAN service addresses.
- Install or enable Traefik as the ingress controller.
- Confirm the working ingress classes.
- Configure LAN DNS for the ingress address.
- Explain the single-node and hostpath durability boundaries.

### Key completed values

```text
StorageClass: microk8s-hostpath
Volume binding: WaitForFirstConsumer
Ingress class: traefik
Ingress address: 192.168.4.240
Initial LAN domain: *.brister.lan
Final LAN domain: *.home.mikebrister.com
```

### Ordered sections

1. Host and network prerequisites
2. MicroK8s installation
3. Cluster readiness validation
4. DNS and hostpath storage
5. MetalLB address configuration
6. Traefik ingress installation
7. IngressClass verification
8. LAN wildcard DNS
9. Test service and Ingress
10. Restart validation
11. Storage and single-node limitations
12. Final traffic architecture

### Completion checkpoints

- MicroK8s reports ready.
- Core system pods are running.
- `microk8s-hostpath` is the default StorageClass.
- A test PVC becomes `Bound` after its consumer is scheduled.
- Traefik has the `192.168.4.240` ingress address.
- A LAN hostname resolves to that address and reaches a test workload.

## Tutorial 2: Argo CD bootstrap and App-of-Apps

Proposed filename:

```text
ArgoCD-gitops-bootstrap-tutorial.md
```

### Purpose

Document how Argo CD was bootstrapped with Helmfile and then made responsible for all later workloads through a root Application.

### Scope

- Create the GitOps repository layout.
- Bootstrap Argo CD from `bootstrap/argocd` with Helmfile.
- Keep Argo CD outside its own initial dependency loop.
- Configure the Argo CD server behind Traefik.
- Create `argocd/root-app.yaml`.
- Point the root Application at `argocd/applications`.
- Explain why an empty directory needs a tracked placeholder.
- Validate the App-of-Apps discovery path.
- Demonstrate automated sync, prune, and self-heal.

### Repository layout

```text
bootstrap/
└── argocd/
    ├── helmfile.yaml
    └── values.yaml

argocd/
├── root-app.yaml
└── applications/
    └── .gitkeep

apps/
```

### Ordered sections

1. Repository and tool prerequisites
2. Argo CD Helmfile definition
3. Bootstrap values
4. Local Helmfile template and diff
5. Install Argo CD
6. Verify pods, services, and initial ingress
7. Create the root Application
8. Track the child-Application directory
9. Apply the one-time root Application
10. Validate `homelab` as `Synced` and `Healthy`
11. Add a harmless child Application test
12. Validate automated reconciliation
13. Explain the bootstrap ownership boundary

### Completion checkpoints

- Argo CD components are healthy in `argo-cd`.
- The UI is reachable through Traefik.
- The `homelab` root Application is `Synced` and `Healthy`.
- A child manifest committed under `argocd/applications` is discovered automatically.
- A Git-managed field altered in the cluster is restored by Argo CD.

## Tutorial 3: Gitea with PostgreSQL and persistent storage

Proposed filename:

```text
Gitea-tutorial.md
```

### Purpose

Deploy a persistent, single-instance Gitea service as the first real child Application managed by the App-of-Apps structure.

### Scope

- Use the official Gitea Helm chart through an Argo CD multi-source Application.
- Keep Helm values in `apps/gitea/values.yaml`.
- Run one Gitea instance with one PostgreSQL instance.
- Persist both Gitea and PostgreSQL data with `microk8s-hostpath`.
- Disable unused Redis and PostgreSQL HA modes.
- Enable push-to-create for user repositories.
- Expose Gitea through Traefik.
- Create the initial administrator without putting its password in Git.
- Validate a real Git push.
- End with the final HTTPS hostname while keeping Gitea HTTP behind Traefik.

### Final configuration points

```text
External URL: https://gitea.home.mikebrister.com/
Internal protocol: HTTP
Ingress class field: ingressClassName
TLS secret: gitea-tls
Gitea PVC: 10Gi microk8s-hostpath
PostgreSQL PVC: 10Gi microk8s-hostpath
PostgreSQL HA: disabled
Redis: disabled
Push-to-create: enabled
```

### Ordered sections

1. Verify StorageClass and IngressClass
2. Create Gitea repository files
3. Pin the Gitea chart
4. Configure Gitea, PostgreSQL, and persistence
5. Create the multi-source Argo CD Application
6. Render the chart locally
7. Commit and let the root Application discover Gitea
8. Validate pods, services, ingress, and PVCs
9. Create the initial administrator
10. Enable user push-to-create
11. Create and push a test repository
12. Apply final HTTPS ingress settings
13. Validate certificate, browser access, and Git-over-HTTPS
14. Backup and recovery notes

### Completion checkpoints

- Argo reports Gitea `Synced` and `Healthy`.
- Gitea and PostgreSQL pods remain stable.
- Both PVCs are `Bound`.
- The admin account works.
- A test repository accepts a Git push.
- HTTPS presents the correct certificate.
- Gitea reports HTTPS URLs while Traefik connects to it over HTTP.

## Tutorial 4: kube-prometheus-stack

Proposed filename:

```text
Kube-prometheus-stack-tutorial.md
```

### Purpose

Install a persistent metrics and alerting stack and verify that it is collecting useful MicroK8s data.

### Scope

- Deploy kube-prometheus-stack from the Prometheus Community Helm repository.
- Use an Argo CD multi-source Application.
- Enable server-side apply for large CRDs.
- Deploy Prometheus Operator, Prometheus, Alertmanager, Grafana, node-exporter, and kube-state-metrics.
- Persist Grafana, Prometheus, and Alertmanager.
- Configure seven-day Prometheus retention.
- Expose Grafana through Traefik.
- Retrieve the generated Grafana administrator password safely.
- Validate dashboards and ServiceMonitor discovery.

### Completed storage settings

```text
Grafana:      5Gi
Prometheus:  10Gi
Alertmanager: 2Gi
StorageClass: microk8s-hostpath
Prometheus retention: 7d
```

### Ordered sections

1. Create the application files
2. Pin the chart version
3. Configure server-side apply
4. Configure Grafana, Prometheus, and Alertmanager persistence
5. Create the Argo CD Application
6. Render and inspect CRDs
7. Commit and deploy
8. Validate pods and PVCs
9. Retrieve the Grafana administrator password
10. Validate the Prometheus datasource
11. Validate node and Kubernetes dashboards
12. Inspect Prometheus Operator CRDs
13. Inspect ServiceMonitors
14. Validate Grafana ingress and HTTPS
15. Persistence and retention notes

### Completion checkpoints

- The Application is `Synced` and `Healthy`.
- Monitoring pods are running.
- All three PVCs are `Bound`.
- Grafana is reachable.
- Prometheus is an active datasource.
- Node CPU, memory, filesystem, and network data are visible.
- ServiceMonitor resources are being discovered.

## Tutorial 5: Loki in monolithic mode

Proposed filename:

```text
Loki-tutorial.md
```

### Purpose

Add persistent, single-node log storage to the monitoring namespace and provision it as a Grafana datasource.

### Scope

- Deploy the Grafana Community Loki chart.
- Pin chart `18.7.6` from the completed setup.
- Use monolithic mode with one replica.
- Use TSDB schema v13 and filesystem storage.
- Persist logs on a 10Gi hostpath PVC.
- Set seven-day retention.
- Disable distributed components and MinIO.
- Use `loki-gateway` as the common API endpoint.
- Provision the Loki datasource through kube-prometheus-stack values.

### Final configuration points

```text
Deployment mode: Monolithic
Replication factor: 1
Schema: v13
Object store: filesystem
Retention: 168h
PVC: 10Gi microk8s-hostpath
Grafana URL: http://loki-gateway.monitoring.svc.cluster.local
```

### Ordered sections

1. Create Loki repository files
2. Add the Grafana Community Helm repository
3. Pin chart `18.7.6`
4. Configure monolithic filesystem storage
5. Configure required chart bucket-name placeholders
6. Disable distributed modes and MinIO
7. Create the Argo CD Application
8. Render locally
9. Commit and deploy
10. Validate pod, service, and PVC
11. Validate the Loki process readiness endpoint
12. Validate the gateway API endpoint
13. Add the Grafana datasource declaratively
14. Confirm Grafana can reach Loki
15. Retention and storage limitations

### Completion checkpoints

- Loki is `Synced` and `Healthy`.
- The single-binary pod is running.
- Its PVC is `Bound`.
- The Loki process returns `ready`.
- The gateway responds to Loki API requests.
- Grafana shows the provisioned Loki datasource.

## Tutorial 6: Grafana Alloy Kubernetes log collection

Proposed filename:

```text
Grafana-Alloy-tutorial.md
```

### Purpose

Collect Kubernetes pod logs and send them to Loki so they can be queried in Grafana.

### Scope

- Deploy Alloy chart `1.11.1` from Grafana's main Helm repository.
- Run one Alloy Deployment.
- Create the required RBAC.
- Discover pods through the Kubernetes API.
- Relabel namespace, pod, container, and application metadata.
- Tail pod logs with `loki.source.kubernetes`.
- Send logs to `loki-gateway`.
- Validate logs with LogQL in Grafana Explore.

### Final data path

```text
Kubernetes API
      |
      v
discovery.kubernetes
      |
      v
discovery.relabel
      |
      v
loki.source.kubernetes
      |
      v
loki.write
      |
      v
loki-gateway -> Loki -> Grafana
```

### Ordered sections

1. Confirm Loki and its Grafana datasource
2. Create Alloy repository files
3. Add Grafana's Helm repository
4. Pin Alloy chart `1.11.1`
5. Configure Deployment and RBAC
6. Configure discovery and relabeling
7. Configure Kubernetes log collection
8. Configure the Loki write endpoint
9. Create the Argo CD Application
10. Render locally
11. Commit and deploy
12. Validate Alloy pod and permissions
13. Validate successful Loki writes
14. Query monitoring namespace logs
15. Query Gitea namespace logs
16. Explain labels and query cardinality

### Completion checkpoints

- Alloy is `Synced` and `Healthy`.
- Its pod is running without repeated permission or push errors.
- `{namespace="monitoring"}` returns logs in Grafana Explore.
- `{namespace="gitea"}` returns Gitea or PostgreSQL logs.
- Results include useful pod, container, namespace, and app labels.

## Tutorial 7: cert-manager, Cloudflare DNS-01, and HTTPS

Proposed filename:

```text
Cert-manager-Cloudflare-tutorial.md
```

### Purpose

Install cert-manager, prove Cloudflare DNS-01 with Let's Encrypt staging, create a production issuer, and migrate the homelab UIs to trusted HTTPS without exposing the cluster publicly.

### Scope

- Deploy cert-manager `v1.21.1` through Argo CD.
- Install CRDs with server-side apply.
- Enable Prometheus monitoring and a ServiceMonitor.
- Bootstrap a narrowly scoped Cloudflare API token as a Kubernetes Secret.
- Create a staging ClusterIssuer.
- Issue and validate a staging certificate.
- Create the production ClusterIssuer.
- Use split DNS for `*.home.mikebrister.com`.
- Configure Grafana, Argo CD, and Gitea ingress TLS.
- Keep TLS termination at Traefik.
- Validate public trust and service routing.
- Note that the Cloudflare token is moved to OpenBao in tutorial 9.

### Final hostnames

```text
https://grafana.home.mikebrister.com
https://argocd.home.mikebrister.com
https://gitea.home.mikebrister.com
```

### Ordered sections

1. Domain and Cloudflare prerequisites
2. Internal wildcard DNS to `192.168.4.240`
3. Create cert-manager values
4. Create the Argo CD Application
5. Render and validate CRDs
6. Commit and deploy cert-manager
7. Validate controller, cainjector, and webhook
8. Create the Cloudflare token Secret
9. Create the staging ClusterIssuer
10. Create the cert-manager configuration Application
11. Issue a staging test certificate
12. Validate CertificateRequest and ACME Order
13. Create the production ClusterIssuer
14. Migrate Grafana ingress to HTTPS
15. Migrate Argo CD ingress to HTTPS
16. Migrate Gitea ingress to HTTPS
17. Validate TLS Secrets and certificate SANs
18. Validate browser and `curl` access
19. Explain renewal and namespace-local TLS Secrets
20. Bootstrap-secret handoff note for OpenBao

### Important final application settings

Argo CD:

```text
External URL: HTTPS
Traefik terminates TLS
server.insecure: true
TLS Secret: argocd-server-tls
```

Gitea:

```text
ROOT_URL: https://gitea.home.mikebrister.com/
PROTOCOL: http
Ingress class key: ingressClassName
TLS Secret: gitea-tls
```

Grafana:

```text
Ingress managed by kube-prometheus-stack
Issuer annotation: letsencrypt-prod
TLS Secret: grafana-tls
```

### Completion checkpoints

- cert-manager is `Synced` and `Healthy`.
- Its three controllers are available.
- The staging issuer is ready.
- The staging CertificateRequest is ready and its Order is valid.
- The production issuer is ready.
- Each application has a namespace-local TLS Secret.
- All three HTTPS URLs return trusted certificates.
- Internal traffic remains LAN-only.

## Tutorial 8: External Secrets Operator

Proposed filename:

```text
External-Secrets-Operator-tutorial.md
```

### Purpose

Install ESO as a GitOps-managed platform component and validate its controllers and CRDs before connecting it to OpenBao.

### Scope

- Deploy ESO chart `2.8.0` from the official chart repository.
- Install ESO CRDs.
- Use server-side apply for the CRDs.
- Run one controller replica.
- Enable the ServiceMonitor.
- Validate the main controller, webhook, and cert-controller.
- Validate the ExternalSecret, SecretStore, and ClusterSecretStore APIs.
- Define the responsibility boundary between the operator installation and backend configuration.
- Hand off to the OpenBao tutorial for provider authentication and real secret synchronization.

### Ordered sections

1. Confirm cert-manager and monitoring prerequisites
2. Create ESO values
3. Create the Argo CD multi-source Application
4. Pin chart `2.8.0`
5. Enable CRDs and server-side apply
6. Render locally
7. Commit and deploy
8. Validate all ESO pods
9. Validate CRDs and API resources
10. Validate the ServiceMonitor
11. Explain `ExternalSecret`, `SecretStore`, and `ClusterSecretStore`
12. Explain that installation alone does not configure a provider
13. Hand off to `Openbao-tutorial.md`

### Completion checkpoints

- The ESO Application is `Synced` and `Healthy`.
- The operator, webhook, and cert-controller pods are running.
- ESO CRDs exist.
- Kubernetes exposes the expected ESO API resources.
- Prometheus discovers the ESO ServiceMonitor.

## Tutorial 9: OpenBao, ESO integration, and Raft

Existing filename:

```text
Openbao-tutorial.md
```

This tutorial is already complete and covers:

- OpenBao installation through Helm and Argo CD
- Shamir initialization
- migration to static auto-unseal
- KV v2
- Kubernetes authentication
- TokenReview RBAC
- ESO policy and role
- ESO 2.8.0 Vault-provider compatibility
- Cloudflare token migration
- Argo CD ownership and self-heal
- file-to-Raft migration
- single-node HA/Raft cutover
- restart, Raft leadership, and ESO validation
- cleanup and rollback notes

## Tutorial 10: Longhorn persistent storage migration — planned

Proposed filename:

```text
Longhorn-tutorial.md
```

### Purpose

Install Longhorn as the homelab's managed block-storage platform, validate it on the single MicroK8s node, and migrate selected workloads from `microk8s-hostpath` without risking the existing stateful data.

### Prerequisites and decisions

- Complete tutorials 1–9 and take application-level backups first.
- Record every existing PVC, PV, StorageClass, reclaim policy, node path, and consuming workload.
- Confirm the node satisfies the Longhorn host requirements before installation.
- Reserve a dedicated Longhorn data path and decide its capacity.
- Use one replica for this single-node lab and state clearly that this is not node-level redundancy.
- Pin the validated Longhorn chart and application versions.
- Decide which existing PVCs will remain on hostpath and which will migrate.

### Scope

- Run and record the Longhorn environment preflight.
- Install Longhorn through a multi-source Argo CD Application.
- Configure a single-node-appropriate default replica count and data path.
- Expose the Longhorn UI privately through Traefik and cert-manager.
- Add ServiceMonitor integration when supported by the tested release.
- Validate dynamic provisioning, attachment, filesystem writes, pod recreation, and remounting.
- Create and restore a Longhorn snapshot or backup using the selected target.
- Migrate stateful workloads one at a time with application-aware backup and restore procedures.
- Retain old hostpath volumes through an explicit rollback window.

### Ordered sections

1. Inventory the existing storage and application backups
2. Explain single-node durability limits
3. Validate host packages, kernel modules, mounts, and free capacity
4. Select and pin the tested Longhorn release
5. Create Longhorn values and the Argo CD Application
6. Render manifests and inspect privileged resources
7. Commit, sync, and validate Longhorn system components
8. Configure the data path, replica count, reclaim policy, and StorageClass
9. Add private HTTPS access to the Longhorn UI
10. Provision a disposable PVC and write test data
11. Delete and recreate its consumer pod
12. Validate volume detach, attach, remount, and data persistence
13. Create and restore a test snapshot or backup
14. Choose the first low-risk workload to migrate
15. Stop writes and take an application-consistent backup
16. Create the Longhorn-backed replacement PVC
17. Restore data and update the Git-managed workload
18. Validate the application before migrating another workload
19. Define the old-PVC retention and rollback window
20. Repeat the migration pattern for approved workloads

### Required safety gates

- Do not make Longhorn the default StorageClass before the disposable-volume test passes.
- Do not migrate OpenBao, PostgreSQL, Prometheus, Loki, or Grafana without an application-specific restore test.
- Do not delete an old hostpath PVC during the migration commit.
- Stop writers or use the application's supported backup mechanism before copying data.
- Keep an out-of-cluster backup; one Longhorn replica on one node is not a backup.

### Completion checkpoints

- The Longhorn Application is `Synced` and `Healthy`.
- All required Longhorn components are ready.
- A test PVC provisions, attaches, remounts, and retains data.
- A snapshot or backup restores successfully.
- The UI is reachable only through private HTTPS.
- At least one approved workload completes a backup, restore, cutover, restart, and rollback rehearsal.
- Old volumes remain available for the documented rollback period.

## Tutorial 11: Cilium networking and Hubble — planned

Proposed filename:

```text
Cilium-Hubble-tutorial.md
```

### Purpose

Replace or augment the existing MicroK8s networking stack with Cilium, enable Hubble network observability, and prove pod, service, DNS, ingress, and LoadBalancer connectivity without losing access to the cluster.

### Prerequisites and decisions

- Complete and back up tutorials 1–10.
- Record the active CNI, pod CIDR, service CIDR, kube-proxy mode, MetalLB configuration, Traefik service, and NetworkPolicies.
- Confirm the exact MicroK8s and kernel compatibility for the chosen Cilium release.
- Decide and document whether kube-proxy remains enabled or is replaced.
- Decide whether Cilium or MetalLB will own LoadBalancer address advertisement; never let both advertise the same pool unintentionally.
- Schedule a maintenance window and retain console access to the node.

### Scope

- Capture a working pre-migration connectivity baseline.
- Select and pin compatible Cilium CLI, chart, and application versions.
- Define the final CNI and kube-proxy configuration explicitly.
- Install Cilium through a controlled bootstrap/GitOps boundary.
- Enable Hubble Relay and the Hubble UI privately.
- Preserve Traefik and the existing `192.168.4.240` application address.
- Add Cilium and Hubble metrics to Prometheus and Grafana.
- Validate DNS, ClusterIP, pod-to-pod, ingress, LoadBalancer, egress, and host connectivity.
- Prove a NetworkPolicy with allowed and denied flows visible in Hubble.
- Document rollback to the recorded MicroK8s networking state.

### Ordered sections

1. Explain why a CNI migration is cluster-critical
2. Capture CNI, CIDR, route, iptables/nftables, kube-proxy, and MetalLB state
3. Export all existing NetworkPolicies and networking manifests
4. Verify kernel and Cilium compatibility
5. Choose the kube-proxy and LoadBalancer ownership models
6. Take backups and establish console recovery access
7. Create pinned Cilium values
8. Render and validate the chart against the recorded cluster settings
9. Perform the tested CNI transition during the maintenance window
10. Wait for Cilium agents and operator readiness
11. Run Cilium's connectivity health checks
12. Validate CoreDNS and service routing
13. Validate Traefik and `192.168.4.240`
14. Validate every existing private HTTPS endpoint
15. Enable Hubble Relay and UI
16. Configure private HTTPS access for Hubble UI
17. Enable Prometheus metrics and import validated dashboards
18. Apply a namespaced policy to a disposable test workload
19. Observe permitted and denied flows in Hubble
20. Restart test workloads and the node
21. Record the tested rollback procedure and final ownership model

### Required safety gates

- Do not remove the existing CNI until the tested transition procedure reaches that step.
- Do not enable full kube-proxy replacement by assumption.
- Do not let Cilium and MetalLB advertise overlapping address pools.
- Do not perform the migration over a remote-only connection without console recovery.
- Stop and roll back if CoreDNS, the Kubernetes API, or existing ingress validation fails.

### Completion checkpoints

- Cilium reports healthy agents and operator state.
- CoreDNS, ClusterIP services, pod networking, and internet egress work.
- Traefik retains the intended LAN address.
- Argo CD, Gitea, Grafana, and other HTTPS services remain reachable.
- Hubble shows live flows and policy verdicts.
- Prometheus collects the enabled Cilium and Hubble metrics.
- Node and workload restarts preserve networking.
- A tested recovery procedure exists for the former networking configuration.

## Tutorial 12: Kyverno policy management — planned

Proposed filename:

```text
Kyverno-tutorial.md
```

### Purpose

Install Kyverno through Argo CD and introduce Kubernetes policy safely: observe existing violations in Audit mode, add exceptions where justified, then enforce a small validated baseline without breaking platform controllers.

### Prerequisites and decisions

- Complete tutorials 1–11 so policies are designed against the final storage and networking resources.
- Inventory privileged pods, host mounts, host networking, image registries, and namespaces used by platform components.
- Select and pin a Kyverno chart/version compatible with the cluster.
- Define policy ownership, exception review, and emergency rollback rules.
- Start new policies in Audit mode.

### Scope

- Install Kyverno controllers and CRDs through Argo CD.
- Enable metrics and policy reports.
- Separate the Kyverno installation from Git-managed policy definitions.
- Add a small baseline covering labels, resource requests/limits, approved registries, and risky pod settings.
- Exclude or explicitly handle system namespaces and known platform requirements.
- Prove validation, mutation, background scanning, exceptions, and reporting.
- Promote only a low-risk, clean policy from Audit to Enforce.

### Ordered sections

1. Inventory current workloads and policy-sensitive settings
2. Define namespace scope and exception governance
3. Select and pin the tested Kyverno release
4. Create Kyverno values and the Argo CD Application
5. Render CRDs, RBAC, webhooks, and controllers
6. Commit, sync, and validate controller readiness
7. Connect metrics and dashboards to Prometheus/Grafana
8. Create a separate Kyverno policies Application
9. Add the first validation policy in Audit mode
10. Review PolicyReports for existing violations
11. Correct workloads or create narrow, documented exceptions
12. Add and prove one mutation policy on a test namespace
13. Add a disposable workload that should be admitted
14. Add a disposable workload that should be reported or denied
15. Promote one clean policy from Audit to Enforce
16. Validate Argo CD, Helm hooks, and platform controllers after enforcement
17. Test policy self-heal and rollback

### Initial policy candidates

- require the standard application ownership labels;
- require CPU and memory requests on selected application namespaces;
- reject privileged containers outside explicitly exempt platform namespaces;
- reject hostPath use outside the approved storage and platform components;
- restrict image registries for application namespaces;
- add safe default labels or security settings through mutation only where deterministic.

### Completion checkpoints

- Kyverno's controllers and webhooks are healthy.
- PolicyReports are visible and metrics are collected.
- Audit policies identify real state without blocking reconciliation.
- Exceptions are narrow, named, justified, and Git-managed.
- Both admission and background scanning are demonstrated.
- One validated baseline policy is enforced successfully.
- Existing Argo CD-managed platform applications remain healthy.

## Tutorial 13: Argo Rollouts progressive delivery — planned

Proposed filename:

```text
Argo-Rollouts-tutorial.md
```

### Purpose

Install Argo Rollouts and demonstrate a GitOps-owned canary release with observable steps, manual promotion, automated analysis, abort, and rollback.

### Prerequisites and decisions

- Complete tutorials 1–12.
- Choose a disposable sample application before converting a real workload.
- Use the existing Traefik ingress path and Prometheus metrics.
- Select and pin compatible controller, chart, CLI, and dashboard versions.
- Decide whether the first validated strategy is weighted canary or blue-green; document the tested Traefik integration.

### Scope

- Install the Rollouts controller and CRDs through Argo CD.
- Install the matching CLI used for observing and controlling releases.
- Optionally expose the dashboard through private HTTPS.
- Deploy stable and canary Services plus a sample Rollout.
- Define explicit canary weights and pause steps.
- Add Prometheus-backed AnalysisTemplates.
- Demonstrate promote, pause, resume, abort, rollback, and self-heal.
- Explain the ownership boundary between Git desired state and operator actions.

### Ordered sections

1. Select the sample application and measurable success signal
2. Select and pin the tested Rollouts release and CLI
3. Create values and the Argo CD Application
4. Render CRDs, RBAC, controller, and metrics resources
5. Commit, sync, and validate the controller
6. Configure ServiceMonitor and dashboards
7. Create stable and canary Services
8. Convert the sample Deployment manifest to a Rollout
9. Define canary steps and pauses
10. Configure the tested Traefik traffic-routing method
11. Deploy the initial stable revision
12. Commit a new image tag and observe the canary
13. Validate replica counts, traffic, metrics, and events at each step
14. Manually promote a healthy canary
15. Create a Prometheus AnalysisTemplate
16. Prove a successful automated analysis
17. Trigger a controlled failed analysis and verify abort
18. Restore the stable revision through Git
19. Validate restart and Argo CD self-heal behavior
20. Document the pattern for future application adoption

### Completion checkpoints

- The controller and CRDs are healthy and GitOps-managed.
- A stable sample revision serves traffic.
- A new revision pauses at the declared canary steps.
- Promotion completes successfully.
- Prometheus analysis succeeds for a healthy version.
- A controlled bad version aborts without replacing the stable version.
- Git rollback restores the desired revision.
- Traefik, Rollouts, and Argo CD ownership responsibilities are documented.

## Tutorial 14: Renovate dependency automation — planned

Proposed filename:

```text
Renovate-tutorial.md
```

### Purpose

Run a self-hosted Renovate service against Gitea so pinned Helm charts, container images, and selected GitOps dependencies receive controlled update pull requests without placing credentials in Git.

### Prerequisites and decisions

- Complete tutorials 1–13.
- Create a dedicated, least-privilege Gitea bot identity and token.
- Store the token in OpenBao and deliver it with ESO.
- Select and pin the Renovate image/chart version.
- Decide the repository discovery scope, schedule, grouping, and initial no-automerge policy.
- Decide whether Renovate runs as a CronJob or another tested execution model.

### Scope

- Create the bot account and repository permissions.
- Store and sync its credential through OpenBao and ESO.
- Deploy Renovate through Argo CD.
- Configure Gitea platform access and explicit repository discovery.
- Add a repository-owned `renovate.json` configuration.
- Enable managers for Helm, Argo CD YAML, container images, and other formats actually used by the repository.
- Start with dry-run/log validation, then allow onboarding and update pull requests.
- Apply schedules, grouping, dependency dashboard, and rate limits.
- Validate one real patch/minor update from discovery through merged GitOps reconciliation.

### Ordered sections

1. Inventory dependency formats in the GitOps repository
2. Define update, grouping, schedule, and merge policy
3. Create the Gitea Renovate bot with minimum permissions
4. Store its token in OpenBao
5. Create the ESO ExternalSecret for Renovate
6. Select and pin the tested Renovate release
7. Create Renovate configuration and the Argo CD Application
8. Render and validate the workload without secret values
9. Run the tested dry-run mode against one repository
10. Review discovered managers and dependencies
11. Add `renovate.json` to the repository
12. Enable onboarding and the dependency dashboard
13. Generate the first controlled update pull request
14. Review version, changelog, rendered manifests, and checks
15. Merge the update and watch Argo CD reconcile it
16. Validate schedules, rate limits, grouping, and duplicate prevention
17. Rotate the bot token through OpenBao and ESO
18. Validate restart and GitOps self-heal

### Completion checkpoints

- Renovate authenticates as the dedicated Gitea bot.
- The credential comes from OpenBao through ESO and never appears in Git.
- Dry-run discovery matches the intended repositories and dependency types.
- The dependency dashboard and update pull request are created.
- An approved update passes repository validation and Argo CD reconciliation.
- Automerge remains disabled until a separate policy is explicitly validated.
- Token rotation succeeds without redeploying secret material from Git.

## Tutorial 15: Atlantis Terraform pull-request automation — planned

Proposed filename:

```text
Atlantis-tutorial.md
```

### Purpose

Deploy Atlantis as a private GitOps-managed service, connect it to the validated version-control workflow, and prove a Terraform pull-request plan/apply cycle with locked state, policy boundaries, auditability, and secrets supplied by OpenBao.

### Prerequisites and decisions

- Complete tutorials 1–14.
- Verify the selected Atlantis release's support for the homelab's Gitea version and required webhook events before implementation.
- Choose a disposable Terraform repository and non-production test target.
- Use a remote state backend with locking; never validate against important infrastructure first.
- Define who may plan, approve, apply, unlock, and administer Atlantis.
- Decide whether the webhook endpoint is LAN-only, reached through a controlled tunnel, or exposed through another authenticated path.
- Store VCS, webhook, cloud, and backend credentials in OpenBao and sync them with ESO.

### Scope

- Validate VCS compatibility and webhook payloads before deployment.
- Create a dedicated Atlantis bot identity and least-privilege credentials.
- Install Atlantis through Argo CD with persistent data where the tested chart requires it.
- Expose its UI/webhook through Traefik and cert-manager.
- Configure repository allowlists and server-side workflow rules.
- Integrate Terraform formatting, validation, plan, approval, apply, and locking.
- Prevent unapproved workflow overrides and secret exposure in logs/plans.
- Add metrics and alerts.
- Validate webhook delivery, pull-request comments, plan output, locking, apply, drift, and recovery.

### Ordered sections

1. Confirm tested Atlantis and Gitea compatibility
2. Select the disposable Terraform repository and remote backend
3. Define plan/apply authorization and branch protections
4. Create the Atlantis bot and webhook secret
5. Store all credentials in OpenBao
6. Create ESO resources for the Atlantis namespace
7. Select and pin the tested Atlantis chart and application versions
8. Create values and the Argo CD Application
9. Configure private HTTPS and the webhook route
10. Render and inspect RBAC, persistence, environment, and secret references
11. Commit, sync, and validate the Atlantis service
12. Configure the Gitea webhook and verify delivery
13. Add the repository allowlist and server-side workflow
14. Open a Terraform pull request and receive a plan comment
15. Verify state locking with a competing pull request
16. Complete review and execute an authorized apply
17. Confirm remote state and target infrastructure
18. Merge and verify lock cleanup
19. Test a rejected unauthorized apply and disallowed workflow override
20. Validate metrics, restart behavior, token rotation, and self-heal
21. Document recovery for stale locks, unavailable VCS, and failed applies

### Required safety gates

- Do not deploy until Gitea compatibility and webhook delivery are proven with the selected versions.
- Do not use local Terraform state.
- Do not give the Atlantis pod unrestricted cluster or cloud credentials.
- Do not permit arbitrary repository-defined workflows before server-side restrictions are validated.
- Do not test the first apply against production or irreplaceable homelab infrastructure.
- Keep plan output free of sensitive values and rotate any credential exposed during testing.

### Completion checkpoints

- Atlantis is healthy, privately reachable, and monitored.
- Gitea webhook delivery and signature validation succeed.
- Credentials are supplied through OpenBao and ESO.
- A pull request receives the expected Terraform plan.
- Concurrent work demonstrates state and project locking.
- Only an authorized, approved apply succeeds.
- Remote state records the applied result and the lock clears.
- Restart, credential rotation, and Argo CD self-heal tests pass.

## Cross-tutorial dependency notes

Some settings are introduced in one tutorial and finalized in another:

- Argo CD and Gitea are initially reachable over LAN HTTP, then migrated to HTTPS in the cert-manager tutorial.
- Grafana is installed by kube-prometheus-stack, then receives its Loki datasource in the Loki tutorial and HTTPS ingress in the cert-manager tutorial.
- The Cloudflare API token is initially a manually bootstrapped Kubernetes Secret in the cert-manager tutorial, then moved to OpenBao in the OpenBao tutorial.
- ESO is installed independently first; its OpenBao backend and real ExternalSecrets are configured in the OpenBao tutorial.
- All workload tutorials depend on the root App-of-Apps structure from the Argo CD tutorial.
- Longhorn is planned before later platform additions because storage migration is disruptive and requires verified backups and rollback copies.
- Cilium follows Longhorn and requires a dedicated maintenance window because it changes cluster networking beneath every later controller.
- Kyverno follows the infrastructure migrations so its initial Audit reports reflect the final storage and networking resource shapes.
- Argo Rollouts depends on Traefik, Prometheus, Argo CD, and the Kyverno exception/policy model.
- Renovate depends on Gitea, OpenBao, ESO, and the final GitOps file layout.
- Atlantis is last because it depends on Gitea webhooks, private HTTPS, OpenBao/ESO credentials, persistent storage, monitoring, and an established pull-request policy.

These transitions should be clearly cross-referenced so readers follow the series in order without duplicating or prematurely applying later-state configuration.

## Topics still excluded

The six planned extensions above remain plans until implemented and validated. The following topics are still outside this tutorial series:

- multi-node MicroK8s
- a custom application deployment

They can be added later after their implementation and end-to-end validation are complete.

## Suggested production order for writing

Write the tutorials in this order so later documents can link to stable prerequisites:

1. `MicroK8s-foundation-tutorial.md`
2. `ArgoCD-gitops-bootstrap-tutorial.md`
3. `Gitea-tutorial.md`
4. `Kube-prometheus-stack-tutorial.md`
5. `Loki-tutorial.md`
6. `Grafana-Alloy-tutorial.md`
7. `Cert-manager-Cloudflare-tutorial.md`
8. `External-Secrets-Operator-tutorial.md`
9. Review cross-links in `Openbao-tutorial.md`
10. Implement, validate, and write `Longhorn-tutorial.md`
11. Implement, validate, and write `Cilium-Hubble-tutorial.md`
12. Implement, validate, and write `Kyverno-tutorial.md`
13. Implement, validate, and write `Argo-Rollouts-tutorial.md`
14. Implement, validate, and write `Renovate-tutorial.md`
15. Implement, validate, and write `Atlantis-tutorial.md`

Each tutorial should be written from the recorded final working configuration after its implementation succeeds. Conversation history may supply supporting context, but the live Git manifests and validated cluster state are authoritative. Check every finished tutorial for:

- correct dependency order
- exact filenames and namespaces
- pinned versions
- no secret material
- no abandoned alternatives
- balanced Markdown code fences
- commands that can be copied in sequence
- a functional end-state validation
