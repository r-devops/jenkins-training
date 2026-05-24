# Module 04 — Kubernetes Plugin & Dynamic Agents

## Learning Objectives
- Configure the Kubernetes plugin to launch agents in your cluster.
- Author pod templates as YAML in a `Jenkinsfile`.
- Build multi-container agents and share workspaces between them.
- Pick a sensible caching strategy.

## 1. The Kubernetes Plugin

The Kubernetes plugin makes Jenkins create an **agent Pod for every build**, runs the build in it, and deletes it at the end.

The chart from Module 03 already installed it. To verify:
- **Manage Jenkins → Plugins → Installed** — confirm `kubernetes` is present.
- **Manage Jenkins → Clouds** — a `kubernetes` cloud should already exist (the chart creates it).

## 2. Cloud Configuration

The cloud entry is what tells Jenkins where to launch agents. The chart pre-configures most of it. The fields you might tune:

- **Name:** `kubernetes`.
- **Kubernetes URL:** blank (uses in-cluster config).
- **Kubernetes Namespace:** where to create agent Pods — `jenkins` by default; we'll often use a separate `jenkins-builds` namespace later.
- **Credentials:** the controller's ServiceAccount token (auto-detected).
- **Jenkins URL:** `http://jenkins.jenkins.svc.cluster.local:8080`.
- **Jenkins tunnel:** `jenkins-agent.jenkins.svc.cluster.local:50000` (for legacy JNLP).
- **Connection timeouts:** raise to 5–10s if your cluster is slow.
- **Container Cap / Pod Retention:** how many concurrent Pods; how long to keep Pods after a build for debugging (`Never` is the default).

For JCasC:
```yaml
jenkins:
  clouds:
    - kubernetes:
        name: kubernetes
        namespace: jenkins-builds
        jenkinsUrl: http://jenkins.jenkins.svc.cluster.local:8080
        jenkinsTunnel: jenkins-agent.jenkins.svc.cluster.local:50000
        connectTimeout: 5
        readTimeout: 15
        containerCapStr: "100"
        podRetention: never
```

## 3. Your First Pod Template (in the cloud)

A pod template is the spec the plugin uses to create an agent Pod. The cloud config can define defaults — but in practice we declare templates inline in `Jenkinsfile`s.

A minimal cloud-level template via JCasC:
```yaml
jenkins:
  clouds:
    - kubernetes:
        templates:
          - name: default
            label: k8s-default
            containers:
              - name: jnlp
                image: jenkins/inbound-agent:latest
            yamlMergeStrategy: merge
```

A pipeline targeting it:
```groovy
pipeline {
  agent { label 'k8s-default' }
  stages { stage('hi') { steps { sh 'hostname; uname -a' } } }
}
```

This works, but you're stuck with one image. We want richer agents.

## 4. Pod Templates in `Jenkinsfile`

The expressive pattern: declare the Pod's YAML in the `Jenkinsfile`.

```groovy
pipeline {
  agent {
    kubernetes {
      yaml '''
      apiVersion: v1
      kind: Pod
      spec:
        containers:
          - name: jnlp
            image: jenkins/inbound-agent:latest
          - name: maven
            image: maven:3.9-eclipse-temurin-17
            command: ['sleep']
            args: ['9999999']
          - name: kaniko
            image: gcr.io/kaniko-project/executor:latest
            command: ['sleep']
            args: ['9999999']
      '''
      defaultContainer 'maven'
    }
  }

  stages {
    stage('build') {
      steps { sh 'mvn -B verify' }       // runs in 'maven' (default)
    }
    stage('image') {
      steps {
        container('kaniko') {            // switch container for this step
          sh '/kaniko/executor --dockerfile=Dockerfile --context=. --no-push'
        }
      }
    }
  }
}
```

Things to notice:
- The `jnlp` container is the Jenkins agent process — must be present.
- Tool containers run `sleep` so they stay alive while Jenkins drives them via `kubectl exec`.
- `container('name') { ... }` switches the working container for a block of steps.
- All containers share the workspace volume — files written in `maven` are visible to `kaniko`.

### `yamlFile` variant
Keep the pod spec in a separate file and reference it:
```groovy
agent {
  kubernetes {
    yamlFile 'ci/agent-pod.yaml'
    defaultContainer 'maven'
  }
}
```
Easier to version and lint.

### `inheritFrom`
You can inherit a base pod template and override pieces:
```groovy
agent { kubernetes { inheritFrom 'jvm-base'; yaml '...' } }
```

## 5. Standard Containers

| Container | Image | Purpose |
|-----------|-------|---------|
| `jnlp` | `jenkins/inbound-agent` | Connects to controller; required. |
| `maven` | `maven:3.9-eclipse-temurin-17` | Java/Maven builds. |
| `gradle` | `gradle:8-jdk17` | Gradle builds. |
| `node` | `node:20` | JavaScript builds. |
| `python` | `python:3.12-slim` | Python builds. |
| `kaniko` | `gcr.io/kaniko-project/executor` | OCI image builds (no Docker daemon). |
| `kubectl` | `bitnami/kubectl` | Cluster deploys. |
| `helm` | `alpine/helm` | Helm deploys. |
| `aws` | `amazon/aws-cli` | AWS CLI. |

Trim to what each pipeline actually needs — a 6-container Pod is slow to schedule.

## 6. Resources, Selectors, Tolerations

Set requests/limits per container in the pod spec — otherwise scheduling is unpredictable.

```yaml
- name: maven
  image: maven:3.9-eclipse-temurin-17
  resources:
    requests: { cpu: "500m", memory: "1Gi" }
    limits:   { cpu: "2",    memory: "4Gi" }
```

Pin agents to a "builds" node pool:
```yaml
spec:
  nodeSelector:
    workload: ci
  tolerations:
    - key: workload
      operator: Equal
      value: ci
      effect: NoSchedule
```

On managed clusters use a separate node pool with spot/preemptible instances for builds.

## 7. Workspace Volume

By default Jenkins mounts an `emptyDir` for the workspace, shared across containers in the Pod. The Pod dies when the build ends — workspace is gone.

If you must persist:
- Mount a PVC (`ReadWriteOnce`) — but then only one build at a time can use it.
- Better: use a remote artifact store, or a cache service (see next section).

## 8. Caching Strategies

Naive dynamic agents make every build a cold build. Speed it up:

### Local cache via persistent volume per agent label
- Define a pool of "warm" agents tied to a label like `k8s-maven-cached`.
- Mount a PVC for the Maven local repo (`/root/.m2`).
- Use `podRetention: never` *or* `onFailure` carefully — caches survive only across builds on the same Pod.

### Sidecar cache server
- Run a small Nexus or sccache server in-cluster.
- Agents fetch dependencies through it. The cache survives all Pods.

### Cloud cache services
- Use a remote cache (S3/GCS-backed) for Bazel/Gradle/Nx/Turborepo.
- Works without any persistent volumes.

### Image-baked dependencies
- Pre-build container images that already contain common dependencies.
- Trade-off: image grows, but cold-start is faster.

Pick one strategy and stick with it. Mixing leads to confusion.

## 9. Volumes for Secrets and Configs

Mount config files (e.g., `~/.npmrc`, `~/.docker/config.json`) from Secrets or ConfigMaps:
```yaml
spec:
  containers:
    - name: node
      image: node:20
      volumeMounts:
        - name: npmrc
          mountPath: /root/.npmrc
          subPath: .npmrc
  volumes:
    - name: npmrc
      secret:
        secretName: npmrc
```

Use this for registry credentials, kubeconfig pieces, GPG keys, etc.

## 10. Debugging Agent Pods

```bash
kubectl -n jenkins-builds get pods            # see live agents
kubectl -n jenkins-builds describe pod <name> # events, scheduling reasons
kubectl -n jenkins-builds logs <pod> -c jnlp  # the agent's own log
kubectl -n jenkins-builds exec -it <pod> -c maven -- sh
```

Set `podRetention: onFailure` while debugging — failed Pods stick around so you can inspect them.

## 11. A Realistic Example

```groovy
pipeline {
  agent {
    kubernetes {
      yaml '''
      apiVersion: v1
      kind: Pod
      metadata:
        labels: { app: jenkins-agent }
      spec:
        nodeSelector: { workload: ci }
        tolerations:
          - key: workload
            operator: Equal
            value: ci
            effect: NoSchedule
        containers:
          - name: jnlp
            image: jenkins/inbound-agent:latest
            resources:
              requests: { cpu: "200m", memory: "256Mi" }
              limits:   { cpu: "500m", memory: "512Mi" }
          - name: maven
            image: maven:3.9-eclipse-temurin-17
            command: ['sleep']; args: ['9999999']
            resources:
              requests: { cpu: "1",   memory: "2Gi" }
              limits:   { cpu: "2",   memory: "4Gi" }
          - name: kaniko
            image: gcr.io/kaniko-project/executor:latest
            command: ['sleep']; args: ['9999999']
            volumeMounts:
              - name: docker-config
                mountPath: /kaniko/.docker
        volumes:
          - name: docker-config
            secret:
              secretName: registry-creds
              items:
                - key: .dockerconfigjson
                  path: config.json
      '''
      defaultContainer 'maven'
    }
  }
  stages {
    stage('build') { steps { sh 'mvn -B verify' } }
    stage('image') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --dockerfile=Dockerfile \
              --context=. \
              --destination=registry.example.com/payments-api:${BUILD_NUMBER}
          '''
        }
      }
    }
  }
  post { always { junit '**/target/surefire-reports/*.xml' } }
}
```

## 12. Hands-On Exercise

1. Modify your `values.yaml` to give the Kubernetes cloud a dedicated agent namespace `jenkins-builds` (create the namespace and grant RBAC).
2. Create a `Jenkinsfile` with a multi-container pod (jnlp + maven + kaniko) using `yaml`.
3. Build, run unit tests, build the container with Kaniko, push to a registry. Use a Secret for registry credentials.
4. Watch agent Pods come and go in `kubectl get pods -n jenkins-builds -w`.
5. Add a Maven cache PVC and observe build time improve on subsequent runs.

## 13. Knowledge Check
1. Why is the `jnlp` container required?
2. What does `container('maven')` actually do?
3. How do you switch which container is the default in a pipeline?
4. Name three caching strategies and a trade-off of each.
5. How do you keep failed agent Pods around for debugging?

## What's Next
**Module 05** covers identity, secrets, and security on Kubernetes — RBAC, network policies, External Secrets, and image pull credentials.
