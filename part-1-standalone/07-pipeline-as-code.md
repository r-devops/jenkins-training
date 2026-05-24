# Module 07 — Pipeline as Code

## Learning Objectives
- Explain why Pipeline as Code is the modern Jenkins default.
- Write Declarative and Scripted pipelines.
- Use `agent`, `stages`, `steps`, `environment`, `parameters`, `when`, `post`.
- Run stages in parallel.
- Use credentials and stash/unstash safely in pipelines.

## 1. Why Pipeline as Code?

Freestyle jobs (Module 03) live in the Jenkins UI. They have problems:
- Not versioned with the code.
- Hard to review (no diffs).
- Easy to break by clicking the wrong checkbox.
- Don't move with the repo if you migrate Jenkins.

**Pipeline as Code** puts the definition into a file in your repo called `Jenkinsfile`. Jenkins reads it, builds based on it, and the file moves with the code.

Benefits:
- Pull-request review of pipeline changes.
- Same pipeline runs on every branch, with branch-specific behavior controlled by code.
- Easy to clone — copy `Jenkinsfile` to a new repo and you're 80% done.
- Disaster recovery — the pipeline isn't trapped inside one Jenkins instance.

## 2. Declarative vs Scripted

Jenkins has two pipeline syntaxes.

### Declarative (recommended)
Structured, opinionated. Easier to read; covers ~95% of needs.

```groovy
pipeline {
  agent any
  stages {
    stage('build')  { steps { sh 'mvn -B package' } }
    stage('test')   { steps { sh 'mvn -B test' } }
  }
}
```

### Scripted
Free-form Groovy. More powerful, more rope to hang yourself with.

```groovy
node {
  stage('build') { sh 'mvn -B package' }
  stage('test')  { sh 'mvn -B test'   }
}
```

**Start with Declarative.** Drop into a `script { ... }` block for the rare bit you can't express declaratively.

## 3. The `Jenkinsfile`

A `Jenkinsfile` is just a text file with Groovy. Put it at the repo root (the conventional name and location).

### Minimal Declarative pipeline
```groovy
pipeline {
  agent any                    // run on any available agent
  stages {
    stage('hello') {
      steps {
        echo 'Hello, world.'
        sh 'date'
      }
    }
  }
}
```

### Hooking it up in Jenkins
1. **New Item → Pipeline** (or, better, **Multibranch Pipeline**).
2. **Pipeline → Definition: Pipeline script from SCM**.
3. Repository URL, credentials, script path (`Jenkinsfile`).
4. Save → Build Now. Jenkins clones, reads `Jenkinsfile`, runs the stages.

A **Multibranch Pipeline** does this automatically for every branch that contains a `Jenkinsfile`.

## 4. `agent` — Choosing Where to Run

```groovy
agent any                         // any available executor
agent none                        // no global agent; each stage picks its own
agent { label 'linux && docker' } // node with matching labels
agent { docker { image 'maven:3.9-eclipse-temurin-17' } }  // run inside a container
```

When `agent` is at the top level, every stage runs on that node. With `agent none`, each `stage` must declare its own agent.

## 5. `stages` and `steps`

A pipeline has one or more stages; each stage has one or more steps.

```groovy
stages {
  stage('checkout') {
    steps { checkout scm }
  }
  stage('build') {
    steps { sh './build.sh' }
  }
  stage('test') {
    steps {
      sh './run-tests.sh'
      junit '**/test-results/*.xml'
    }
  }
}
```

`checkout scm` checks out the same revision Jenkins detected (works in multibranch).

## 6. `environment` — Variables for the Pipeline

```groovy
environment {
  APP_NAME = 'sample-app'
  VERSION  = "1.0.${env.BUILD_NUMBER}"
  REGISTRY = 'registry.example.com'
}
```

`environment` blocks can also bind credentials:
```groovy
environment {
  AWS = credentials('aws-deploy')  // creates AWS_USR and AWS_PSW
}
```

## 7. `parameters` — User Input

```groovy
parameters {
  string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
  choice(name: 'ENV',   choices: ['dev','staging','prod'], description: 'Target env')
  booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip the test stage')
}
```

The first build won't see them; subsequent builds will display a parameterized form. Access them as `params.BRANCH`, `params.ENV`, etc.

## 8. `when` — Conditional Stages

```groovy
stage('deploy prod') {
  when {
    allOf {
      branch 'main'
      expression { params.ENV == 'prod' }
    }
  }
  steps { sh './deploy-prod.sh' }
}
```

Other `when` conditions: `tag 'v*'`, `not { changeRequest() }`, `environment name: 'X', value: 'Y'`, `triggeredBy 'UserIdCause'`.

## 9. `parallel` — Run Stages Side by Side

```groovy
stage('verify') {
  parallel {
    stage('unit')        { steps { sh 'mvn test' } }
    stage('lint')        { steps { sh 'mvn checkstyle:check' } }
    stage('security')    { steps { sh 'trivy fs --severity HIGH .' } }
  }
}
```

Each parallel branch can use a different agent. Failure semantics: by default, all branches start, and if any fails the others are aborted (`failFast true` makes that immediate).

## 10. `post` — Always/Success/Failure Handlers

```groovy
post {
  always   { junit '**/target/surefire-reports/*.xml' }
  success  { slackSend channel: '#builds', color: 'good', message: "OK ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  failure  { slackSend channel: '#builds', color: 'danger', message: "FAIL ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  unstable { slackSend channel: '#builds', color: 'warning', message: "UNSTABLE ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  cleanup  { cleanWs() }
}
```

`post` blocks can live at the pipeline level and inside each stage.

## 11. Credentials in Pipelines

Use Jenkins-stored credentials inside a pipeline via `withCredentials` (the value is masked in the console log automatically):
```groovy
stage('publish') {
  steps {
    withCredentials([usernamePassword(credentialsId: 'nexus',
                                      usernameVariable: 'NEXUS_USR',
                                      passwordVariable: 'NEXUS_PSW')]) {
      sh 'mvn -B deploy -DnexusUser=$NEXUS_USR -DnexusPass=$NEXUS_PSW'
    }
  }
}
```

Or via `environment { CRED = credentials('id') }`.

## 12. `stash` / `unstash` — Moving Files Between Stages

Different stages can run on different agents. To share files:
```groovy
stage('build')  {
  agent { label 'linux' }
  steps {
    sh 'mvn -B package'
    stash name: 'app', includes: 'target/*.jar'
  }
}
stage('deploy') {
  agent { label 'deployer' }
  steps {
    unstash 'app'
    sh './deploy.sh target/*.jar'
  }
}
```

Stash is fast and convenient for small files. For large artifacts use an artifact repository.

## 13. A Realistic Example

```groovy
pipeline {
  agent { label 'linux && docker' }

  options {
    timeout(time: 30, unit: 'MINUTES')
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }

  parameters {
    booleanParam(name: 'PUBLISH', defaultValue: false, description: 'Publish artifact?')
  }

  environment {
    APP_NAME = 'sample-app'
    VERSION  = "1.0.${env.BUILD_NUMBER}"
  }

  stages {
    stage('checkout') {
      steps { checkout scm }
    }

    stage('build')    {
      steps { sh 'mvn -B clean package -DskipTests' }
    }

    stage('verify') {
      parallel {
        stage('unit')     { steps { sh 'mvn -B test' } }
        stage('lint')     { steps { sh 'mvn -B checkstyle:check' } }
        stage('security') { steps { sh 'trivy fs --severity HIGH,CRITICAL --exit-code 1 .' } }
      }
    }

    stage('publish') {
      when { allOf { branch 'main'; expression { params.PUBLISH } } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus',
                                          usernameVariable: 'NEXUS_USR',
                                          passwordVariable: 'NEXUS_PSW')]) {
          sh 'mvn -B deploy -DskipTests'
        }
      }
    }
  }

  post {
    always  { junit '**/target/surefire-reports/*.xml' }
    success { slackSend channel: '#builds', color: 'good',   message: "OK ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
    failure { slackSend channel: '#builds', color: 'danger', message: "FAIL ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  }
}
```

## 14. Tips

- Use **Pipeline Syntax** (link on the side of any pipeline job) — a snippet generator that builds the Groovy for any step.
- The **Replay** feature lets you run a modified version of a pipeline without committing — fast for iteration; never your final answer.
- Use `options { timeout(...) }` to keep runaway builds from hogging executors.
- Keep `Jenkinsfile` small; push complexity into shell scripts or shared libraries (Module 10).

## 15. Hands-On Exercise

1. Convert your Module 06 Freestyle job into a Declarative `Jenkinsfile` in the same repo.
2. Add: timestamps, build timeout, parallel test + lint, JUnit publishing, Slack on failure.
3. Hook it up via a Multibranch Pipeline that scans all branches.
4. Create a feature branch with a deliberately failing test. Verify only that branch goes red while `main` stays green.

## 16. Knowledge Check
1. Where does a `Jenkinsfile` live, and why?
2. What does `agent none` mean?
3. When would you use `parallel`?
4. What's the difference between `environment` and `parameters`?
5. How do you move build outputs from one stage's agent to another?

## What's Next
**Module 08** goes deep on advanced pipeline patterns — input, matrix, retries, error handling, optimization.
