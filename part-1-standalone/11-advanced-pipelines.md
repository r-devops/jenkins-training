# Module 11 — Advanced Pipelines

## Learning Objectives
- Pause pipelines for human input and approvals.
- Use matrix builds to test across multiple dimensions.
- Apply retry, timeout, and structured error handling.
- Build reusable functions and patterns within a `Jenkinsfile`.
- Optimize pipelines for speed and executor usage.

## 1. The `input` Step

`input` pauses the pipeline until a user approves.

```groovy
stage('approve prod') {
  steps {
    input message: 'Deploy to production?',
          submitter: 'release-managers,jane.doe',
          ok: 'Promote'
  }
}
```

- `submitter` restricts who can approve (comma-separated users/groups).
- `ok` sets the button label.

### Release the executor while waiting
A naive `input` holds an executor — wasteful if approvals take hours.

Use this idiom to free the executor:
```groovy
stage('approve prod') {
  agent none                              // no executor for this stage
  steps {
    input message: 'Deploy to production?', submitter: 'release-managers'
  }
}
stage('deploy prod') {
  agent { label 'deployer' }
  steps { sh './deploy-prod.sh' }
}
```

### Collecting input values
```groovy
def reply = input(
  message: 'Approve and choose strategy',
  parameters: [
    choice(name: 'STRATEGY', choices: ['rolling','blue-green'], description: ''),
    string(name: 'TICKET',  description: 'Change ticket ID')
  ]
)
echo "Strategy: ${reply.STRATEGY}, ticket: ${reply.TICKET}"
```

Note: when only one parameter is collected, `input(...)` returns the value directly (not a map).

## 2. `matrix` — Build Across Dimensions

When you need to test the same code on multiple JDKs, OSes, or browsers:

```groovy
pipeline {
  agent none
  stages {
    stage('test all') {
      matrix {
        axes {
          axis { name 'OS';   values 'linux', 'windows' }
          axis { name 'JDK';  values '17', '21' }
        }
        excludes {
          exclude {
            axis { name 'OS';  values 'windows' }
            axis { name 'JDK'; values '21' }
          }
        }
        agent { label "${OS}" }
        stages {
          stage('test') {
            steps { sh "./test.sh --jdk ${JDK}" }
          }
        }
      }
    }
  }
}
```

Each cell runs in parallel on a matching agent. Use `excludes` to skip impossible combinations.

## 3. `retry`

Wrap flaky steps:
```groovy
retry(3) {
  sh 'flaky-network-call.sh'
}
```
The step runs up to 3 times; succeeds on first non-zero exit, fails on the last.

For exponential backoff between attempts, use a small `script` block with `sleep`.

## 4. `timeout`

Use generously. Stages should not run forever.

```groovy
stage('integration') {
  options { timeout(time: 20, unit: 'MINUTES') }
  steps { sh './integration-tests.sh' }
}
```

Pipeline-level timeout (Module 10):
```groovy
options { timeout(time: 1, unit: 'HOURS') }
```

## 5. Structured Error Handling

### `try/catch` for cleanup
```groovy
stage('test') {
  steps {
    script {
      try {
        sh './run-tests.sh'
      } catch (err) {
        currentBuild.result = 'UNSTABLE'
        emailext to: 'team@example.com', subject: 'Tests unstable', body: "${err}"
      } finally {
        junit '**/test-results/*.xml'
      }
    }
  }
}
```

### `catchError`
Declarative shorthand:
```groovy
catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
  sh './optional-step.sh'
}
```

### `error`
Fail the pipeline with a clear message:
```groovy
if (versionMismatch) {
  error "Version in pom.xml (${pom}) does not match Git tag (${tag})"
}
```

## 6. `lock` and `milestone`

### Lock — serialize critical regions
With the **Lockable Resources** plugin:
```groovy
stage('deploy staging') {
  steps {
    lock(resource: 'staging-environment') {
      sh './deploy-staging.sh'
      sh './integration-tests.sh'
    }
  }
}
```
Only one build can hold `staging-environment` at a time.

### Milestone — abort older builds
```groovy
milestone label: 'pre-deploy', ordinal: 1
stage('deploy') { ... }
```
If build #5 reaches milestone 1 before #4 does, #4 is aborted automatically. Combine with `disableConcurrentBuilds()` for stronger guarantees.

## 7. Reusable Functions in a `Jenkinsfile`

You can define plain Groovy functions:
```groovy
def deployTo(String env, String version) {
  echo "Deploying ${version} to ${env}"
  sh "./deploy.sh --env ${env} --version ${version}"
}

pipeline {
  agent any
  stages {
    stage('staging') { steps { script { deployTo('staging', env.VERSION) } } }
    stage('prod')    { steps { script { deployTo('prod',    env.VERSION) } } }
  }
}
```

When functions are used across many pipelines, promote them to a Shared Library (Module 12).

## 8. Pipeline Performance Tips

### Cheap wins
- **Don't checkout twice.** `checkout scm` once in an early stage and stash; downstream stages unstash.
- **Use shallow clones** if history isn't needed: configure in the SCM behavior or the `checkout` step.
- **Cache dependencies** between builds via tool-managed caches or persistent agent volumes.
- **Run independent stages in parallel.**
- **Avoid `agent any` on every stage** — entering and leaving agents has overhead.

### Lightweight checkout
For multibranch pipelines, Jenkins can read `Jenkinsfile` without doing a full clone. Configure it in the multibranch project settings — saves time when many pipeline triggers fire.

### Skip work that didn't change
The **CI-skip** pattern uses `[skip ci]` in commit messages; check `currentBuild.changeSets` to bail out early when nothing relevant changed.

### Pin agents close to caches
Pipelines that need a Maven local repo, npm cache, or Docker layer cache run faster on the same agent each time. Use a label that ties the job to a small pool of agents.

### Measure
Install the **Pipeline Stage View** and **Pipeline Graph Analysis** plugins. Look at stage durations across recent builds — you'll see the obvious slow spots.

## 9. Side Effects to Avoid

- **Global state on agents.** Two builds on the same agent must not collide. Use unique work directories or Docker containers per build.
- **Hidden background processes.** A `nohup`-ed process started by a build keeps running after the build ends. Kill anything you start.
- **Long-running test reports inside Jenkins.** Push large artifacts to object storage; keep Jenkins lean.

## 10. A Larger Worked Example

```groovy
pipeline {
  agent none
  options {
    timeout(time: 1, unit: 'HOURS')
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }
  parameters {
    choice(name: 'TARGET', choices: ['staging','prod'], description: 'Where to deploy')
  }
  stages {
    stage('checkout') {
      agent { label 'linux' }
      steps {
        checkout scm
        stash name: 'src', includes: '**'
      }
    }

    stage('verify') {
      parallel {
        stage('build & unit') {
          agent { label 'linux && docker' }
          steps {
            unstash 'src'
            sh 'mvn -B verify'
            junit '**/target/surefire-reports/*.xml'
            stash name: 'app', includes: 'target/*.jar'
          }
        }
        stage('lint') {
          agent { label 'linux' }
          steps { unstash 'src'; sh 'mvn -B checkstyle:check' }
        }
        stage('security') {
          agent { label 'linux && docker' }
          steps { unstash 'src'; sh 'trivy fs --severity HIGH,CRITICAL --exit-code 1 .' }
        }
      }
    }

    stage('approve') {
      when { expression { params.TARGET == 'prod' } }
      steps {
        timeout(time: 2, unit: 'HOURS') {
          input message: 'Promote to production?', submitter: 'release-managers'
        }
      }
    }

    stage('deploy') {
      agent { label 'deployer' }
      steps {
        unstash 'app'
        lock(resource: "${params.TARGET}-env") {
          retry(2) {
            sh "./deploy.sh --env ${params.TARGET} target/*.jar"
          }
          retry(5) {
            sleep 6
            sh "curl -fsS https://${params.TARGET}.example.com/health"
          }
        }
      }
    }
  }

  post {
    failure { slackSend channel: '#builds', color: 'danger', message: "FAIL ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  }
}
```

## 11. Hands-On Exercise

1. Take your Module 10 pipeline.
2. Add a `matrix` stage that runs tests against JDK 17 and JDK 21.
3. Add an `input` step before "deploy", configured with `agent none` so it doesn't hold an executor.
4. Wrap the deploy in `lock` and `retry`.
5. Compare stage durations before and after parallelizing — measure the gain.

## 12. Knowledge Check
1. Why use `agent none` around an `input` step?
2. What does `matrix` do that nested `parallel` cannot do as concisely?
3. When does `catchError` vs `try/catch` make sense?
4. What does the **Lockable Resources** plugin solve?
5. Name three concrete pipeline performance optimizations.

## What's Next
**Module 12** covers Shared Libraries — the way to share pipeline code across many projects.
