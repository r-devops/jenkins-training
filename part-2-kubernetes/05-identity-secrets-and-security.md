# Module 05 — Identity, Secrets & Security

## Learning Objectives
- Configure least-privilege RBAC for controller and agents.
- Apply Pod Security Standards and Network Policies.
- Pull secrets from Kubernetes Secrets, External Secrets Operator, or Vault.
- Use image pull credentials for private registries.
- Run agents as non-root with read-only filesystems.

## 1. Threat Model in One Page

If an attacker compromises a build, what can they reach?
- The controller's `JENKINS_HOME` (via the controller's API).
- Other builds running concurrently (via shared infra, shared caches).
- Your cluster (via the controller's or the agent's ServiceAccount).
- Your cloud account (via mounted cloud credentials).
- Your source code, registries, and deploy targets.

Goal: each layer has minimal blast radius. A compromised agent should be a *nuisance*, not a *breach*.

## 2. RBAC for the Controller

The controller's ServiceAccount needs just enough to manage agent Pods (and read its own ConfigMaps/Secrets).

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: jenkins, namespace: jenkins }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: jenkins-agent-manager, namespace: jenkins-builds }
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec", "pods/log"]
  verbs: ["get","list","watch","create","update","patch","delete"]
- apiGroups: [""]
  resources: ["secrets","configmaps"]
  verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: jenkins-agent-manager, namespace: jenkins-builds }
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
roleRef:
  kind: Role
  name: jenkins-agent-manager
  apiGroup: rbac.authorization.k8s.io
```

If agents live in the same namespace as the controller, you can omit the cross-namespace binding — but I recommend a separate `jenkins-builds` namespace for blast-radius reasons.

### What the controller should NOT have
- Cluster-admin.
- Permission to list Secrets across all namespaces.
- Permission to create/modify RBAC.

## 3. RBAC for Agents

Default: minimal — no privileges. A build that's just compiling code doesn't need an API token.

If a build needs to deploy:
- Create a *dedicated* ServiceAccount in the build namespace, scoped to *just* the deploy targets.
- Use it only for the deploy step. Mount the token only in the relevant container.

Example: a deploy-only ServiceAccount for a target namespace:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: deployer-payments, namespace: payments-prod }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: deployer, namespace: payments-prod }
rules:
- apiGroups: ["apps"]
  resources: ["deployments","statefulsets","daemonsets"]
  verbs: ["get","patch","update"]
- apiGroups: [""]
  resources: ["services","configmaps","secrets"]
  verbs: ["get","patch","update","create"]
```

In your pipeline pod spec, tell the agent Pod to use this ServiceAccount (with `serviceAccountName`) or mount its token as a Secret into the deploy container.

## 4. Pod Security Standards (PSS)

PSS defines three profiles: `privileged`, `baseline`, `restricted`. Enforce them per namespace with labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins-builds
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit:   restricted
    pod-security.kubernetes.io/warn:    restricted
```

Aim for `restricted` if you can; `baseline` is a pragmatic compromise. The `restricted` profile bars privileged containers, hostPath volumes, host networking, etc.

### Agent pods that fit `restricted`
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: maven
      securityContext:
        allowPrivilegeEscalation: false
        capabilities: { drop: ["ALL"] }
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: workspace
          mountPath: /home/jenkins/agent
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: workspace
      emptyDir: {}
    - name: tmp
      emptyDir: {}
```

Note: many tool images don't tolerate `readOnlyRootFilesystem: true` without extra `emptyDir` mounts for caches. Test, adjust.

## 5. Network Policies

Default in most clusters: everything can talk to everything. Tighten:

### Lock down the controller
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: controller-ingress, namespace: jenkins }
spec:
  podSelector: { matchLabels: { app.kubernetes.io/instance: jenkins } }
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { name: ingress-nginx } }
          podSelector: { matchLabels: { app.kubernetes.io/name: ingress-nginx } }
      ports: [ { port: 8080 } ]
    - from:
        - namespaceSelector: { matchLabels: { name: jenkins-builds } }
      ports:
        - { port: 8080 }
        - { port: 50000 }
  policyTypes: ["Ingress"]
```

### Lock down build namespace
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: builds-default-deny, namespace: jenkins-builds }
spec:
  podSelector: {}
  policyTypes: ["Ingress","Egress"]
  egress:
    - to: [ { namespaceSelector: { matchLabels: { name: jenkins } } } ]
      ports: [ { port: 8080 }, { port: 50000 } ]
    - to: [ { namespaceSelector: { matchLabels: { name: kube-system } } } ]
      ports: [ { port: 53, protocol: UDP } ]
    - to: []
      ports: [ { port: 443 }, { port: 80 } ]
```

This allows builds to:
- Reach the controller for the agent connection.
- Resolve DNS via CoreDNS.
- Reach the internet on 80/443 (Git, registries, package managers).

Lock down further if your security model requires going through a proxy.

## 6. Secrets

### Kubernetes Secrets
Fine for low-sensitivity items if etcd is encrypted at rest. Otherwise treat them as base64-encoded plaintext.

Mounting a Secret into the controller (via Helm values):
```yaml
controller:
  additionalExistingSecrets:
    - name: jenkins-credentials-bundle
      keyName: jenkins-credentials-bundle
```

For pipelines, the standard credential store works. The Jenkins Kubernetes Credentials provider plugin can also surface labeled Kubernetes Secrets *as* Jenkins credentials.

### External Secrets Operator
ESO syncs secrets from external stores (Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) into Kubernetes Secrets.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: nexus, namespace: jenkins }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: vault, kind: ClusterSecretStore }
  target: { name: nexus }
  data:
    - secretKey: username
      remoteRef: { key: kv/data/jenkins/nexus, property: username }
    - secretKey: password
      remoteRef: { key: kv/data/jenkins/nexus, property: password }
```

The Jenkins Kubernetes Credentials Provider plugin picks this up if you add the label `jenkins.io/credentials-type: usernamePassword`.

### HashiCorp Vault directly
The **HashiCorp Vault** Jenkins plugin works on Kubernetes too. Authenticate via the Kubernetes auth method (the controller's ServiceAccount token).

### Sealed Secrets / SOPS
For GitOps shops that want to commit secret YAML to Git, Sealed Secrets and SOPS-encrypted Secrets are common patterns. The encryption key lives in the cluster; only the cluster can decrypt.

## 7. Image Pull Credentials

For private registries:
1. Create a `kubernetes.io/dockerconfigjson` Secret:
   ```bash
   kubectl -n jenkins-builds create secret docker-registry regcred \
     --docker-server=registry.example.com \
     --docker-username=ci-bot \
     --docker-password=$REG_PAT
   ```
2. Reference it in your agent Pod template:
   ```yaml
   spec:
     imagePullSecrets:
       - name: regcred
   ```
3. Or attach it to the ServiceAccount once and forget:
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata: { name: jenkins-agent, namespace: jenkins-builds }
   imagePullSecrets:
     - name: regcred
   ```

For agent Pods to also *push* images, mount registry credentials into the build container (Module 04 example shows this for Kaniko).

## 8. Audit and Detection

- Enable Kubernetes audit logging at the API server level.
- Watch for suspicious actions by the Jenkins ServiceAccount (create/delete Pods outside expected namespaces, secret reads outside its namespace).
- Forward audit logs to your SIEM.

In the controller, keep the **Audit Trail** plugin from Part 1 — UI/API actions are still important to log.

## 9. Hardening Checklist

- [ ] Controller in its own namespace, agents in another.
- [ ] Controller ServiceAccount has Role (not ClusterRole) with only Pod management in agent namespace.
- [ ] Agents run with minimal/no ServiceAccount unless they need to deploy.
- [ ] PSS `baseline` or stricter enforced on agent namespace.
- [ ] NetworkPolicies in place (controller, agents).
- [ ] etcd encrypted at rest, or all sensitive secrets fetched from an external store.
- [ ] Image pull credentials per-namespace; rotated.
- [ ] No agent pod runs as root or with `privileged: true` (use rootless image builders like Kaniko).
- [ ] Audit logging enabled and shipped centrally.

## 10. Hands-On Exercise

1. Create namespaces `jenkins` (controller) and `jenkins-builds` (agents). Label them for PSS `baseline`.
2. Define the controller ServiceAccount + Role + RoleBinding so the controller can manage Pods in `jenkins-builds` but nothing else.
3. Apply a default-deny NetworkPolicy in `jenkins-builds`, then allow the egress paths your builds actually need.
4. Configure External Secrets Operator (or Sealed Secrets) to populate one credential without committing it plaintext.
5. Verify that a build agent runs as non-root with read-only root filesystem.

## 11. Knowledge Check
1. What's the maximum scope you should grant the controller's ServiceAccount?
2. Why split controller and agents into separate namespaces?
3. How would an agent Pod pull from a private registry?
4. Name two ways to inject secrets without committing them to Git.
5. What does `readOnlyRootFilesystem: true` defend against?

## What's Next
**Module 06** brings Configuration as Code into the Kubernetes context — driving the entire controller from ConfigMaps and Secrets in Git.
