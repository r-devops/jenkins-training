# Module 08 — Operations on a Standalone Server

## Learning Objectives
- Read and rotate Jenkins logs effectively.
- Implement a backup and restore strategy for `JENKINS_HOME`.
- Tune the JVM for the controller workload.
- Monitor Jenkins with Prometheus and Grafana.
- Upgrade Jenkins core and plugins safely.
- Diagnose stuck or failing builds.

## 1. Logs

### Where logs live
- Service-managed install (Linux): `journalctl -u jenkins`, and `/var/log/jenkins/jenkins.log`.
- Windows: `<install>\jenkins.err.log` / `jenkins.out.log`.
- Inside the application: **Manage Jenkins → System Log**.

### System Log
The System Log page lets you create named log recorders that tail specific loggers (e.g. `hudson.plugins.git`, `jenkins.security`). This is the right way to debug a plugin without flooding the main log.

### Log rotation
On Linux with the package install, `logrotate` is configured by default at `/etc/logrotate.d/jenkins`. Confirm:
```bash
cat /etc/logrotate.d/jenkins
```

For Jenkins-managed logs inside `JENKINS_HOME/logs/`, monitor size and clean old files via a cron job.

### What to look for
- `WARNING` / `SEVERE` lines on startup.
- `OutOfMemoryError` — heap too small or memory leak.
- `agent went offline` — networking or agent host issue.
- `Form submission denied` — CSRF crumb issue.

## 2. Backup Strategy

`JENKINS_HOME` is everything. Lose it and the controller is gone.

### What to back up
- `JENKINS_HOME` in its entirety **except**:
  - `workspace/` — re-creatable.
  - `caches/`, `tools/`, `updates/` — caches.
  - very large `jobs/*/builds/*` history if you must save space (you can prune).
- Off-host: ship backups to S3/Azure Blob/GCS or a separate volume.

### When to back up
- **Daily full backup** (overnight).
- **Hourly incremental** for high-change environments.
- Always run a backup before upgrading.

### How
Options, simplest first:
1. **Stop Jenkins, tar `JENKINS_HOME`, restart.** Works, but causes downtime.
2. **rsync while running** — generally safe; some files may be partly written. Run an integrity check after restoring.
3. **Filesystem snapshot** (LVM, EBS, ZFS) — fast and consistent. Best for production.
4. **ThinBackup plugin** — Jenkins-internal scheduled backups; OK for small installs.
5. **Velero / restic** for container-level installs.

### Restore drill
**Untested backups don't exist.** At least once a quarter:
1. Provision a clean VM.
2. Install the same Jenkins version.
3. Stop the service, replace `JENKINS_HOME` with the backup.
4. Start, log in, verify a few representative jobs run.

Document the procedure. Time it. If it takes 6 hours, that's your worst-case RTO; decide whether that's acceptable.

## 3. JVM Tuning

Controller stability often comes down to JVM settings.

### Heap
- Start at `-Xms2g -Xmx4g` for small installs; double until builds stop GC-thrashing.
- The controller should rarely sustain >75% heap utilization.
- Set `-Xms == -Xmx` to avoid runtime growth.

### Garbage collector
- Java 17/21 defaults to G1GC — fine for most controllers.
- For very large heaps (>16 GB), consider tuning G1 region size or evaluating ZGC.

### Useful flags
```
-Xms4g -Xmx4g
-XX:+UseG1GC
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/jenkins/heapdumps/
-Djava.awt.headless=true
-Dhudson.model.LoadStatistics.clock=2000
```

Set them in `/etc/default/jenkins` (Debian) or `/etc/sysconfig/jenkins` (RHEL).

### Symptoms of trouble
- Slow UI under load → heap pressure or thread starvation.
- Frequent restarts after `OutOfMemoryError` → bigger heap, capture heap dumps, hunt for the offending plugin.
- High CPU on the controller while idle → a plugin polling too aggressively.

## 4. Disk Management

`JENKINS_HOME` grows. Common offenders:

- `jobs/*/builds/` — old build records.
  - Use **Build Discarder** per job: keep last N builds or last N days.
- `workspace/` (controller workspaces).
  - Move builds to agents; clean controller workspaces periodically.
- `plugins/*.bak` from updates — safe to remove after a confirmed-good upgrade.

Set up alerting at 70% and 85% disk usage. Don't let `JENKINS_HOME` fill the disk — the JVM will start failing in opaque ways.

## 5. Monitoring with Prometheus + Grafana

### Expose metrics
Install the **Prometheus** plugin. It serves metrics at `JENKINS_URL/prometheus/`.

Add a scrape job in `prometheus.yml`:
```yaml
- job_name: jenkins
  metrics_path: /prometheus/
  static_configs:
    - targets: ['jenkins.example.com']
```

### Key metrics
- `jenkins_queue_size_value` — builds waiting.
- `jenkins_executor_count_value` / `jenkins_executor_in_use_value` — capacity vs utilization.
- `jenkins_builds_failure_build_count_total` — failure rates.
- `default_jvm_memory_used_bytes` — heap usage.
- `process_cpu_seconds_total` — CPU.

### Dashboards
Grafana has community dashboards (search for "Jenkins") that you can import as a starting point. Tune to your traffic.

### Alerts
- Queue >50 for 10 minutes.
- Controller heap > 80% for 5 minutes.
- Disk on `JENKINS_HOME` filesystem > 80%.
- Build failure rate > 25% in 1 hour (catches broken main branches).

## 6. Upgrading Jenkins

### Order of operations
1. Read release notes for the new LTS version — look for "incompatibility" and "deprecation".
2. Take a backup.
3. Note current core and plugin versions.
4. Update to the latest patch on your current LTS line first; confirm things still work.
5. Upgrade to the new LTS.
6. Update plugins in a single batch; restart once.
7. Run a smoke test: a representative pipeline, an agent build, a notification.

### When things go wrong
- Plugin compatibility: check `JENKINS_HOME/plugins/*.jpi.disabled` — Jenkins disables incompatible plugins on startup. Fix or roll back.
- A bad plugin update can crash startup. Move it out of `plugins/`, start Jenkins, install a compatible version.

### Plan a maintenance window
Even with rolling restart tricks, plan downtime. Communicate to users.

## 7. Troubleshooting Recipes

### "Build stuck in queue"
- Check the build's "Why is this build pending?" link.
- Common causes: no matching agent label, all executors busy, throttle plugin limits.

### "Build hangs forever"
- Click the build → **Pipeline Steps** → see which step is running.
- Use `timeout` in pipeline stages.
- If a process is wedged on the agent, SSH in and check `ps`, `top`.

### "Agent went offline"
- Node page → **Log**.
- Network reachability? Java still installed? Disk full?

### "Controller is slow"
- Take a thread dump: `JENKINS_URL/threadDump`.
- Take a heap dump if memory is suspect: `JENKINS_URL/headOnlineNodes/...` (or `jmap` against the JVM).
- Generate a **Support bundle**: install the **Support Core** plugin, **Manage Jenkins → Support** → generate. Attach to bug reports.

### "Plugin update broke a job"
- `JENKINS_HOME/plugins/<plugin>.hpi` — roll back to the previous version (you kept it, right?).

## 8. Hands-On Exercise

1. Configure log rotation for Jenkins on your test server. Verify a forced rotation works.
2. Set up a nightly backup of `JENKINS_HOME` to a separate disk; restore into a fresh VM to verify.
3. Install the Prometheus plugin; scrape with a local Prometheus; import a community Jenkins Grafana dashboard.
4. Generate a support bundle and inspect what's inside.
5. Simulate a stuck build: write a shell step that runs `sleep 600`. Configure a 5-minute timeout in the job and confirm Jenkins aborts the build.

## 9. Knowledge Check
1. What goes in `JENKINS_HOME` that you must back up, and what can you skip?
2. What's the recommended order: backup, plugin update, core update?
3. Name three Prometheus metrics worth alerting on.
4. How do you safely roll back a plugin update that broke jobs?
5. What's a support bundle and when do you produce one?

## What's Next
**Module 09** covers Deployment & Release workflows — using Jenkins to deliver software to servers and cloud targets.
