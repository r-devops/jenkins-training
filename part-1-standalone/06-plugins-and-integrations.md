# Module 06 — Plugins & Integrations

## Learning Objectives
- Use the Plugin Manager to install, update, and pin plugins safely.
- Identify the essential plugins every Jenkins admin should know.
- Wire Jenkins up to common build, test, quality, and notification tools.

## 1. The Plugin Manager

**Manage Jenkins → Plugins** has four tabs:
- **Updates** — plugins with newer versions.
- **Available plugins** — installable plugins.
- **Installed plugins** — what's currently installed.
- **Advanced** — manual upload (`.hpi` files), proxy config, update sites.

### Installing a plugin
1. **Available plugins** → search → tick → **Install**.
2. Optionally tick "Restart Jenkins when installation is complete".
3. Plugins are downloaded to `JENKINS_HOME/plugins/`.

### Updating a plugin
- Read the changelog first — major version bumps can break pipelines.
- Update during a quiet window; have a backup.
- Restart Jenkins to load the new version.

### Pinning and rollback
- To pin a version: download the specific `.hpi` from `plugins.jenkins.io` and **Advanced → Deploy Plugin**.
- To roll back: stop Jenkins, replace the `.hpi`/`.jpi` in `JENKINS_HOME/plugins/`, start Jenkins.
- Plugins as code (Module 12) makes this much cleaner.

### Dependency hell
Plugins depend on other plugins and on core Jenkins. The Plugin Manager will tell you about missing or incompatible dependencies. The **Plugin Health Score** on `plugins.jenkins.io` is a useful sanity check before adopting an unfamiliar plugin.

## 2. Essential Plugins

These cover ~80% of normal Jenkins use. Many ship with "suggested plugins".

### Core / Pipeline
- **Pipeline** (`workflow-aggregator`) — enables Pipelines.
- **Pipeline: Stage View** — visualizes stages.
- **Blue Ocean** *(optional)* — alternative UI focused on pipelines.
- **Pipeline Utility Steps** — read/write JSON/YAML/properties.

### SCM
- **Git** — core Git support.
- **GitHub Branch Source** — multibranch + PRs for GitHub.
- **GitLab plugin** / **Bitbucket Branch Source** — equivalents.

### Credentials
- **Credentials** + **Credentials Binding** — exposes credentials to builds as env vars.

### Authorization / Security
- **Matrix Authorization Strategy** — fine-grained permissions.
- **Role-based Authorization Strategy** — group permissions by role.
- **OWASP Markup Formatter** — safer HTML rendering for descriptions.

### UI & UX
- **Dashboard View** — custom dashboards.
- **Build Monitor View** — wallboard view for radiators.
- **Timestamper** — adds timestamps to console output.
- **AnsiColor** — renders ANSI colors in console.

### Build tools
- **Maven Integration** — first-class Maven support.
- **Gradle** — Gradle build steps and reporting.
- **NodeJS** — manage Node installations.

### Quality / Test
- **JUnit** — test result reporting.
- **HTML Publisher** — publish arbitrary HTML reports (coverage, lint).
- **Warnings Next Generation** — aggregates static analysis findings.
- **SonarQube Scanner** — scans and posts results to SonarQube.

### Notifications
- **Mailer** / **Email Extension (email-ext)** — email on success/failure.
- **Slack Notification** — Slack messages.
- **MS Teams** — Teams messages.

### Operations
- **Configuration as Code** (JCasC) — config-as-YAML.
- **Job DSL** — generate jobs from Groovy.
- **Monitoring** — JVM/system metrics.
- **Prometheus** — exposes metrics on `/prometheus`.

## 3. Wiring Up Build Tools

### Maven
1. **Manage Jenkins → Tools → Maven installations → Add** — name it `maven-3.9`.
2. In a pipeline:
   ```groovy
   tools { maven 'maven-3.9' }
   stages {
     stage('build') { steps { sh 'mvn -B verify' } }
   }
   ```
3. Or in a Freestyle job: **Build → Invoke top-level Maven targets**.

### Gradle
- Same pattern: declare a Gradle installation in **Tools**.
- In pipelines: `tools { gradle 'gradle-8' }` and `sh './gradlew build'`.

### Node.js
- Install the **NodeJS** plugin.
- Configure named installations under **Tools**.
- In pipelines: `tools { nodejs 'node-20' }` then `sh 'npm ci && npm test'`.

## 4. Quality Gates with SonarQube

1. Install **SonarQube Scanner** plugin.
2. **Manage Jenkins → System → SonarQube servers** — add server URL + authentication token credential.
3. In a pipeline:
   ```groovy
   stage('sonar') {
     steps {
       withSonarQubeEnv('sonar-prod') {
         sh 'mvn sonar:sonar'
       }
     }
   }
   stage('quality gate') {
     steps { timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
   }
   ```
4. SonarQube must call back to Jenkins with the result — set up the webhook on the Sonar side.

## 5. Test Reporting (JUnit)

Almost every test framework can emit JUnit-style XML.

In a pipeline:
```groovy
post {
  always {
    junit '**/target/surefire-reports/*.xml'
  }
}
```

The build page will show:
- Pass / fail / skip counts and trend graph.
- Drill-down into individual failed test cases with stack traces.

## 6. Notifications

### Slack
1. Create a Slack app and a "Bot token" with `chat:write` permission.
2. Install the **Slack Notification** plugin.
3. **Manage Jenkins → System → Slack** — configure workspace + token credential.
4. In a pipeline:
   ```groovy
   post {
     failure { slackSend channel: '#builds', color: 'danger', message: "Build failed: ${env.BUILD_URL}" }
     success { slackSend channel: '#builds', color: 'good',   message: "Build OK: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
   }
   ```

### Email
- Use **email-ext** for templated emails:
  ```groovy
  emailext(
    to: 'team@example.com',
    subject: "Build ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
    body: '${SCRIPT, template="groovy-html.template"}',
    mimeType: 'text/html'
  )
  ```

## 7. Artifact Repositories

You don't store binaries in Git. Push them to a repository instead.

### Nexus Repository
- Use the **Nexus Artifact Uploader** plugin or `mvn deploy` configured with credentials.
- Suitable for Maven/Gradle/Docker/npm.

### JFrog Artifactory
- Use the **JFrog plugin** or run the JFrog CLI in a build step.
- Supports build info publishing, vulnerability scanning, and promotion.

In both cases:
- Store credentials in the Jenkins credentials store, not in `pom.xml` or scripts.
- Tag artifacts with the build number, Git SHA, and branch.

## 8. Static Analysis

The **Warnings Next Generation** plugin parses output from dozens of tools:
- Checkstyle, PMD, SpotBugs (Java)
- ESLint (JS/TS)
- Pylint (Python)
- golangci-lint (Go)

Configure once in your pipeline:
```groovy
post {
  always {
    recordIssues tools: [checkStyle(pattern: '**/checkstyle-result.xml'),
                         spotBugs(pattern: '**/spotbugsXml.xml')]
  }
}
```

You get per-file issue trends and threshold gates.

## 9. Plugin Hygiene

- **Audit** — review **Manage Jenkins → Plugins → Installed** once a quarter. Remove what's unused.
- **Pin major versions** — don't blanket auto-update.
- **Track end-of-life** — some plugins go unmaintained; the Update Center will warn.
- **Restart after batch updates** — don't update plugins one by one with restarts in between.
- **Avoid niche plugins** if a small shell script can do the job.

## 10. Hands-On Exercise

1. Install: Pipeline, Git, JUnit, Slack Notification, SonarQube Scanner (if you have one running), Timestamper, AnsiColor.
2. Configure a Slack workspace integration.
3. Take a small Java/Maven repo (or Node.js / Python) and create a pipeline that:
   - Builds.
   - Runs tests and publishes JUnit results.
   - Sends a Slack message on success and failure.
4. Verify the build page shows the test trend graph after a couple of runs.

## 11. Knowledge Check
1. Where do installed plugins live on disk?
2. How do you roll a plugin back to an older version manually?
3. Which plugin would you install to publish an HTML coverage report?
4. What's the difference between the Mailer and email-ext plugins?
5. Why is "audit installed plugins" a quarterly hygiene task?

## What's Next
**Module 07** introduces Pipeline as Code — defining your build in a `Jenkinsfile` checked into Git.
