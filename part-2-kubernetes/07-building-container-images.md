# Module 07 — Building Container Images in Kubernetes Agents

## Learning Objectives
- Compare Docker-in-Docker, Kaniko, Buildah, and BuildKit (rootless) for in-cluster builds.
- Push to ECR/GCR/ACR/Harbor from a Jenkins agent.
- Sign images and emit SBOMs.
- Cache layers without compromising security.

## 1. The Problem

You want to build container images from Jenkins. Traditionally:
```
docker build -t app:1.0 .
docker push app:1.0
```

That uses the Docker daemon — and a Docker daemon needs **root** + access to the host's kernel. On a shared Kubernetes cluster, granting that to every build is unsafe.

## 2. Options

| Tool | Daemon needed? | Root needed? | Notes |
|------|---------------|--------------|-------|
| Docker-in-Docker (DinD) | Yes (sidecar) | Yes (`privileged: true`) | Familiar; broad compat; least secure. |
| Kaniko | No | Optional (works rootless) | Google project. Mature. Slower on large images. |
| Buildah | No | Rootless mode | Good Dockerfile compat. |
| BuildKit (rootless) | Yes (rootless daemon) | No | Fast, modern; better caching than Kaniko. |

For most teams: **start with Kaniko** (simple, secure). Move to **BuildKit rootless** if you need its caching speed.

## 3. Kaniko

Kaniko runs as a single binary inside a container, builds the image, and pushes to a registry.

### Pod template
```yaml
containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    command: ['sleep']
    args: ['9999999']
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
```

### Pipeline step
```groovy
container('kaniko') {
  sh '''
    /kaniko/executor \
      --dockerfile=Dockerfile \
      --context=$(pwd) \
      --destination=registry.example.com/payments-api:${BUILD_NUMBER} \
      --destination=registry.example.com/payments-api:${GIT_COMMIT} \
      --cache=true \
      --cache-repo=registry.example.com/payments-api-cache
  '''
}
```

### Notes
- The Secret `registry-creds` is a `kubernetes.io/dockerconfigjson` Secret (see Module 05).
- `--cache=true --cache-repo=...` keeps layer caches in a separate registry repo — major speed boost for repeat builds.
- Kaniko can read the build context from S3/GCS too; useful for very large monorepos.

## 4. BuildKit (Rootless)

`docker buildx` uses BuildKit under the hood. The rootless variant runs without privileged.

### Pod template
```yaml
containers:
  - name: buildkit
    image: moby/buildkit:v0.13.0-rootless
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
      seccompProfile: { type: Unconfined }   # required for user namespaces
    args:
      - --oci-worker-no-process-sandbox
    volumeMounts:
      - name: docker-config
        mountPath: /home/user/.docker
```

### Pipeline step
```groovy
container('buildkit') {
  sh '''
    buildctl build \
      --frontend dockerfile.v0 \
      --local context=. \
      --local dockerfile=. \
      --opt build-arg:VERSION=${BUILD_NUMBER} \
      --output type=image,name=registry.example.com/payments-api:${BUILD_NUMBER},push=true
  '''
}
```

BuildKit shines when:
- You have many parallel builds and want shared caching.
- You need advanced features (mounts, secrets in `--mount=type=secret`, multi-platform builds).

## 5. Buildah

Buildah is Podman's lower-level build tool. Rootless out of the box, good Docker compat.

```yaml
- name: buildah
  image: quay.io/buildah/stable:latest
  securityContext: { runAsUser: 1000 }
```

```groovy
container('buildah') {
  sh '''
    buildah --storage-driver vfs bud -t registry.example.com/payments-api:${BUILD_NUMBER} .
    buildah --storage-driver vfs push registry.example.com/payments-api:${BUILD_NUMBER}
  '''
}
```

Slower than BuildKit; faster than Kaniko on some workloads. The `vfs` storage driver is the universal-fallback choice in restricted environments.

## 6. Docker-in-Docker (DinD)

If you must:
```yaml
containers:
  - name: dind
    image: docker:24-dind
    securityContext:
      privileged: true
    args: ["--host=tcp://0.0.0.0:2375"]
  - name: dind-client
    image: docker:24-cli
    env:
      - name: DOCKER_HOST
        value: tcp://localhost:2375
```

Then `docker build` works as you'd expect. Trade-offs:
- Requires `privileged: true`. Often blocked by PSS `baseline`/`restricted`.
- Each Pod boots its own daemon — slow.
- No layer cache between Pods (unless you mount a PVC for `/var/lib/docker`, which has its own integrity risks).

Use DinD only as a transitional fallback.

## 7. Pushing to Cloud Registries

### AWS ECR
- IAM-based auth via IRSA (preferred): attach an IAM role to the agent's ServiceAccount with `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:UploadLayer*`, `ecr:PutImage`.
- The agent calls `aws ecr get-login-password` and writes it to `~/.docker/config.json` before running Kaniko/BuildKit.

### GCP Artifact Registry / GCR
- Workload Identity (preferred): bind the agent's ServiceAccount to a GCP service account with `artifactregistry.writer`.
- Auth via gcloud helper: `gcloud auth configure-docker us-central1-docker.pkg.dev`.

### Azure Container Registry
- Workload Identity (preferred); the agent gets an AAD token, exchanged for an ACR token.
- Or a service principal: `az acr login`.

### Self-hosted (Harbor, Nexus)
- Standard username/password Secret as a `dockerconfigjson`.

## 8. Image Signing and SBOMs

### Cosign
After build, sign the image:
```groovy
container('cosign') {
  withCredentials([file(credentialsId: 'cosign-key', variable: 'KEY')]) {
    sh 'cosign sign --key $KEY registry.example.com/payments-api:${BUILD_NUMBER}'
  }
}
```

Or keyless signing via OIDC (Sigstore Fulcio) if your cluster supports the workload identity setup.

Consumers verify:
```bash
cosign verify --key cosign.pub registry.example.com/payments-api:1234
```

### SBOM generation
Use `syft` to generate a Software Bill of Materials:
```groovy
container('syft') {
  sh 'syft registry.example.com/payments-api:${BUILD_NUMBER} -o spdx-json > sbom.json'
  archiveArtifacts 'sbom.json'
}
```

Push SBOM to your registry as an OCI artifact, or store with the build record.

### Vulnerability scan
```groovy
container('trivy') {
  sh '''
    trivy image \
      --severity HIGH,CRITICAL \
      --exit-code 1 \
      registry.example.com/payments-api:${BUILD_NUMBER}
  '''
}
```

Fail the build on high/critical vulnerabilities. Add the result as a build report.

## 9. Layer Caching Strategies

| Strategy | Speed | Complexity | Notes |
|----------|-------|------------|-------|
| No cache | Slow | None | Baseline. |
| Kaniko `--cache=true` + cache repo | Fast | Low | Recommended starting point. |
| BuildKit shared cache (registry) | Faster | Medium | Best when many builds run in parallel. |
| BuildKit local cache on PVC | Fastest | High | Coordinate concurrent writers carefully. |

## 10. A Realistic Pipeline

```groovy
pipeline {
  agent {
    kubernetes {
      yaml '''
      apiVersion: v1
      kind: Pod
      spec:
        serviceAccountName: jenkins-builder
        containers:
          - name: jnlp
            image: jenkins/inbound-agent:latest
          - name: maven
            image: maven:3.9-eclipse-temurin-17
            command: ['sleep']; args: ['9999999']
          - name: kaniko
            image: gcr.io/kaniko-project/executor:latest
            command: ['sleep']; args: ['9999999']
            volumeMounts:
              - name: docker-config
                mountPath: /kaniko/.docker
          - name: cosign
            image: gcr.io/projectsigstore/cosign:v2
            command: ['sleep']; args: ['9999999']
          - name: trivy
            image: aquasec/trivy:latest
            command: ['sleep']; args: ['9999999']
        volumes:
          - name: docker-config
            secret:
              secretName: registry-creds
              items: [ { key: .dockerconfigjson, path: config.json } ]
      '''
      defaultContainer 'maven'
    }
  }

  environment {
    IMAGE = "registry.example.com/payments-api:${env.BUILD_NUMBER}"
  }

  stages {
    stage('build & test') {
      steps { sh 'mvn -B verify' }
      post { always { junit '**/target/surefire-reports/*.xml' } }
    }

    stage('image') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --dockerfile=Dockerfile \
              --context=$(pwd) \
              --destination=${IMAGE} \
              --cache=true \
              --cache-repo=registry.example.com/payments-api-cache
          '''
        }
      }
    }

    stage('scan') {
      steps {
        container('trivy') {
          sh 'trivy image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE}'
        }
      }
    }

    stage('sign') {
      steps {
        container('cosign') {
          withCredentials([file(credentialsId: 'cosign-key', variable: 'KEY')]) {
            sh 'cosign sign --key $KEY ${IMAGE}'
          }
        }
      }
    }
  }
}
```

## 11. Hands-On Exercise

1. Pick a small app (or use the sample from earlier modules).
2. Build and push its image with Kaniko to a private registry.
3. Add layer caching; observe the second build is significantly faster.
4. Add Trivy scanning; intentionally pin a vulnerable base image and confirm the build fails.
5. Add cosign signing; verify the signature from your laptop.

## 12. Knowledge Check
1. Why is DinD often unsafe on shared Kubernetes clusters?
2. What does Kaniko's `--cache-repo` flag do?
3. How does IRSA / Workload Identity avoid static cloud credentials in your agents?
4. What's the difference between cosign signing and an SBOM?
5. Which scanner step would you put *before* pushing the image, and which *after*?

## What's Next
**Module 08** uses Jenkins to deploy applications onto Kubernetes — `kubectl`, `helm`, environment promotion, approvals.
