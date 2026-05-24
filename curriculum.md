# Jenkins Training Curriculum

The curriculum is split into two parts. Each module is a standalone `.md` file inside its part's directory.

- **Part 1** — Running Jenkins on a standalone server (controller + standalone agents), all features supported in that model, ending with everything-as-code.
- **Part 2** — Hosting Jenkins on Kubernetes, end-to-end.

---

## Part 1 — Jenkins on a Standalone Server

`part-1-standalone/`

### Section A: Foundations
- [01 — Introduction & Fundamentals](part-1-standalone/01-introduction-and-fundamentals.md)
- [02 — Standalone Installation & Setup](part-1-standalone/02-standalone-installation-and-setup.md)
- [03 — Jenkins UI & Job Basics](part-1-standalone/03-jenkins-ui-and-job-basics.md)

### Section B: Core Capabilities on Standalone
- [04 — Source Code Management](part-1-standalone/04-source-code-management.md)
- [05 — Standalone Agents (Build Nodes)](part-1-standalone/05-standalone-agents.md)
- [06 — Plugins & Integrations](part-1-standalone/06-plugins-and-integrations.md)
- [07 — Security & User Management](part-1-standalone/07-security-and-user-management.md)
- [08 — Operations on a Standalone Server](part-1-standalone/08-operations.md)
- [09 — Deployment & Release Workflows](part-1-standalone/09-deployment-and-release.md)

### Section C: Everything-as-Code
- [10 — Pipeline as Code](part-1-standalone/10-pipeline-as-code.md)
- [11 — Advanced Pipelines](part-1-standalone/11-advanced-pipelines.md)
- [12 — Shared Libraries](part-1-standalone/12-shared-libraries.md)
- [13 — Configuration as Code (JCasC)](part-1-standalone/13-configuration-as-code.md)
- [14 — Jobs & Plugins as Code](part-1-standalone/14-jobs-and-plugins-as-code.md)
- [15 — Part 1 Capstone](part-1-standalone/15-capstone.md)

---

## Part 2 — Jenkins on Kubernetes (End-to-End)

`part-2-kubernetes/`

### Section A: Kubernetes Foundations for Jenkins
- [01 — Kubernetes Refresher (Jenkins-Relevant)](part-2-kubernetes/01-kubernetes-refresher.md)
- [02 — Architecture on Kubernetes](part-2-kubernetes/02-architecture-on-kubernetes.md)

### Section B: Installing & Configuring Jenkins on Kubernetes
- [03 — Installation](part-2-kubernetes/03-installation.md)
- [04 — Kubernetes Plugin & Dynamic Agents](part-2-kubernetes/04-kubernetes-plugin-and-dynamic-agents.md)
- [05 — Identity, Secrets & Security](part-2-kubernetes/05-identity-secrets-and-security.md)
- [06 — Configuration as Code on Kubernetes](part-2-kubernetes/06-configuration-as-code-on-kubernetes.md)

### Section C: Building & Deploying with Jenkins on Kubernetes
- [07 — Building Container Images in K8s Agents](part-2-kubernetes/07-building-container-images.md)
- [08 — Deploying to Kubernetes from Jenkins](part-2-kubernetes/08-deploying-to-kubernetes.md)
- [09 — Testing & Quality Gates in K8s Pipelines](part-2-kubernetes/09-testing-and-quality-gates.md)

### Section D: Operating Jenkins on Kubernetes
- [10 — Scaling & Performance](part-2-kubernetes/10-scaling-and-performance.md)
- [11 — Observability & Troubleshooting](part-2-kubernetes/11-observability-and-troubleshooting.md)
- [12 — Backup, DR & Upgrades](part-2-kubernetes/12-backup-dr-and-upgrades.md)
- [13 — Multi-Team & Enterprise Patterns](part-2-kubernetes/13-multi-team-and-enterprise-patterns.md)
- [14 — Part 2 Capstone](part-2-kubernetes/14-capstone.md)

---

## Suggested Schedule
- **Part 1:** 4–5 weeks (~30–40 hours), ~60% hands-on
- **Part 2:** 3–4 weeks (~25–35 hours), ~70% hands-on

## Prerequisites
- **Part 1:** Linux basics, Git, basic scripting
- **Part 2:** Part 1 + Kubernetes fundamentals, Docker, Helm basics
