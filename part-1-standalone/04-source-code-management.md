# Module 04 — Source Code Management

## Learning Objectives
- Connect Jenkins to Git repositories on GitHub, GitLab, and Bitbucket.
- Configure credentials for SSH and HTTPS access.
- Set up webhooks for instant build triggering.
- Understand multibranch projects.

## 1. The Git Plugin

The **Git plugin** ships with the suggested plugin set. It adds a "Git" option under **Source Code Management** for every job, plus pipeline steps for checking out repos.

To confirm it's installed: **Manage Jenkins → Plugins → Installed → search "Git"**.

## 2. Adding Git to a Freestyle Job

In your job config:
1. **Source Code Management → Git**.
2. **Repository URL** — paste the clone URL:
   - HTTPS: `https://github.com/acme/sample-app.git`
   - SSH: `git@github.com:acme/sample-app.git`
3. **Credentials** — select the credential to use (we set these up below).
4. **Branches to build** — e.g. `*/main`, `*/release-*`, or a specific tag.
5. Optional advanced behaviors:
   - **Clean before checkout** — wipe local changes.
   - **Shallow clone** — `--depth 1` for faster clones on big repos.
   - **Sparse checkout** — only pull part of the tree.

When the job builds, Jenkins clones the repo into the workspace and exposes Git env vars (`GIT_COMMIT`, `GIT_BRANCH`, `GIT_URL`).

## 3. Credentials Management

Jenkins has a built-in credentials store. To add credentials:

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

### Credential types you'll use most
| Kind | Use for |
|------|---------|
| Username with password | HTTPS Git, basic auth APIs. |
| SSH Username with private key | SSH Git, SSH to agents/servers. |
| Secret text | API tokens, webhook secrets. |
| Secret file | Service account JSON, kubeconfig. |
| Username and password (separated) | Tools that take user + token. |

### Scopes
- **System** — only the Jenkins server itself can use them.
- **Global** — usable by jobs.
- **Folder** — limited to a folder of jobs (good for multi-team setups).

### IDs
Give each credential a stable ID like `github-deploy-key` or `nexus-publisher`. Your pipelines will reference these IDs by name.

## 4. HTTPS vs SSH

### HTTPS with a Personal Access Token (PAT)
- Username: your Git user (or `x-access-token` for GitHub apps).
- Password: a PAT with `repo` (read) scope.
- Easy through firewalls (only outbound 443 needed).

### SSH with a deploy key
- Generate a key pair: `ssh-keygen -t ed25519 -C "jenkins@example.com" -f jenkins-deploy`.
- Add the public key as a **deploy key** on the repo (read-only is usually enough).
- Add the private key as an **SSH Username with private key** credential in Jenkins.
- Requires outbound TCP 22 to the Git host.

For multiple repos, prefer a **GitHub App** or service account over personal credentials — when a person leaves, jobs keep working.

## 5. Webhooks

Polling Git wastes resources. Webhooks let the Git provider notify Jenkins on push.

### General flow
1. Jenkins exposes a webhook URL such as `https://jenkins.example.com/github-webhook/`.
2. You add this URL in the repo's webhook settings.
3. On push, the Git server POSTs a payload to that URL.
4. The Jenkins plugin parses the payload and triggers matching jobs.

### GitHub
Plugin: **GitHub plugin** (or **GitHub Branch Source** for multibranch).
- In Jenkins: **Manage Jenkins → System → GitHub → Add GitHub Server**.
- On the repo: **Settings → Webhooks → Add webhook**.
  - Payload URL: `https://jenkins.example.com/github-webhook/`
  - Content type: `application/json`
  - Events: "Just the push event" (or also pull requests).

In the job: **Build Triggers → GitHub hook trigger for GITScm polling**.

### GitLab
Plugin: **GitLab plugin**.
- Webhook URL: `https://jenkins.example.com/project/<job-name>`.
- Add a **GitLab API Token** credential in Jenkins.
- Enable **Build when a change is pushed to GitLab** in the job.

### Bitbucket
Plugin: **Bitbucket plugin** or **Bitbucket Branch Source**.
- Webhook URL: `https://jenkins.example.com/bitbucket-hook/`.

### Webhook security
- Always use HTTPS.
- Most providers support a **shared secret** — configure both the provider and the Jenkins plugin with the same secret so payloads can be verified.
- Restrict inbound traffic to the provider's documented IP ranges if you can.

## 6. Multibranch Pipelines (introduction)

A multibranch project automatically:
- Discovers every branch (and optionally every pull request) in a repo.
- Creates a sub-job for each one.
- Runs the `Jenkinsfile` from that branch.
- Removes sub-jobs when branches are deleted.

This is the modern default for team projects. We cover the configuration depth in Module 08 (Pipeline as Code), but it's worth knowing now that you typically wire SCM up *once*, at the multibranch level, instead of repeating it per branch.

## 7. Behaviors and Strategy Options

In the SCM section you can layer behaviors:
- **Sparse checkout paths** — only check out subdirectories you need.
- **Wipe out repository & force clone** — guarantee a clean state (slower).
- **Polling ignores commits in certain paths** — don't trigger on doc-only changes.
- **Custom user name/email** for any commits Jenkins makes back.

## 8. Hands-On Exercise

1. Create a small Git repo on GitHub with a `README.md` and a shell script that prints "hello".
2. In Jenkins, add an HTTPS PAT credential called `gh-readonly`.
3. Create a Freestyle job called `sample-app-build`:
   - SCM: Git, your repo, credential `gh-readonly`, branch `*/main`.
   - Build step: `bash hello.sh`.
4. Configure the GitHub webhook to hit your Jenkins URL.
5. Push a commit. Verify the build runs within seconds.
6. Inspect `GIT_COMMIT` in the console output.

## 9. Knowledge Check
1. What's the difference between Global and System credential scope?
2. Why prefer webhooks over SCM polling?
3. Which credential type would you use for an SSH deploy key?
4. What does a multibranch pipeline do that a freestyle Git job does not?
5. How do you verify that a webhook payload actually came from your Git server?

## What's Next
**Module 05** covers standalone agents — adding build nodes, distributing work, and managing labels and tools.
