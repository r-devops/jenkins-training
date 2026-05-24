# Module 10 — Shared Libraries

## Learning Objectives
- Understand why Shared Libraries exist and when to use them.
- Build a library with the canonical `vars/`, `src/`, `resources/` layout.
- Load libraries implicitly, with `@Library`, and dynamically.
- Version libraries via Git tags.
- Test library code.
- Build organization-wide pipeline templates.

## 1. Why Shared Libraries?

When you copy the same 200 lines of Groovy into ten `Jenkinsfile`s, you have a problem:
- Bugs must be fixed in ten places.
- Pipelines drift apart over time.
- New repos can't get the latest improvements without manual updates.

**Shared Libraries** let you write that code once, in its own Git repository, and consume it from any pipeline. Updates to the library propagate.

## 2. Repository Layout

A library is a Git repo with this canonical structure:
```
my-pipeline-library/
├── vars/
│   ├── deployToK8s.groovy
│   ├── deployToK8s.txt
│   └── notifySlack.groovy
├── src/
│   └── com/
│       └── acme/
│           └── ci/
│               └── Build.groovy
├── resources/
│   └── com/
│       └── acme/
│           └── ci/
│               └── helm-values-template.yaml
└── README.md
```

- `vars/` — global variables and "steps" (one file = one step). The file `vars/foo.groovy` becomes the step `foo` in pipelines.
- `src/` — classic Java/Groovy classes, organized by package. Use when logic gets non-trivial.
- `resources/` — static files (templates, schemas) loadable via `libraryResource('path')`.

## 3. Configuring a Library on the Controller

**Manage Jenkins → System → Global Pipeline Libraries → Add**:
- **Name:** `acme-ci` (used to reference the library).
- **Default version:** `main` (or a release branch).
- **Source Code Management:** point at the library's Git repo + credentials.
- Tick **Load implicitly** if every pipeline in this Jenkins should auto-load the library.
- Tick **Allow default version to be overridden** so pipelines can pin a tag.

You can also configure libraries scoped to a **Folder** instead of globally — useful for multi-tenant Jenkins.

## 4. Loading the Library in a Pipeline

### Implicit (when "Load implicitly" is on)
The library's `vars/` are always available — no `@Library` annotation needed.

### Annotation
```groovy
@Library('acme-ci') _
pipeline { ... }
```
The trailing underscore is required syntax.

### Pin a version
```groovy
@Library('acme-ci@v1.4.0') _
```
Use a tag or branch. Pinning is good practice — uncontrolled drift breaks pipelines.

### Multiple libraries
```groovy
@Library(['acme-ci@v1.4.0','security-checks@main']) _
```

### Dynamic load (rare)
```groovy
library identifier: 'acme-ci@v1.4.0',
        retriever: modernSCM([$class: 'GitSCMSource',
                              remote: 'git@github.com:acme/pipeline-lib.git',
                              credentialsId: 'git'])
```

## 5. Writing a Step in `vars/`

`vars/notifySlack.groovy`:
```groovy
def call(Map args) {
  def channel = args.channel ?: '#builds'
  def color   = currentBuild.currentResult == 'SUCCESS' ? 'good' : 'danger'
  def msg     = args.message ?:
                "${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
  slackSend channel: channel, color: color, message: msg
}
```

In a pipeline:
```groovy
post { always { notifySlack channel: '#deploys' } }
```

Optional documentation in `vars/notifySlack.txt` — Markdown shown on the library help page.

### `call` vs other methods
`call` is the conventional entry point invoked when you write `notifySlack(...)` in a pipeline. You can also expose extra methods:
```groovy
def info(String msg)  { slackSend color: 'good',  message: msg }
def alert(String msg) { slackSend color: 'danger', message: msg }
```
Call as `notifySlack.alert("oh no")`.

## 6. Classes in `src/`

For logic that's too big for a single `vars/` file, write a class:

`src/com/acme/ci/Build.groovy`:
```groovy
package com.acme.ci

class Build implements Serializable {
  def steps           // the pipeline 'steps' object (sh, echo, etc.)
  String version

  Build(steps) { this.steps = steps }

  void compile() {
    steps.sh "mvn -B clean package -Dversion=${version}"
  }

  void publish(String repo) {
    steps.sh "mvn -B deploy -Drepo=${repo}"
  }
}
```

Use it from a `vars/` step:
```groovy
// vars/javaBuild.groovy
import com.acme.ci.Build

def call(Map args) {
  def b = new Build(this)
  b.version = args.version
  b.compile()
  if (args.publish) b.publish(args.repoUrl)
}
```

**Important:** classes used in pipelines must be `Serializable` because Jenkins serializes pipeline state at each step boundary.

## 7. Using `resources/`

```groovy
def template = libraryResource('com/acme/ci/helm-values-template.yaml')
def values = template
  .replace('@@APP@@', appName)
  .replace('@@VER@@', version)
writeFile file: 'values.yaml', text: values
```

Use for YAML, JSON templates, scripts, or any static asset your library needs.

## 8. Versioning

Treat the library like any other product:
- Tag releases (`v1.0.0`, `v1.1.0`, ...).
- Maintain a `CHANGELOG.md`.
- Use SemVer: bump major for breaking changes.
- Pipelines pin to a tag; you bump the pin when you want a new version.

Don't push breaking changes to `main` without a heads-up — every pipeline using `@Library('lib')` (no pin) reloads against `main` on the next build.

## 9. Testing Shared Libraries

Three layers:

### Static checks
- Run `groovyc` against `src/` to catch syntax errors.
- Lint with `npm-groovy-lint` or `CodeNarc`.
- Validate `vars/*.groovy` actually parses.

### Unit tests
For classes in `src/`, write Spock or JUnit tests:
```groovy
def "compile invokes mvn"() {
  given:
  def stepsMock = Mock(Steps)
  def b = new Build(stepsMock); b.version = '1.0'
  when:
  b.compile()
  then:
  1 * stepsMock.sh({ it.contains('mvn') })
}
```

### Pipeline-level tests
The **Jenkins Pipeline Unit Testing Framework** (`JenkinsPipelineUnit`) lets you exercise `vars/` steps against a mocked pipeline runtime. Use it for high-value steps like `deployToK8s`.

### Smoke pipeline
Maintain a "library smoke" repository whose only job is to consume every public step and run them against a dummy app. Run it on every library change.

## 10. Organization-Wide Pipeline Templates

The biggest win: define a *whole pipeline* as a step, so consuming pipelines are tiny.

`vars/javaService.groovy`:
```groovy
def call(Map cfg) {
  pipeline {
    agent { label 'linux && docker' }
    options { timeout(time: 30, unit: 'MINUTES'); timestamps() }
    stages {
      stage('build')   { steps { sh 'mvn -B verify' } }
      stage('image')   { steps { script { buildContainer cfg.appName } } }
      stage('publish') { when { branch 'main' }
                         steps { script { pushImage cfg.appName, env.BUILD_NUMBER } } }
    }
    post { always { junit '**/target/surefire-reports/*.xml' } }
  }
}
```

A consuming repo's `Jenkinsfile`:
```groovy
@Library('acme-ci@v2') _
javaService(appName: 'payments-api')
```

Three lines. The platform team owns the rest.

### Trade-offs
- Templates concentrate complexity in the library.
- They are great for uniformity, but make sure they're flexible enough for legitimate variation.
- Provide escape hatches (extension closures) for the unusual cases.

## 11. Security Notes

- Sandbox: by default, library code runs in the Groovy sandbox, which restricts what it can do. Code outside the sandbox needs script approval — minimize this.
- Treat the library repo as production code. PR review required. Branch protection on `main`.
- Pin tags in critical pipelines. Don't track `main` in production unless you're confident in the change-control process.

## 12. Hands-On Exercise

1. Create a new Git repo `pipeline-library`.
2. Add:
   - `vars/notifySlack.groovy` (from this module).
   - `vars/javaBuild.groovy` (calls `mvn` and publishes JUnit).
   - `vars/deploySsh.groovy` that takes `host` and `version` args and scp/ssh's.
3. Register it in Jenkins as a Global Pipeline Library named `acme-ci`, pinned to `main`.
4. Convert one of your earlier pipelines to use these steps. The resulting `Jenkinsfile` should be tiny.
5. Tag the library `v1.0.0`. In the pipeline change `@Library('acme-ci')` to `@Library('acme-ci@v1.0.0')`. Push a breaking change to `main` and confirm pinned pipelines are unaffected.

## 13. Knowledge Check
1. What's the difference between `vars/` and `src/`?
2. Why must classes used in pipelines be `Serializable`?
3. How do you pin a pipeline to a specific library version?
4. When would you use `libraryResource`?
5. What's one risk of consuming a library by branch name instead of tag?

## What's Next
**Module 11** covers JCasC — managing the Jenkins controller's configuration as code.
