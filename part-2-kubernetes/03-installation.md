# Module 03 — Installation

## Learning Objectives
- Install Jenkins on Kubernetes using the official Helm chart.
- Configure a persistent `JENKINS_HOME` PVC.
- Expose the UI through Ingress with TLS.
- Set resource requests/limits and probes.
- Upgrade and roll back the chart safely.

## 1. Prerequisites

- A Kubernetes cluster (kind/k3d/minikube for learning; managed cluster for real).
- `kubectl` configured to point at it.
- `helm` v3.x installed.
- An Ingress controller installed (e.g. `ingress-nginx`).
- A StorageClass that provisions ReadWriteOnce volumes (most clusters have one by default).
- A working cert-issuing mechanism if you want automatic TLS (e.g. cert-manager + Let's Encrypt).

Quick sanity check:
```bash
kubectl get nodes
kubectl get storageclass
kubectl get pods -n ingress-nginx       # or your ingress controller's namespace
```

## 2. The Official Helm Chart

The chart is maintained at `https://github.com/jenkinsci/helm-charts`.

Add the repo:
```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

Inspect:
```bash
helm search repo jenkins/jenkins
helm show values jenkins/jenkins > default-values.yaml
```

Reading `default-values.yaml` is the best way to learn what's configurable.

## 3. Namespace and Initial Secret

```bash
kubectl create namespace jenkins

# Choose your own admin password
kubectl -n jenkins create secret generic jenkins-admin \
  --from-literal=jenkins-admin-user=admin \
  --from-literal=jenkins-admin-password='ChangeMe-2026!'
```

The chart can also auto-generate the password; using a pre-created Secret keeps the credential out of the values file.

## 4. A First `values.yaml`

```yaml
controller:
  image: "jenkins/jenkins"
  tag: "lts-jdk17"

  adminSecret: true
  admin:
    existingSecret: jenkins-admin
    userKey: jenkins-admin-user
    passwordKey: jenkins-admin-password

  resources:
    requests: { cpu: "1",  memory: "2Gi" }
    limits:   { cpu: "2",  memory: "4Gi" }

  jenkinsUrl: "https://jenkins.example.com"
  jenkinsUrlProtocol: "https"

  installPlugins:
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - configuration-as-code:latest
    - job-dsl:latest
    - matrix-auth:latest
    - role-strategy:latest
    - prometheus:latest
    - slack:latest

  ingress:
    enabled: true
    hostName: jenkins.example.com
    ingressClassName: nginx
    tls:
      - hosts: [ jenkins.example.com ]
        secretName: jenkins-tls
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/proxy-body-size: "50m"

  probes:
    startupProbe:
      failureThreshold: 60       # allow up to 10 min to start with many plugins
      periodSeconds: 10

persistence:
  enabled: true
  size: 50Gi
  storageClass: "gp3"            # cluster-specific

agent:
  enabled: false                 # we'll wire dynamic agents up in Module 04

serviceAccount:
  create: true
  name: jenkins
```

Save as `jenkins-values.yaml`.

## 5. Install

```bash
helm upgrade --install jenkins jenkins/jenkins \
  -n jenkins -f jenkins-values.yaml \
  --version <pinned-chart-version>
```

Pin the chart version (`helm search repo jenkins/jenkins -l`); don't track `latest`.

Watch the rollout:
```bash
kubectl -n jenkins get pods -w
kubectl -n jenkins describe pod jenkins-0
kubectl -n jenkins logs jenkins-0 -c jenkins -f
```

The Pod will take several minutes the first time as it installs plugins.

## 6. Access the UI

Once the Ingress is ready and DNS resolves:
- Browse `https://jenkins.example.com`.
- Log in with the credentials you placed in the `jenkins-admin` Secret.

If you're sandboxing without DNS/Ingress:
```bash
kubectl -n jenkins port-forward svc/jenkins 8080:8080
# browse http://localhost:8080
```

## 7. What the Chart Created

```bash
kubectl -n jenkins get all
kubectl -n jenkins get pvc
kubectl -n jenkins get serviceaccount,role,rolebinding
kubectl -n jenkins get ingress
```

You should see:
- `pod/jenkins-0` (StatefulSet).
- `pvc/jenkins` (or similar) — the controller's `JENKINS_HOME`.
- `svc/jenkins` (ClusterIP, 8080) and `svc/jenkins-agent` (ClusterIP, 50000) — the latter for legacy JNLP.
- An Ingress for the UI.
- A ServiceAccount, Role, and RoleBinding granting Pod management in the namespace.

## 8. JCasC via the Chart

The chart has first-class JCasC support. Anything under `controller.JCasC.configScripts` is mounted as a ConfigMap and loaded on startup.

```yaml
controller:
  JCasC:
    defaultConfig: true
    configScripts:
      welcome: |
        jenkins:
          systemMessage: |
            Managed by Helm + JCasC. Edits via the UI will be overwritten.
      auth: |
        jenkins:
          authorizationStrategy:
            roleBased:
              roles:
                global:
                  - name: admin
                    permissions: ["Overall/Administer"]
                    assignments: ["admin"]
                  - name: developer
                    permissions: ["Overall/Read","Job/Build","Job/Read","Job/Workspace"]
                    assignments: ["authenticated"]
```

Apply with `helm upgrade`. The Pod hot-reloads the JCasC config (no restart needed) for most changes.

## 9. Upgrades

### Routine upgrade
```bash
# get latest chart info
helm repo update
helm search repo jenkins/jenkins -l | head

# upgrade to a specific chart version
helm upgrade jenkins jenkins/jenkins \
  -n jenkins -f jenkins-values.yaml \
  --version 5.x.x
```

The chart will roll the StatefulSet — there's brief downtime as the new Pod replaces the old one. Watch the logs.

### Rollback
```bash
helm history jenkins -n jenkins
helm rollback jenkins <revision> -n jenkins
```

The PVC keeps your `JENKINS_HOME` across upgrades and rollbacks.

### Strategy
- Test chart upgrades in a non-prod cluster first.
- Bump core image (`controller.tag`) and plugin versions in deliberate steps, not all at once.
- Always have a recent PVC snapshot before a major change.

## 10. Resource Tuning

Watch real usage with `kubectl top pod` and Prometheus, then revise:
- If the controller hits its memory limit, raise it.
- If CPU is consistently near the request, raise it (or reduce concurrent builds).
- Don't set the limit equal to the request unless you know what you're doing — bursts kill builds.

## 11. Common Issues

### Pod stuck Pending
- PVC unbound → check `kubectl describe pvc`. StorageClass missing or wrong access mode.
- No node has resources → check `kubectl describe pod` for scheduler messages.

### Pod CrashLoopBackOff
- Check `kubectl logs jenkins-0 -c jenkins`.
- Plugin install failure or JVM OOM are common first-boot issues. Raise memory; remove a problem plugin.

### Ingress 502 / timeout
- Ingress controller can't reach the Service — usually a Network Policy or namespace selector.
- Probes not passing yet — give Jenkins more startup time.

### TLS not issued
- cert-manager Order/Challenge stuck → check `kubectl describe certificate jenkins-tls`.
- DNS for the hostname not pointing at the Ingress load balancer.

## 12. Hands-On Exercise

1. Install Jenkins on your cluster using the Helm chart.
2. Persist via a 20 GiB PVC on your cluster's default StorageClass.
3. Expose via Ingress with TLS (cert-manager + Let's Encrypt, or a self-signed cert).
4. Set JCasC to display a system message via `configScripts`.
5. Upgrade to a slightly newer chart version; then roll back; confirm the controller comes back up unchanged.
6. Take a snapshot of the PVC.

## 13. Knowledge Check
1. Why pin the Helm chart version?
2. What is the `configScripts` section of the chart's values for?
3. Where do plugins live across a Pod restart? Why?
4. What happens to the PVC during a Helm rollback?
5. How do you give Jenkins more time to start when it has many plugins?

## What's Next
**Module 04** wires up dynamic, ephemeral agents using the Kubernetes plugin.
