# Module 08 — Deploying to Kubernetes from Jenkins

## Learning Objectives
- Deploy with `kubectl` and `helm` from a Jenkins pipeline.
- Promote releases through environments.
- Implement rolling, blue-green, and canary patterns on Kubernetes.
- Add approval gates and change controls.

## 1. Authentication to the Target Cluster

Two cases:

### Same cluster as Jenkins
Use a dedicated ServiceAccount in the *target* namespace. The agent Pod mounts its token. RBAC restricts to the resources the pipeline manages.

```yaml
# in the target namespace
apiVersion: v1
kind: ServiceAccount
metadata: { name: deployer, namespace: payments-prod }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: deployer, namespace: payments-prod }
rules:
- apiGroups: ["apps"]
  resources: ["deployments","statefulsets"]
  verbs: ["get","list","patch","update"]
- apiGroups: [""]
  resources: ["services","configmaps","secrets"]
  verbs: ["get","patch","update","create"]
```

Pipeline pod spec:
```yaml
spec:
  serviceAccountName: deployer  # set per-stage if necessary
```

### Different (remote) cluster
The agent needs a kubeconfig.
- Store a kubeconfig as a `Secret file` Jenkins credential.
- Bind it with `withCredentials([file(...)])` in the deploy stage.
- For cloud clusters, prefer short-lived tokens (IRSA, Workload Identity, federated OIDC) rather than static kubeconfigs.

## 2. Deploying with `kubectl`

```groovy
stage('deploy staging') {
  steps {
    container('kubectl') {
      sh '''
        kubectl -n payments-staging set image deployment/payments-api \
          payments-api=${IMAGE}
        kubectl -n payments-staging rollout status deployment/payments-api --timeout=180s
      '''
    }
  }
}
```

Verify rollout, then smoke test:
```groovy
retry(10) {
  sleep 6
  sh 'curl -fsS https://payments.staging.example.com/health'
}
```

## 3. Deploying with Helm

```groovy
stage('deploy staging') {
  steps {
    container('helm') {
      sh '''
        helm upgrade payments-api ./chart \
          --install \
          --namespace payments-staging \
          --values ./chart/values-staging.yaml \
          --set image.tag=${BUILD_NUMBER} \
          --atomic --timeout 5m
      '''
    }
  }
}
```

`--atomic` rolls back automatically if the release fails. Combined with `--timeout`, you get safe deploys for free.

### Releases vs raw manifests
Helm gives you:
- Reproducible release naming.
- Templating across env-specific values.
- Built-in rollback (`helm rollback`).

For more complex deployments use a chart you maintain. For trivial cases, raw manifests with `kubectl apply` are fine.

## 4. Environment Promotion

A typical Jenkins multi-environment pipeline:

```groovy
pipeline {
  agent { kubernetes { yamlFile 'ci/agent-pod.yaml' } }

  environment {
    IMAGE = "registry.example.com/payments-api:${env.BUILD_NUMBER}"
  }

  stages {
    stage('build & image') { /* ... from Module 07 ... */ }

    stage('deploy dev') {
      steps {
        container('helm') {
          sh 'helm upgrade payments-api ./chart -n payments-dev --install --set image.tag=${BUILD_NUMBER} --atomic --values ./chart/values-dev.yaml'
        }
      }
    }

    stage('integration tests') {
      steps { sh './run-integration-tests.sh payments-dev' }
    }

    stage('deploy staging') {
      when { branch 'main' }
      steps {
        container('helm') {
          sh 'helm upgrade payments-api ./chart -n payments-staging --install --set image.tag=${BUILD_NUMBER} --atomic --values ./chart/values-staging.yaml'
        }
      }
    }

    stage('approve prod') {
      when { branch 'main' }
      agent none
      steps {
        input message: 'Promote to production?', submitter: 'release-managers'
      }
    }

    stage('deploy prod') {
      when { branch 'main' }
      steps {
        container('helm') {
          sh 'helm upgrade payments-api ./chart -n payments-prod --install --set image.tag=${BUILD_NUMBER} --atomic --values ./chart/values-prod.yaml'
        }
      }
    }
  }
}
```

Single artifact (the image), promoted across environments. Same chart, different values files.

## 5. Rolling Updates (Kubernetes-Native)

`kubectl rollout` and Helm both use Kubernetes' native rolling update strategy:
- `maxSurge` — extra Pods above the replica count during rollout.
- `maxUnavailable` — Pods that can be down at once.
- Readiness probe is the gate — new Pods become "ready" only when they pass it, then old ones are terminated.

Tune in the Deployment spec:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0     # zero downtime if you have spare capacity
```

Jenkins watches `kubectl rollout status` and fails on timeout.

## 6. Blue-Green on Kubernetes

Two Deployments behind a Service:
- `payments-api-blue` (current live)
- `payments-api-green` (next version)

Service selector chooses one:
```yaml
spec:
  selector:
    app: payments-api
    track: blue
```

Pipeline:
1. Deploy `green`.
2. Run smoke tests targeting the `green` Deployment directly (use a `headless` service or an internal-only Ingress to reach it).
3. Patch the live Service to select `track: green`.
4. Keep `blue` around for one cycle for instant rollback.

```groovy
container('kubectl') {
  sh 'kubectl -n payments-prod set image deployment/payments-api-green payments-api=${IMAGE}'
  sh 'kubectl -n payments-prod rollout status deployment/payments-api-green --timeout=180s'
  sh './smoke-test.sh https://payments-green.internal'
  sh 'kubectl -n payments-prod patch service payments-api -p \'{"spec":{"selector":{"track":"green"}}}\''
}
```

Rollback = patch the service back to `track: blue`.

## 7. Canary

Send a small fraction of traffic to the new version.

### With an Ingress weighting
ingress-nginx supports `nginx.ingress.kubernetes.io/canary` annotations:
```yaml
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-weight: "10"
```

Two Ingress objects: a "stable" Ingress and a "canary" Ingress pointing at the new Service.

### Pipeline pattern
1. Deploy the canary at 10% weight.
2. Wait, observe metrics (error rate, latency).
3. Ramp to 50%, observe.
4. Ramp to 100%; remove the stable.

You can drive the weight changes from Jenkins:
```groovy
['10','25','50','100'].each { w ->
  sh "kubectl -n payments-prod annotate ingress payments-api-canary nginx.ingress.kubernetes.io/canary-weight=${w} --overwrite"
  sleep 300                              // 5 minutes between steps
  sh './check-metrics.sh canary'         // fail to abort the ramp
}
```

### With a service mesh
If you run Istio/Linkerd, prefer their traffic-splitting CRDs. Cleaner, more testable.

## 8. Approval Gates

Same as Part 1: `input` step, `agent none`, `submitter`.

Practical extras for prod deploys:
- Capture the approver and ticket ID in the build description.
- Post the approval link to Slack so reviewers can click in-context.
- Auto-abort the pipeline if approval doesn't happen within N hours (`timeout`).

```groovy
stage('approve prod') {
  agent none
  options { timeout(time: 4, unit: 'HOURS') }
  steps {
    script {
      def reply = input(
        message: 'Promote to production?',
        submitter: 'release-managers',
        parameters: [string(name: 'TICKET', description: 'Change ticket ID')]
      )
      currentBuild.description = "Approved by: ${reply.submitter}; ticket: ${reply.TICKET}"
    }
  }
}
```

## 9. Change Management Integration

For regulated environments:
- Open a change ticket via REST at the start of the prod stage.
- Block the deploy until the ticket reaches an approved state.
- Close the ticket on success; reopen with error details on failure.

Wrap this in a Shared Library step (`acmeChangeTicket()`) so every pipeline does it consistently.

## 10. Hooks and Health Checks

After deploy:
- Wait for `kubectl rollout status` (Deployments) or readiness across pods (StatefulSet).
- Run a synthetic transaction (sign in, make a fake order, etc.).
- Watch error rate in your monitoring for N minutes; fail the pipeline if it spikes.

If health fails:
- For Helm: `helm rollback payments-api 0 -n payments-prod`.
- For raw: `kubectl rollout undo deployment payments-api`.

Make rollback a *separately runnable* parameterized job (Part 1 Module 09). Promote and rollback use the same code path with different image tags.

## 11. Hands-On Exercise

1. Take your image from Module 07.
2. Add a `chart/` directory with a small Helm chart and `values-dev/staging/prod.yaml`.
3. Build a pipeline that:
   - Builds the image.
   - Deploys to `dev` automatically.
   - Runs an integration test.
   - On `main`, deploys to `staging`, waits for approval, deploys to `prod`.
4. Implement either blue-green or a 25% canary on the prod deploy.
5. Force a failing readiness probe in your chart; confirm Helm rolls back automatically because of `--atomic`.

## 12. Knowledge Check
1. Why prefer Workload Identity / IRSA over static kubeconfigs?
2. What does `helm --atomic` do?
3. In a blue-green setup on K8s, what does "flip the switch" mean concretely?
4. How would you implement a 10% → 50% → 100% canary ramp from a Jenkins pipeline?
5. What single step belongs in *every* post-deploy stage?

## What's Next
**Module 09** brings testing and quality gates into the K8s pipeline — ephemeral test environments per PR.
