# Module 13 — Multi-Team & Enterprise Patterns

## Learning Objectives
- Run one Jenkins for many teams without conflict.
- Use folders, RBAC, namespaces, and quotas to isolate workloads.
- Provide shared libraries and pipeline templates across teams.
- Enforce platform policies (security, cost, quality) globally.
- Understand the managed-controllers pattern.

## 1. The Multi-Team Tension

Two paths:
- **One Jenkins for everyone.** Maximum sharing, minimum operational overhead. Maximum blast radius.
- **One Jenkins per team.** Isolation by default. Many controllers to upgrade and monitor.

In practice most platforms start with **one shared controller**, isolate teams with folders/RBAC/namespaces, and only spin up additional controllers for genuine reasons (regulatory, severe plugin conflict).

## 2. Folders for Logical Isolation

Folders give every team a private workspace inside one Jenkins:

```
/
├── platform/
│   ├── seed
│   └── infra-pipelines/...
├── payments/
│   ├── payments-api (multibranch)
│   ├── payments-worker (multibranch)
│   └── payments-deploy
├── orders/
│   └── ...
└── shared/
    └── library-smoke
```

Each folder:
- Has its own credentials store (folder-scoped) — team A can't read team B's secrets.
- Can pin a different default Shared Library version.
- Has its own permissions, granting the team admin rights inside the folder.

## 3. Role-Based Authorization for Folders

With the **Role-Based Strategy** plugin (Module 07, Part 1):

```yaml
authorizationStrategy:
  roleBased:
    roles:
      global:
        - name: admin
          permissions: ["Overall/Administer"]
          assignments: ["platform-admins"]
        - name: read-everything
          permissions: ["Overall/Read","Job/Read"]
          assignments: ["authenticated"]
      project:
        - name: payments-admin
          pattern: "payments/.*"
          permissions: ["Job/Build","Job/Configure","Job/Create","Job/Delete",
                        "Credentials/Create","Credentials/Update"]
          assignments: ["payments-team"]
        - name: orders-admin
          pattern: "orders/.*"
          permissions: ["Job/Build","Job/Configure","Job/Create","Job/Delete",
                        "Credentials/Create","Credentials/Update"]
          assignments: ["orders-team"]
```

The platform team manages global config. Each product team manages their own folder.

## 4. Namespaces and Quotas for Kubernetes Isolation

Map each team to its own build namespace:
- `jenkins-builds-payments`
- `jenkins-builds-orders`
- ...

In the Kubernetes cloud config, define pod templates per namespace via labels:
```yaml
clouds:
  - kubernetes:
      templates:
        - name: payments-base
          label: team-payments
          namespace: jenkins-builds-payments
        - name: orders-base
          label: team-orders
          namespace: jenkins-builds-orders
```

Pipelines pick the right namespace by label:
```groovy
agent { label 'team-payments' }
```

### ResourceQuota
Cap each namespace so one team can't starve others:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: build-quota, namespace: jenkins-builds-payments }
spec:
  hard:
    requests.cpu: "32"
    requests.memory: "64Gi"
    limits.cpu: "64"
    limits.memory: "128Gi"
    pods: "30"
```

### NetworkPolicy
Builds in `payments` namespace cannot reach builds in `orders`. Done by default-deny + targeted allows (Module 05).

## 5. Shared Libraries Serving Many Teams

The platform team owns one or two libraries everyone consumes:
- `acme-ci` — core pipeline steps (`javaService`, `nodeService`, `deployHelm`, etc.).
- `acme-security` — security checks every pipeline must include.

Tagging:
- Library tags like `v1.x` for stable, `v2.x` for next major.
- Teams pin: `@Library('acme-ci@v1')`.
- Major changes announced ahead; old versions kept available indefinitely.

Configure them in JCasC at folder scope so different folders can pin different majors:
```yaml
globalLibraries:
  libraries:
    - name: acme-ci
      defaultVersion: v1
      implicit: true
```

### Library smoke
A separate `library-smoke` repo and pipeline exercises every public step against a dummy service. Tag the library only after smoke passes. Saves all consumers.

## 6. Pipeline Templates

A platform pipeline template (from Module 12, Part 1) lets a team's `Jenkinsfile` be three lines:
```groovy
@Library('acme-ci@v1') _
javaService(appName: 'payments-api')
```

This is where the platform team enforces:
- Mandatory steps (security scan, SBOM, license check).
- Cache configuration.
- Notification settings.
- Required quality gates.

### Extension points
Provide ways to customize without forking:
```groovy
javaService(
  appName: 'payments-api',
  extraVerify: { sh './custom-check.sh' },     // closure injected into the verify stage
  helmValuesFile: 'chart/values-payments.yaml'
)
```

## 7. Enforcing Policies Globally

Policies you'll want to enforce:

### Build-time
- Every container image scanned.
- Every chart validated with Conftest.
- License check on dependencies (FOSSA, Allay, scancode).
- Critical CVE = build fails.

### Pipeline-time
- All pipelines must inherit the template (or pass a "compliance" review).
- Force `agent none` at top level so no work runs on the controller.
- Force `timeout` on every stage.

### Cluster-time
- Admission policies (Gatekeeper, Kyverno) deny images without signatures or from outside-approved registries.
- ResourceQuotas prevent runaway namespaces.
- Network policies enforced.

The pipeline template enforces "soft" rules; admission controllers enforce "hard" rules. Belt and braces.

## 8. Cost Allocation

Labels are the lever:
- Agent Pods labeled `team=payments`.
- Build namespaces labeled `team=payments`.
- PR preview namespaces labeled `team=payments`.

Your cloud cost tool then reports CI spend by team.

Make this visible in a shared Grafana dashboard. Cost-aware teams optimize themselves.

## 9. When to Add Another Controller

Signals it's time:
- Plugin requirements conflict between teams.
- A single team's load is dominating others and isolation alone isn't enough.
- A team needs different SLAs (different upgrade cadence, different uptime guarantees).
- Regulatory boundary requires separate audit trails.

Pattern: spin up another controller (same Helm chart, different release name and namespace). The platform team operates both. Shared libraries and the same JCasC layout reduce duplication.

## 10. The Managed Controllers Pattern

For very large orgs: one **operations controller** runs platform admin jobs and provisions per-team **execution controllers**, each isolated. This is the model behind some commercial offerings.

The DIY version:
- The operations controller has Job DSL that creates/destroys execution controllers.
- Execution controllers are Helm releases configured purely via Git.
- Teams admin their own execution controllers via folders/RBAC.

Big tradeoff: more controllers = more upgrade and ops cost. Only do this when shared single-controller approaches genuinely break.

## 11. Onboarding a New Team

A documented runbook:
1. Create folder `<team>/`.
2. Create build namespace `jenkins-builds-<team>` with quota and network policy.
3. Define pod template label `team-<team>` pointing at that namespace.
4. Add a role in JCasC granting them admin in their folder.
5. Add team's GitHub org/repo scan to a multibranch organization folder.
6. Deliver a "starter `Jenkinsfile`" using `javaService` (or equivalent).

Goal: a new team is productive in under an hour with one Git PR.

## 12. Platform Engineering Mindset

- The platform team is the *product* team for the rest of the org. Treat your users as customers.
- Provide great defaults. Make the right thing the easy thing.
- Track SLIs: build success rate, queue wait time, deploy success rate, mean time to recovery.
- Run office hours / shared chat where teams ask questions.
- Deprecate noisily and slowly: announce N months ahead, leave the old API working for one major version.

## 13. Hands-On Exercise

1. Add two folders to your Jenkins: `payments/` and `orders/`.
2. Create two build namespaces with quotas; bind to pod templates with labels `team-payments` and `team-orders`.
3. Use JCasC to define role-based permissions per folder.
4. Create test users in each team's role. Confirm `payments-dev` cannot configure jobs in `orders/`.
5. Write a starter `Jenkinsfile` that targets the right namespace based on a label.
6. Verify ResourceQuota actually limits one team's concurrent builds.

## 14. Knowledge Check
1. What does folder-scoped credentials give you?
2. How do you ensure one team's builds can't reach another's services?
3. Why have a separate `library-smoke` pipeline?
4. What's the difference between a pipeline template enforcing policy and an admission controller enforcing policy?
5. When does adding a second Jenkins controller make sense?

## What's Next
**Module 14** is the Part 2 capstone — a full Jenkins-on-Kubernetes platform end-to-end.
