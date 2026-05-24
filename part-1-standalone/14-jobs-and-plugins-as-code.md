# Module 14 — Jobs & Plugins as Code

## Learning Objectives
- Generate jobs programmatically with the Job DSL plugin.
- Use seed jobs to create and update jobs from Git.
- Manage installed plugins as a versioned list.
- Bootstrap an entire Jenkins controller — config, plugins, jobs — from Git.

## 1. Why Jobs as Code?

JCasC (Module 13) configures the controller. But the *jobs* themselves still live in `JENKINS_HOME/jobs/`. If you click "New Item" in the UI to create a job, that job is invisible to Git.

There are two ways to manage jobs as code:
1. **Job DSL plugin** — write Groovy that produces job definitions.
2. **Multibranch / Organization Folder + `Jenkinsfile`** — jobs auto-create themselves from `Jenkinsfile`s in your repos.

In practice you use **both**:
- A handful of *seed* and *administrative* jobs are managed by Job DSL.
- Application pipelines live as `Jenkinsfile`s consumed by Multibranch projects (which Job DSL itself can create).

## 2. Job DSL Plugin

Install: **Manage Jenkins → Plugins → Available → Job DSL → Install**.

### The pattern
1. Write Groovy scripts that declare jobs.
2. Create one **Seed job** in Jenkins of type "Freestyle" with a single build step: **Process Job DSLs**.
3. The seed job reads your scripts and creates/updates all the other jobs.
4. Run the seed job whenever the scripts change.

## 3. A Simple Job DSL Script

`seed/jobs.groovy`:
```groovy
folder('platform') {
  description('Jobs owned by the platform team')
}

pipelineJob('platform/build-tools') {
  description('Build internal CLI tools.')
  parameters {
    stringParam('TAG', 'latest', 'Image tag to publish')
  }
  triggers {
    cron('H 3 * * 1-5')
  }
  definition {
    cpsScm {
      scm {
        git {
          remote { url 'git@github.com:acme/tools.git'; credentials 'git' }
          branches('main')
        }
      }
      scriptPath('Jenkinsfile')
    }
  }
}

multibranchPipelineJob('platform/sample-app') {
  branchSources {
    github {
      id('sample-app')
      scanCredentialsId('github-pat')
      repoOwner('acme')
      repository('sample-app')
    }
  }
  triggers {
    periodicFolderTrigger { interval('1d') }
  }
}
```

The DSL is concise: a folder, a pipeline job pulling from Git, and a multibranch pipeline scanning a GitHub repo.

## 4. The Seed Job

Create once, manually:
1. **New Item → Freestyle → name `seed`.**
2. SCM: the Git repo holding your DSL scripts.
3. Build step: **Process Job DSLs → Look on Filesystem → DSL Scripts: `seed/*.groovy`**.
4. **Action for removed jobs:** `Delete` (or `Disable` to be safer at first).
5. **Action for removed views:** `Delete`.
6. Save → Build Now.

After the seed runs, the folder and child jobs exist. Subsequent runs reconcile state to match the scripts.

### Security
By default Job DSL scripts run in a sandbox. Some DSL features need approval the first time — review approvals carefully; an unsandboxed Job DSL script can do anything to your controller.

## 5. Generating Many Jobs from a List

This is where Job DSL shines.

```groovy
def services = ['payments-api', 'users-api', 'orders-api', 'shipping-api']

folder('services')

services.each { svc ->
  multibranchPipelineJob("services/${svc}") {
    branchSources {
      github {
        id(svc)
        scanCredentialsId('github-pat')
        repoOwner('acme')
        repository(svc)
      }
    }
  }
}
```

Add a new microservice? Add a string to the list, commit, run the seed job. No clicking.

## 6. Bootstrapping the Seed Job from JCasC

You can declare the seed job itself in JCasC, so a brand-new controller becomes fully populated automatically.

`jenkins.yaml`:
```yaml
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

`seed/Jenkinsfile`:
```groovy
pipeline {
  agent any
  stages {
    stage('process dsl') {
      steps {
        jobDsl targets: 'seed/*.groovy',
               removedJobAction: 'DELETE',
               removedViewAction: 'DELETE'
      }
    }
  }
}
```

Result: starting Jenkins applies JCasC → creates the seed job → seed job creates all other jobs from Git.

## 7. Plugins as Code

A list of installed plugins is part of your controller's identity. Manage it.

### `plugins.txt`
Plain text, one plugin ID and version per line:
```
git:5.2.1
workflow-aggregator:600.v4d3ed_2a_24f0e
configuration-as-code:1775.v810dc950b_514
job-dsl:1.87
matrix-auth:3.2.1
role-strategy:711.v3046bb_24f81d
slack:684.v833089650554
prometheus:2.5.0
```

### `install-plugins`
Modern Jenkins ships with a `jenkins-plugin-cli` tool. From a directory containing `plugins.txt`:
```bash
jenkins-plugin-cli --plugin-file plugins.txt --plugin-download-directory /var/jenkins_home/plugins
```

### Workflow
1. `plugins.txt` lives in your `jenkins-config` Git repo, next to `jenkins.yaml`.
2. To update: bump versions in `plugins.txt`, open a PR, get review.
3. CI builds/validates the controller image with the new plugins.
4. Rollout swaps the controller image.

### Tip: lock to specific versions
Avoid `latest`. Pinning avoids surprise breakages — even a minor plugin bump can change behavior.

## 8. Putting It Together — Bootstrap Pattern

The goal: anyone with credentials can rebuild your Jenkins controller in 10 minutes by running:
```
make controller
```

Repo layout:
```
jenkins-config/
├── jenkins.yaml           # JCasC
├── plugins.txt            # plugin manifest
├── seed/
│   └── jobs.groovy        # Job DSL: folders and jobs
├── Dockerfile             # bakes plugins, JCasC, and seed into the image
└── README.md
```

`Dockerfile`:
```dockerfile
FROM jenkins/jenkins:lts-jdk17

USER root
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

COPY jenkins.yaml /etc/jenkins/jenkins.yaml
COPY seed/ /etc/jenkins/seed/

ENV CASC_JENKINS_CONFIG=/etc/jenkins/jenkins.yaml
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

USER jenkins
```

Now: build the image, deploy it. The controller starts up fully configured. To change anything, change the file, commit, redeploy.

## 9. Reviewing the Spectrum

| What | How to manage as code |
|------|-----------------------|
| Controller settings | JCasC `jenkins.yaml` |
| Plugins | `plugins.txt` + `jenkins-plugin-cli` |
| Folders, seed jobs, admin jobs | Job DSL |
| Application pipelines | `Jenkinsfile` in each app repo, consumed by Multibranch |
| Shared logic | Shared Library (Module 12) |
| Build agents | JCasC (permanent) or Cloud configs |

## 10. Hands-On Exercise

1. Create a `jenkins-config` repo with `plugins.txt`, `jenkins.yaml`, and `seed/jobs.groovy`.
2. Put the Dockerfile from this module in the repo.
3. Build the image locally; run it; confirm the controller starts up with your plugins, config, and a seed job.
4. The seed job should automatically run and create a multibranch project pointing at a sample repo.
5. Delete the running container. Restart from the image. Confirm everything's still there.

## 11. Knowledge Check
1. What does the seed job do?
2. Why is "Action for removed jobs: Delete" a setting you should think about carefully?
3. What's the difference between JCasC and Job DSL?
4. How do you pin a plugin to a specific version?
5. What's the goal of the bootstrap pattern?

## What's Next
**Module 15** is the Part 1 capstone — build a complete, code-managed, standalone Jenkins from scratch.
