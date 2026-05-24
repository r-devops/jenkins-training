# Module 03 — Jenkins UI & Job Basics

## Learning Objectives
- Navigate the Jenkins dashboard and core menus.
- Create, configure, and run a Freestyle job.
- Configure build triggers (manual, cron, SCM polling, webhooks).
- Understand workspaces, artifacts, and the build history.

## 1. Dashboard Tour

After login you land on the dashboard. The key areas:

- **Left sidebar:** New Item, People, Build History, Manage Jenkins, My Views.
- **Center pane:** A list of jobs with status icons.
  - Weather icon — health over the last several builds.
  - Ball icon — last build status (blue=success, red=failed, yellow=unstable, grey=not built).
- **Build queue:** Jobs waiting for an executor.
- **Build executor status:** Live status of each executor on each node.

### Common menus
- **New Item** — create a new job.
- **People** — accounts known to Jenkins.
- **Build History** — recent builds across all jobs.
- **Manage Jenkins** — global config, plugins, users, nodes, system info.

## 2. Job Types

When you click **New Item**, Jenkins lists job types. The most useful:

| Type | When to use |
|------|-------------|
| Freestyle project | Simple, GUI-configured jobs. Good for learning. |
| Pipeline | Job defined as code in a `Jenkinsfile`. The modern default. |
| Multibranch Pipeline | Pipeline that automatically discovers branches in a Git repo. |
| Folder | A container for organizing jobs. |
| Organization Folder | Auto-discovers repos across a GitHub/GitLab org. |

We focus on **Freestyle** here for the fundamentals. Pipelines get a deep dive in Module 07.

## 3. Your First Freestyle Job

1. **New Item → Freestyle project → name it `hello-world` → OK**.
2. In the configuration page you'll see sections you can collapse:
   - General
   - Source Code Management
   - Build Triggers
   - Build Environment
   - Build Steps
   - Post-build Actions

### Add a build step
- **Build Steps → Add build step → Execute shell**
- Enter:
  ```bash
  echo "Hello from Jenkins on $(hostname)"
  date
  ```

### Save and run
- Click **Save**.
- Click **Build Now** in the left sidebar.
- Click the build number (e.g. `#1`) → **Console Output**.
- You should see your script's output.

## 4. Build Triggers

Triggers tell Jenkins *when* to run a job.

### Manual ("Build Now")
The default. A human clicks the button (or hits the API).

### Periodic build (cron syntax)
**Build Triggers → Build periodically**:
```
H/15 * * * *        # every ~15 minutes
0 2 * * *           # at 02:00 every night
H 3 * * 1-5         # at ~03:xx on weekdays
```
The `H` (hash) spreads jobs across the window so 50 jobs don't all fire at minute 0.

### Poll SCM
Jenkins periodically checks Git for new commits and starts a build if anything changed:
```
H/5 * * * *         # check every 5 minutes
```
Polling works without webhook access but is wasteful — prefer webhooks where possible.

### Webhook from Git provider
The recommended approach. The Git server (GitHub, GitLab, Bitbucket) calls Jenkins as soon as a push happens. Setup is covered in Module 04.

### Trigger by another job
**Build Triggers → Build after other projects are built** — runs this job whenever the upstream job completes.

### Remote trigger (API)
**Build Triggers → Trigger builds remotely** — gives you a token you can use to hit:
```
curl -X POST "http://jenkins/job/hello-world/build?token=MY_TOKEN"
```

## 5. Build Steps

Common build step types come from plugins. Built-in steps include:

- **Execute shell** (Linux/macOS) — runs a bash script.
- **Execute Windows batch command** — runs `cmd.exe` commands.
- **Invoke Ant / Gradle / Maven** — wraps build tools.
- **Conditional steps** — run only when an expression is true (with the Conditional BuildStep plugin).

A build step is just a command. You can chain many steps; if any step exits non-zero, the build fails.

### Environment variables available to your script
Jenkins exposes useful env vars:

| Variable | Meaning |
|----------|---------|
| `BUILD_NUMBER` | Sequential build number. |
| `BUILD_ID` | Same as `BUILD_NUMBER` on modern Jenkins. |
| `JOB_NAME` | Name of the job. |
| `BUILD_URL` | URL to this build. |
| `WORKSPACE` | Path to the workspace. |
| `NODE_NAME` | Name of the agent (or `built-in`). |
| `GIT_COMMIT` | SHA of the checked-out commit (if Git plugin used). |

## 6. Workspace

Every build runs inside a **workspace** — a directory on the agent where files live during the build.

- Path on a Linux agent: `<JENKINS_HOME>/workspace/<job-name>/` (for built-in node) or under the agent's root.
- Workspaces are reused across builds of the same job by default. This makes builds faster but can hide bugs caused by leftover files. You can configure "Clean workspace" before each build via the **Workspace Cleanup** plugin.
- You can browse the current workspace contents from the job page → **Workspace**.

## 7. Artifacts

An **artifact** is a file produced by a build that you want to keep — typically a JAR, a WAR, a tarball, a test report, or a binary.

### Archive artifacts
**Post-build Actions → Archive the artifacts**:
- Files to archive: `target/*.jar, build/distributions/*.tar.gz`
- Jenkins copies the matching files into the build record so they survive workspace cleanup.

Artifacts appear on the build page and can be downloaded via the UI or API.

### Fingerprinting
Optionally, Jenkins can fingerprint artifacts (compute an MD5 hash) so you can track which job and build produced a given file across the system.

## 8. Post-Build Actions

Run after the build steps complete. Common ones:

- **Archive the artifacts** — described above.
- **Publish JUnit test result report** — point at `**/surefire-reports/*.xml`. Jenkins parses results and shows pass/fail counts and trends.
- **Email notification** — send mail on failure.
- **Trigger downstream project** — start another job.

## 9. Build History & Console Output

For each job:
- **Build History** sidebar shows previous runs with timestamps.
- Click a build number to see details:
  - Status, duration, who/what triggered it.
  - Console Output — the full log.
  - Changes — Git changes since the previous build.
  - Workspace — file browser.
  - Artifacts — downloadable.

### Tips
- The Console Output is the first place to look when a build fails.
- Use **Pipeline Steps** (for pipelines) or the **Timestamper** plugin to add timestamps to log lines.
- The **Build Discarder** option (under General) lets you keep, e.g., only the last 20 builds to save disk space.

## 10. Hands-On Exercise

1. Create a Freestyle job named `count-files`.
2. Build step (Execute shell):
   ```bash
   echo "Run #$BUILD_NUMBER"
   ls -la
   echo "File count: $(ls | wc -l)" > file-count.txt
   ```
3. Post-build action: archive `file-count.txt`.
4. Configure it to run every 10 minutes via cron (`H/10 * * * *`).
5. Run it manually. Verify:
   - Console output shows the file listing.
   - `file-count.txt` is archived on the build page.
6. Wait 10 minutes — confirm it triggered automatically.

## 11. Knowledge Check
1. What's the difference between Build Now and a periodic trigger?
2. What does `H` mean in Jenkins cron syntax?
3. Where do artifacts live after a build is archived?
4. What's the difference between a workspace and an artifact?
5. Why might you choose webhooks over SCM polling?

## What's Next
**Module 04** covers Source Code Management — wiring Jenkins up to Git providers, using webhooks, and managing credentials.
