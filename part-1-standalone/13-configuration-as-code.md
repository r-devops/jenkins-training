# Module 13 — Configuration as Code (JCasC)

## Learning Objectives
- Explain why Jenkins controller configuration belongs in Git.
- Install and bootstrap the Configuration as Code (JCasC) plugin.
- Express auth, credentials, clouds, agents, and tools as YAML.
- Validate and reload configuration safely.
- Inject secrets without committing them to Git.

## 1. The Problem with UI-Driven Config

A typical Jenkins instance is configured by clicking through dozens of screens. That config lives in `JENKINS_HOME/*.xml` files. Problems:
- No history of who changed what (without extra plugins).
- Hard to recreate the controller — you'd need to remember every click.
- Drift between dev/staging/prod controllers.
- Onboarding a new team means manual setup over and over.

JCasC inverts this: you write a YAML file, and Jenkins applies it on startup. Want a new controller? Bring up the JVM, point it at the YAML, you're done.

## 2. Installing the JCasC Plugin

**Manage Jenkins → Plugins → Available → Configuration as Code → Install**.

After install you'll see a new menu: **Manage Jenkins → Configuration as Code**. Three useful actions:
- **Reload existing configuration** — re-applies the YAML.
- **Apply new configuration** — apply a different YAML file.
- **View configuration** — Jenkins exports its current state as YAML. Great for discovery.
- **Documentation** — schema for every section, generated from your installed plugins.

## 3. Pointing Jenkins at a YAML File

Set the environment variable `CASC_JENKINS_CONFIG` (or the system property `casc.jenkins.config`):

```
CASC_JENKINS_CONFIG=/var/jenkins_config/jenkins.yaml
```

It can also be a directory (Jenkins concatenates every `.yaml` file in it), a URL, or a comma-separated list.

On a Debian install, set it in `/etc/default/jenkins`:
```bash
JAVA_ARGS="-Xms2g -Xmx4g -Dcasc.jenkins.config=/var/jenkins_config/jenkins.yaml"
```

Restart Jenkins. Check **Manage Jenkins → Configuration as Code → View configuration** to confirm it loaded.

## 4. A Minimal `jenkins.yaml`

```yaml
jenkins:
  systemMessage: "Managed by JCasC. Do not edit through the UI."
  numExecutors: 0                          # no builds on the controller
  mode: EXCLUSIVE                          # only run jobs that match labels

  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: admin
          password: ${ADMIN_PASSWORD}      # from env var, see secrets section
        - id: developer
          password: ${DEVELOPER_PASSWORD}

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

  globalNodeProperties:
    - envVars:
        env:
          - key: ORG_NAME
            value: acme

unclassified:
  location:
    url: https://jenkins.example.com/
    adminAddress: jenkins-admin@example.com

  timestamper:
    allPipelines: true

  slackNotifier:
    teamDomain: acme
    tokenCredentialId: slack-bot-token
```

Restart Jenkins (or hit **Reload existing configuration**). The system message, executors, users, roles, URL, and Slack settings are all set from this single file.

## 5. The Big Sections

JCasC YAML has a fixed top-level shape:

```yaml
jenkins: ...        # core controller (executors, security, nodes, clouds, tools)
credentials: ...    # credentials (often via env vars or external secrets)
security: ...       # script approvals, queue settings, etc.
tool: ...           # tool installers: JDK, Maven, Gradle, Node
unclassified: ...   # plugin-contributed config; the kitchen sink
jobs: ...           # job DSL or scripted job definitions
```

The set of valid keys depends on which plugins are installed. **Manage Jenkins → Configuration as Code → Documentation** is the authoritative reference for your specific instance.

## 6. Credentials in JCasC

```yaml
credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: nexus
              username: jenkins-ci
              password: ${NEXUS_PASSWORD}
          - string:
              scope: GLOBAL
              id: slack-bot-token
              secret: ${SLACK_BOT_TOKEN}
          - basicSSHUserPrivateKey:
              scope: GLOBAL
              id: git
              username: git
              privateKeySource:
                directEntry:
                  privateKey: ${GITHUB_SSH_KEY}
```

## 7. Tools

```yaml
tool:
  jdk:
    installations:
      - name: jdk-17
        properties:
          - installSource:
              installers:
                - adoptOpenJdkInstaller:
                    id: "jdk-17.0.10+7"
  maven:
    installations:
      - name: maven-3.9
        properties:
          - installSource:
              installers:
                - maven:
                    id: "3.9.6"
```

Pipelines then reference `tools { jdk 'jdk-17'; maven 'maven-3.9' }`.

## 8. Agents and Clouds

Permanent SSH agent declared in YAML:
```yaml
jenkins:
  nodes:
    - permanent:
        name: linux-agent-01
        labelString: "linux docker"
        numExecutors: 4
        remoteFS: "/home/jenkins/agent"
        launcher:
          ssh:
            host: agent-01.example.com
            port: 22
            credentialsId: agent-ssh
            sshHostKeyVerificationStrategy:
              knownHostsFileVerificationStrategy: {}
```

(Cloud agent providers — Docker, Kubernetes, EC2 — are configured similarly under `jenkins.clouds`. Kubernetes is covered in Part 2.)

## 9. Secrets in JCasC

You don't commit passwords to Git. Three common patterns:

### Environment variables
`${ADMIN_PASSWORD}` in YAML is expanded from the `ADMIN_PASSWORD` env var when Jenkins starts. Set them in your service unit, container env, or secret store.

### File-based secrets
```yaml
password: ${file:/run/secrets/admin_password}
```

### External secrets manager
Use the **HashiCorp Vault Plugin** to fetch credentials at startup. Or use a sidecar that materializes secrets to env vars before Jenkins starts.

### Default expressions
```yaml
password: ${ADMIN_PASSWORD:-changeit}
```

## 10. Validating and Reloading

### Static validation
Lint the YAML against your installed plugins. The plugin exposes a validator at:
```
JENKINS_URL/configuration-as-code/checkNewSource?newSource=...
```

Or use the JCasC export feature to dump the current config and compare to your file — anything outside your YAML is drift to bring back into the YAML.

### Reload
- **Manage Jenkins → Configuration as Code → Reload existing configuration** applies the file without a restart.
- Some changes (security realm, port) still need a restart.

## 11. Workflow

Recommended workflow:

1. Put `jenkins.yaml` in a Git repository (could be the same as your shared library).
2. Open a PR for every config change.
3. CI for the YAML repo:
   - Lint syntax.
   - Diff against `JENKINS_URL/configuration-as-code/export`.
4. On merge to `main`, a Jenkins job (or external job) pulls the YAML and triggers a JCasC reload.
5. Block UI changes — convention or via "configuration is read-only" plugins.

Alternative: bake `jenkins.yaml` into the controller's container image. The image is rebuilt and rolled out via your normal deploy process.

## 12. Pitfalls

- **Plugin updates can change YAML schema.** When you upgrade plugins, re-validate the YAML.
- **Order matters in some sections.** Authorization can lock you out if applied before users exist; structure your YAML carefully.
- **The export contains everything.** Diffing it against your file shows drift, but you don't want to *commit* the full export — keep your YAML opinionated, not exhaustive.
- **JCasC + jobs as code:** JCasC handles controller config, not jobs. For jobs, use Job DSL (next module).

## 13. Hands-On Exercise

1. Take your current Jenkins controller. Export its config (**Configuration as Code → View configuration**) and save the YAML.
2. Trim the export down to: security realm, authorization, URL, executors, one credential, one agent.
3. Set `CASC_JENKINS_CONFIG=/etc/jenkins/jenkins.yaml`, restart.
4. Verify everything still works.
5. Move the file to a Git repo. Make a change (e.g., system message), commit, push.
6. Set up a tiny pipeline that reloads JCasC when the repo's `main` updates.

## 14. Knowledge Check
1. Where does Jenkins look for the JCasC YAML file?
2. How are secrets injected without writing them to Git?
3. What can JCasC NOT configure (that you'd typically need)?
4. How do you confirm that the YAML matches the controller's actual state?
5. Why pin plugin versions when adopting JCasC?

## What's Next
**Module 14** continues the everything-as-code thread with Job DSL and plugin-management-as-code.
