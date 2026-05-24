# Module 14 — Part 2 Capstone

## Goal
Deliver an end-to-end Jenkins-on-Kubernetes platform: code-managed, observable, secure, and recoverable.

## Scenario
Acme Inc. is moving CI/CD from a legacy standalone Jenkins onto Kubernetes. You're the platform lead. You have:
- A Kubernetes cluster (managed or self-hosted).
- An object store for backups (S3/GCS/Azure Blob).
- Three application teams (Payments, Orders, Shipping), each with a small microservice in GitHub.
- An expectation that the new platform is reproducible from Git and operable by a small team.

## Deliverables Checklist

### 1. Cluster prep
- [ ] Namespaces created: `jenkins`, `jenkins-builds-payments`, `jenkins-builds-orders`, `jenkins-builds-shipping`.
- [ ] Pod Security Standards `baseline` enforced on build namespaces.
- [ ] Ingress controller installed (e.g. ingress-nginx).
- [ ] cert-manager installed with a Let's Encrypt (or internal CA) ClusterIssuer.
- [ ] StorageClass configured for fast SSD ReadWriteOnce volumes.
- [ ] Cluster autoscaler or Karpenter deployed with a `workload=ci` build node pool.

### 2. Config-as-code repo (`jenkins-config`)
Repo contains:
- [ ] `chart/values.yaml` — base Helm values.
- [ ] `chart/values-prod.yaml` — production overrides.
- [ ] `jcasc/*.yaml` — broken into focused files (auth, clouds, libraries, credentials).
- [ ] `plugins.txt` — pinned plugin versions.
- [ ] `seed/jobs.groovy` — Job DSL for folders, multibranch projects per team.
- [ ] `secrets/*.yaml` — Sealed Secrets / ExternalSecrets (never plaintext).
- [ ] `manifests/` — RBAC, NetworkPolicies, ResourceQuotas for each namespace.
- [ ] `README.md` — deploy and operate.

A pull-request-driven workflow validates and deploys changes.

### 3. Controller installation
- [ ] Jenkins controller deployed via Helm with pinned chart version.
- [ ] `JENKINS_HOME` on a 50 GiB SSD PVC.
- [ ] Ingress `jenkins.example.com` with valid TLS.
- [ ] Probes tuned for ~50-plugin startup.
- [ ] Resource requests/limits set.
- [ ] No work runs on the controller (`numExecutors: 0`).

### 4. Dynamic agents
- [ ] Kubernetes plugin configured for build namespaces.
- [ ] Pod templates per team (`team-payments`, `team-orders`, `team-shipping`).
- [ ] Agents run as non-root with read-only root filesystems where possible.
- [ ] Pre-pulled base images on build nodes via a DaemonSet (or equivalent).
- [ ] Spot/preemptible nodes for general builds; on-demand for `prod-deploy` workloads.

### 5. Security
- [ ] Controller ServiceAccount scoped to a Role in build namespaces (no ClusterRole).
- [ ] Agent ServiceAccounts minimal; deploy ServiceAccounts namespaced per target.
- [ ] Default-deny NetworkPolicies in all build namespaces; explicit allows.
- [ ] etcd encrypted at rest, or all credentials sourced from Vault via ESO.
- [ ] Image pull credentials per-namespace; rotated.
- [ ] Cosign signing key configured; SBOM generation in pipelines.

### 6. JCasC + Shared Libraries
- [ ] Authorization: `admin`, `developer`, `reader` roles globally; team-admin roles scoped to folders.
- [ ] Each folder has a folder-scoped credentials store.
- [ ] Global library `acme-ci` registered, default version `v1`.
- [ ] Library repo contains `vars/javaService.groovy` (template), `vars/buildContainer.groovy` (Kaniko), `vars/deployHelm.groovy`, `vars/notifySlack.groovy`.
- [ ] `library-smoke` pipeline exercises every public step; tags are cut only after it's green.

### 7. Application pipelines
For each of Payments, Orders, Shipping:
- [ ] A one-page `Jenkinsfile` calling `javaService(appName: '...')`.
- [ ] Multibranch project auto-created by Job DSL.
- [ ] Webhook from GitHub firing builds in seconds.
- [ ] PR builds create an ephemeral preview namespace `<app>-pr-<num>` and tear it down on close.
- [ ] PR preview URL posted back to the GitHub PR.

### 8. Build → image → deploy
For each service:
- [ ] Maven build + JUnit + coverage.
- [ ] Container image built with Kaniko, pushed to a private registry.
- [ ] Image signed with cosign; SBOM archived.
- [ ] Trivy scan (HIGH/CRITICAL = fail).
- [ ] Helm deploy to `dev` automatically on `main`.
- [ ] Integration test against `dev`.
- [ ] Promote to `staging`.
- [ ] `input` approval gate (`release-managers`) for `prod`.
- [ ] Deploy to `prod` with `helm upgrade --atomic`.
- [ ] Smoke test against prod; auto-rollback if it fails.
- [ ] Git tag on successful prod deploy.

### 9. Observability
- [ ] Prometheus scraping Jenkins (`/prometheus/`) and cluster metrics.
- [ ] Loki / EFK ingesting controller and agent logs.
- [ ] Grafana dashboard: queue, executor utilization, build success rate, deploy success rate, top-5 failing jobs, controller heap.
- [ ] Alerts: queue >50 for 10 min, heap >85% for 5 min, controller down for 2 min, PVC >80% used.
- [ ] OpenTelemetry plugin emitting per-build traces (optional but recommended).

### 10. Backup & DR
- [ ] Velero installed, daily backup of `jenkins`, `jenkins-builds-*`.
- [ ] Backups shipped off-cluster to object storage with versioning.
- [ ] Volume snapshot of the controller PVC at least daily.
- [ ] Documented restore procedure with measured time.
- [ ] Restore drill performed: rebuild controller in a fresh cluster from Git + backups.

### 11. Scaling & cost
- [ ] Build node pool scales to 0 outside business hours.
- [ ] Agent Pod startup < 90s p95.
- [ ] Caching strategy implemented (Maven via Nexus, container layers via Kaniko cache repo).
- [ ] Cost dashboard breaking spend by team.

### 12. Multi-team
- [ ] Folders per team with role-based access.
- [ ] ResourceQuotas per build namespace.
- [ ] NetworkPolicies isolate teams' agent traffic.
- [ ] Onboarding runbook for a new team (target: under 1 hour from PR to first green build).

## Acceptance Tests

Run each on the completed platform.

1. **Reproducibility.** Tear down the Helm release and the PVC. From the `jenkins-config` repo, run the install script. Within 1 hour the controller is back with folders, pipelines, credentials, agents — verified by a smoke pipeline.
2. **PR test.** Open a PR on Payments. Within 3 minutes: namespace `payments-pr-XYZ` exists, preview URL posted in PR, all checks green.
3. **Merge test.** Merge a PR to `main` on Orders. Pipeline auto-deploys to dev, runs integration, deploys to staging, pauses for approval. Approve; prod deploy succeeds; Git tag created.
4. **Bad image test.** Pin a deliberately vulnerable base image. Confirm Trivy fails the build before image push.
5. **Permissions test.** Log in as a Payments dev. Confirm you cannot configure Orders jobs or read Orders credentials.
6. **Quota test.** Trigger 30 builds in Payments. Confirm Payments builds queue once the quota fills, and Orders builds run unaffected.
7. **DR test.** Wipe the entire `jenkins` namespace, including the PVC. Restore from yesterday's Velero backup. Within 1 hour, all builds and credentials are back.
8. **Spot test.** Confirm a normal build survives a spot interruption (Pod gets killed; Jenkins reschedules; build succeeds).
9. **Upgrade test.** Bump the chart by a minor version. `helm upgrade`. Verify smoke pipelines still pass. Then `helm rollback`. Verify everything still works.
10. **Cost test.** Confirm last week's cost dashboard splits spend across the three teams.

## Stretch Goals

- Add a fourth team and validate the onboarding runbook end-to-end.
- Add canary deploys (10% → 50% → 100%) for one service using ingress-nginx canary annotations.
- Implement keyless cosign signing via OIDC instead of a stored key.
- Add OPA/Conftest policy: deny any Helm chart whose manifests omit resource limits.
- Add a second cluster in another region with warm-standby DR (regular restore-from-backup, no live traffic).

## Reflection

Write a one-pager:
- Where is the highest blast-radius risk in this platform? What single change would reduce it most?
- Which automated step would catch the most common production incident on this stack?
- If you handed the platform to a new on-call engineer tomorrow, what would they need to learn first? Is that documented?
- What manual step still exists, and what would it take to eliminate it?

## Closing

You've built a Jenkins platform that:
- Is fully reproducible from Git.
- Runs builds elastically and cheaply.
- Isolates teams while sharing common infrastructure.
- Recovers from disaster within a defined RTO.
- Enforces security and quality at build time and deploy time.

That's the Kubernetes-native target. The same foundation extends to additional teams, additional services, and additional environments without doubling the operating burden.
