# Module 05 — Standalone Agents (Build Nodes)

## Learning Objectives
- Explain why and when to use agents.
- Add permanent agents over SSH and via JNLP/inbound.
- Use labels to direct work to the right machine.
- Install per-agent tools (JDK, Maven, Node).
- Monitor agent health and clean up workspaces.

## 1. Why Use Agents?

On a small server you can run all builds on the **built-in node** (the controller itself). This works only for a while:
- Builds compete with the UI and scheduler for CPU and memory.
- A build that crashes the JVM takes Jenkins down with it.
- You can't mix OSes — the controller is one OS.

**Best practice:** keep the controller for orchestration. Run all real builds on agents.

## 2. Agent Concepts

- **Node** — any machine Jenkins can run work on (controller or agent).
- **Permanent agent** — a long-lived agent process on a specific machine.
- **Cloud / ephemeral agent** — agents created on demand (Docker, Kubernetes, EC2). Covered in Modules 09 and Part 2.
- **Executor** — a build slot on a node. An agent with 4 executors runs 4 builds at once.
- **Label** — a tag attached to one or more nodes. Jobs request a label; Jenkins assigns the build to a matching free executor.

## 3. Adding a Permanent SSH Agent

The most common pattern for Linux/macOS agents.

### Prerequisites on the agent host
- SSH access from the controller.
- A user account, e.g. `jenkins`, with a writable home directory.
- Java installed (matching the Jenkins requirement).
- Tools you'll need (Git, Maven, Docker, etc.).

### Steps
1. **Manage Jenkins → Nodes → New Node**.
2. **Node name:** `linux-agent-01`, choose **Permanent Agent**.
3. Configure:
   - **# of executors:** 2–4 (one per available CPU core is a reasonable start).
   - **Remote root directory:** e.g. `/home/jenkins/agent`.
   - **Labels:** `linux docker java17` — space-separated tags.
   - **Usage:** Use this node as much as possible (or "Only build jobs with label expressions matching this node" for dedicated agents).
   - **Launch method:** *Launch agents via SSH*.
     - Host, credentials, port.
     - **Host Key Verification Strategy:** "Known hosts file" or "Manually trusted key" (don't use "Non-verifying" in production).
4. Save → Jenkins SSHes in, copies `agent.jar`, and starts it.
5. On the Nodes page, the agent should turn green within a few seconds.

### Troubleshooting
- Check **Log** on the node page for SSH errors.
- Verify the `jenkins` user can `ssh` from the controller without a password prompt.
- Confirm Java is on the `PATH` of the agent user.

## 4. Adding an Inbound (JNLP) Agent

Use when:
- The controller cannot reach the agent (firewall, NAT, locked-down network).
- The agent is Windows and you don't want SSH.

The agent process starts on the agent host and *connects out* to the controller.

### Steps
1. Create the node, choose **Launch method: Launch agent by connecting it to the controller**.
2. Save → click the node → Jenkins shows a command like:
   ```
   java -jar agent.jar -url https://jenkins.example.com/ -secret <token> -name win-agent-01 -webSocket
   ```
3. On the agent host:
   - Install Java.
   - Download `agent.jar` from `<JENKINS_URL>/jnlpJars/agent.jar`.
   - Run the command above (typically as a Windows Service or systemd unit).
4. Make sure the controller's JNLP port (default 50000) or WebSocket is reachable. WebSocket mode (`-webSocket`) reuses the HTTPS port and avoids opening 50000.

### Running the agent as a Windows Service
- Easiest: use `nssm` or the Jenkins **Windows Service Installer**.
- Configure the service to log on as a user with the rights it needs.

## 5. Labels and Node Selection

Labels are the language jobs use to request a machine.

- A node has labels: `linux docker gpu`.
- A job declares a label expression: `linux && docker`.

Label expression syntax:
- `linux` — single label.
- `linux && docker` — both labels.
- `linux || mac` — either.
- `linux && !gpu` — linux but not gpu.

In a Freestyle job: **Restrict where this project can be run** → enter the expression.
In a pipeline: `agent { label 'linux && docker' }`.

### Strategy
- Use coarse labels for OS and arch (`linux`, `windows`, `arm64`).
- Use feature labels for tools (`docker`, `nodejs`, `maven`).
- Use team or purpose labels sparingly — they create coupling.

## 6. Tool Installation

Two approaches:

### A) Install tools on the agent OS (recommended for standalone)
- Use the OS package manager (`apt`, `dnf`, `choco`, `brew`).
- Bake everything into a VM image with Packer or Ansible.
- Stable, predictable; agents are interchangeable.

### B) Use Jenkins **Tool installations**
**Manage Jenkins → Tools** lets you declare named installations:
- JDK 17 (auto-downloaded)
- Maven 3.9.x
- Node 20.x

In a job, choose the named tool. Jenkins downloads it to the agent on first use. Useful when:
- Different jobs need different tool versions.
- You can't or don't want to manage agent images.

## 7. Executor Count

A starting rule of thumb:
- **Light builds (compile small projects):** executors = vCPU count.
- **Heavy builds (large compiles, integration tests):** executors = vCPU / 2.
- **I/O-bound builds:** can sometimes exceed vCPU count.

If executors are saturated for hours each day, add capacity — don't oversubscribe.

## 8. Workspace Management on Agents

Workspaces live under the agent's remote root directory. They can grow indefinitely.

### Cleanup options
- **Workspace Cleanup plugin** — adds a "Delete workspace before/after build" option to jobs.
- **Periodic agent cleanup** via cron on the agent host:
  ```bash
  find /home/jenkins/agent/workspace -mindepth 1 -maxdepth 1 \
      -type d -mtime +14 -exec rm -rf {} +
  ```
- **Per-job retention** in the job config (Build Discarder).

## 9. Monitoring Agent Health

**Manage Jenkins → Nodes** lists all nodes with:
- Up/down status.
- Free disk space, free swap, response time (when the **Monitoring** plugins are installed).
- Number of running and queued builds.

### Taking an agent offline
On the node page, click **Mark this node temporarily offline**. Running builds finish; no new builds get scheduled there. Useful for maintenance.

### Disconnecting cleanly
SSH agents reconnect automatically. For JNLP, restart the agent process. If an agent goes "dead", check:
- Disk full on the agent.
- Clock skew between controller and agent (TLS issues).
- Network drops; switch to WebSocket mode.

## 10. Hands-On Exercise

1. Provision a second Linux VM. Install Java, Git, and Maven.
2. Create a `jenkins` user; copy the controller's `jenkins` user public key to `~/.ssh/authorized_keys`.
3. Add an SSH credential in Jenkins for that user.
4. Add the VM as a permanent SSH agent named `linux-agent-01` with labels `linux maven`.
5. Modify your `sample-app-build` job to require label `linux && maven` and observe that it runs on the new agent.
6. Take the agent temporarily offline; trigger another build and watch it queue.

## 11. Knowledge Check
1. Why not run all builds on the controller?
2. What's the difference between SSH and JNLP launch methods?
3. Write a label expression for "linux, has docker, no gpu".
4. Where do agent workspaces live on disk?
5. How do you safely apply updates to an agent without losing in-flight builds?

## What's Next
**Module 06** explores the Plugin ecosystem and the key integrations every Jenkins admin should know.
