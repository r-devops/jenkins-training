# Module 06 — Configuration as Code on Kubernetes

## Learning Objectives
- Drive the controller's configuration from Helm values + JCasC ConfigMaps.
- Manage plugins as code on Kubernetes.
- Inject Secrets from External Secrets / Sealed Secrets / Vault.
- Reproducibly rebuild a controller from a Git repo.

## 1. The Goal

On Kubernetes, "config as code" means: **the cluster contains nothing you didn't put in Git.** If your Helm release is deleted and the PVC wiped, the same Git repo and the same `helm install` brings everything back — same plugins, same auth, same agents.

## 2. Three Layers of Code

| Layer | Lives in | Managed via |
|-------|----------|-------------|
| Cluster shape | `values.yaml`, manifests | Helm |
| Controller config | `jenkins.yaml` (JCasC) | ConfigMap mounted into the controller |
| Jobs | Job DSL scripts | Seed pipeline (Part 1 Module 14) |

All three live in a single `jenkins-config` Git repo (or a small number of related repos).

## 3. JCasC via the Helm Chart

The chart's `controller.JCasC.configScripts` is the cleanest way:

```yaml
controller:
  JCasC:
    defaultConfig: true
    configScripts:
      welcome: |
        jenkins:
          systemMessage: "Managed by Helm + JCasC."
          numExecutors: 0
          mode: EXCLUSIVE

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

      clouds: |
        jenkins:
          clouds:
            - kubernetes:
                name: kubernetes
                namespace: jenkins-builds
                jenkinsUrl: http://jenkins.jenkins.svc.cluster.local:8080
                jenkinsTunnel: jenkins-agent.jenkins.svc.cluster.local:50000

      libraries: |
        unclassified:
          globalLibraries:
            libraries:
              - name: acme-ci
                defaultVersion: v1.0.0
                retriever:
                  modernSCM:
                    scm:
                      git:
                        remote: "git@github.com:acme/pipeline-library.git"
                        credentialsId: "git"
```

Each `configScripts` key is merged into one large `jenkins.yaml` that the chart mounts into the controller. The chart automatically reloads JCasC when the ConfigMap changes.

## 4. Plugins as Code

The chart's `controller.installPlugins` list is read on startup:

```yaml
controller:
  installPlugins:
    - kubernetes:4253.v7700d91739e5
    - workflow-aggregator:600.v4d3ed_2a_24f0e
    - git:5.2.1
    - configuration-as-code:1775.v810dc950b_514
    - job-dsl:1.87
    - matrix-auth:3.2.1
    - role-strategy:711.v3046bb_24f81d
    - prometheus:2.5.0
    - slack:684.v833089650554
```

**Always pin versions.** A floating dependency is the easiest way to wake up to a broken controller.

For larger lists, keep them in a separate file and reference via Helm `--values plugins.yaml`:

```yaml
controller:
  installPlugins:
    - kubernetes:4253.v7700d91739e5
    # ...etc...
```

### Custom controller image (alternative)
You can bake plugins into the image instead. Helpful when:
- The chart's plugin install at runtime is too slow.
- You want immutable infra: same image = same controller.

```dockerfile
FROM jenkins/jenkins:2.452.1-lts-jdk17

COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

COPY jcasc/ /var/jenkins_home/casc_configs/
ENV CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs/
```

Reference it in your Helm values:
```yaml
controller:
  image: registry.example.com/jenkins
  tag: 2.452.1-acme-3
  installPlugins: false      # don't double-install
  customJenkinsLabels: []
```

## 5. Secrets That Power JCasC

JCasC supports `${ENV_VAR}` and `${file:/path}` expansion. On Kubernetes you have:

### Mount env vars from a Secret
```yaml
controller:
  containerEnvFrom:
    - secretRef: { name: jenkins-jcasc-env }
```
JCasC YAML references `${ADMIN_PASSWORD}` etc.

### Mount files from a Secret
```yaml
controller:
  additionalVolumes:
    - name: jcasc-secrets
      secret: { secretName: jenkins-jcasc-files }
  additionalVolumeMounts:
    - name: jcasc-secrets
      mountPath: /run/jenkins-secrets
      readOnly: true
```
JCasC YAML references `${file:/run/jenkins-secrets/github-pat}`.

### External Secrets Operator
Use ESO to materialize `jenkins-jcasc-env` and `jenkins-jcasc-files` from Vault/SSM/SecretManager. The Helm values stay in Git; the secrets never do.

### Sealed Secrets
Encrypt secrets to YAML; commit the encrypted YAML; the cluster decrypts at apply time.

## 6. Bootstrapping Credentials via JCasC

```yaml
configScripts:
  credentials: |
    credentials:
      system:
        domainCredentials:
          - credentials:
              - usernamePassword:
                  scope: GLOBAL
                  id: git
                  username: jenkins-ci
                  password: ${GIT_PASSWORD}
              - string:
                  scope: GLOBAL
                  id: slack-bot-token
                  secret: ${SLACK_TOKEN}
              - usernamePassword:
                  scope: GLOBAL
                  id: registry-creds
                  username: jenkins-ci
                  password: ${REGISTRY_PASSWORD}
```

Combined with ESO/Sealed Secrets, credentials become reproducible — no clicking through "Add Credential" in the UI.

## 7. Job DSL via JCasC

Bootstrap the seed job (Module 14, Part 1) from JCasC:

```yaml
configScripts:
  seed: |
    jobs:
      - script: >
          pipelineJob('seed') {
            definition {
              cpsScm {
                scm {
                  git {
                    remote { url 'git@github.com:acme/jenkins-config.git'; credentials 'git' }
                    branches('main')
                  }
                }
                scriptPath('seed/Jenkinsfile')
              }
            }
            triggers { cron('H/30 * * * *') }
          }
```

The chain becomes:
- Helm install/upgrade → JCasC applied → seed job created → seed job runs Job DSL → folders, multibranch projects, admin jobs all created.

## 8. Repository Layout

```
jenkins-config/
├── chart/
│   ├── values.yaml            # Helm values (the bulk of the config)
│   ├── values-prod.yaml       # prod-specific overrides
│   └── values-staging.yaml    # staging overrides
├── jcasc/                     # broken into focused files (if not inline in values)
│   ├── 00-jenkins.yaml
│   ├── 10-auth.yaml
│   ├── 20-clouds.yaml
│   └── 30-libraries.yaml
├── plugins.yaml               # the plugin pin list
├── seed/
│   └── jobs.groovy            # Job DSL
├── secrets/                   # SealedSecrets or ExternalSecrets manifests
│   ├── jcasc-env.yaml
│   └── registry-creds.yaml
├── Dockerfile                 # optional: pre-baked image
└── README.md
```

## 9. The Deploy Workflow

A pull-request-driven flow:

1. Engineer opens a PR against `jenkins-config`.
2. CI validates:
   - YAML lints.
   - JCasC schema check (`configuration-as-code/checkNewSource` in a throwaway controller, or `helm template` + a JCasC validator).
   - `helm template` succeeds.
3. PR approved and merged.
4. A delivery pipeline runs `helm upgrade jenkins jenkins/jenkins -f values.yaml ...`.
5. The controller reloads JCasC. The seed job (re)runs Job DSL. Everything is in sync with Git.

This pipeline runs on Jenkins itself. To avoid Jenkins managing its own break-glass changes, have an out-of-band path (a small `kubectl` admin job, or a manual `helm upgrade` from a bastion) for emergencies.

## 10. Reproducibility Test

Once a quarter, in a non-prod cluster:
1. Delete the namespace (or the entire Helm release and PVC).
2. Re-create from Git.
3. Time it. Document the procedure.

If it doesn't work end-to-end, your "config as code" has gaps. Fill them.

## 11. Hands-On Exercise

1. Move all your Module 03 JCasC config into a separate `jcasc/` directory with multiple files. Use Helm's templating to assemble `configScripts` from them.
2. Set up External Secrets Operator with a small Vault dev server (or any of the cloud providers). Sync one secret into the cluster.
3. Reference that secret via `${ENV_VAR}` in JCasC.
4. Confirm the controller starts up with the value injected, and the value never appears in your Git repo.
5. Add `seed/jobs.groovy` declaring one multibranch project. Bootstrap the seed job from JCasC. Confirm the project appears after one Helm upgrade.

## 12. Knowledge Check
1. What's the difference between `installPlugins` in Helm values and a custom image with `plugins.txt`?
2. How does JCasC reference a secret without that secret being in Git?
3. Why split JCasC into multiple files under `configScripts`?
4. What's the typical chain from `helm upgrade` to "jobs exist"?
5. What's the reproducibility test, and why run it?

## What's Next
**Module 07** moves from configuring Jenkins to *using* it — building container images on Kubernetes agents.
