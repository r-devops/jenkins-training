# Module 11 — Observability & Troubleshooting

## Learning Objectives
- Collect logs from controller, agent Pods, and application pods.
- Scrape metrics from Jenkins and visualize them in Grafana.
- Trace slow pipelines and diagnose stuck builds.
- Diagnose common Kubernetes-side failures (image pulls, OOMKills, PVC contention).

## 1. The Three Signals

| Signal | What to use | What it tells you |
|--------|-------------|-------------------|
| Logs | Loki, EFK, cloud-native | What happened. |
| Metrics | Prometheus + Grafana | How much, how often, how slow. |
| Traces | OpenTelemetry, Tempo, Jaeger | Where the time went in a single build. |

You can ship without traces; you cannot ship without logs and metrics.

## 2. Logs

### Controller logs
The controller's stdout is collected by your cluster's logging stack just like any other Pod:
```bash
kubectl -n jenkins logs jenkins-0 -c jenkins -f
```

For long-term retention, route to Loki/Elastic/CloudWatch via Fluent Bit, Vector, or the cloud's built-in.

### Agent Pod logs
Each build creates an agent Pod whose logs disappear when the Pod is deleted. Two strategies:

1. **Stream live** — `kubectl -n jenkins-builds logs <pod> -c <container> -f` while a build runs.
2. **Persist** — set `podRetention: onFailure` on the cloud config so failed Pods stick around. Their logs are then accessible via Pod log forwarding to your central log store.

Most teams forward all Pod logs to Loki/Elastic and search by labels — `app=jenkins-agent`, `build_id=...`.

### Pipeline log
Jenkins keeps the build console log in `JENKINS_HOME/jobs/<job>/builds/<n>/log`. Keep build retention reasonable (`buildDiscarder` per job).

### Best practices
- Structured logs from your pipelines (JSON via `jq`, or a logging helper in your shared library).
- Tag every log line with the build's URL, repo, and branch — searchable.
- Don't echo secrets; Jenkins masks credentials, but only those bound via `withCredentials`.

## 3. Metrics from Jenkins

The **Prometheus** plugin (installed in Module 03) exposes metrics at `/prometheus`.

Scrape config (in your `prometheus.yml` or via a PodMonitor/ServiceMonitor in a Prometheus Operator setup):
```yaml
- job_name: jenkins
  scrape_interval: 30s
  metrics_path: /prometheus/
  static_configs:
    - targets: ['jenkins.jenkins.svc.cluster.local:8080']
```

With Prometheus Operator (kube-prometheus-stack):
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: jenkins, namespace: jenkins }
spec:
  selector: { matchLabels: { app.kubernetes.io/name: jenkins } }
  endpoints:
    - port: http
      interval: 30s
      path: /prometheus/
```

### Useful metrics
- `jenkins_queue_size_value` — pending builds.
- `jenkins_node_count_value` / `jenkins_node_online_value` — agent fleet health.
- `jenkins_executor_count_value` / `jenkins_executor_in_use_value` — utilization.
- `jenkins_builds_failure_build_count_total` / `..._success_..._total` — outcome counts.
- `jenkins_runs_duration_milliseconds` — build durations (histograms).
- `default_jvm_memory_used_bytes` — controller heap.

### Kubernetes-side metrics
Already in your Prometheus via kube-state-metrics and the kubelet:
- `kube_pod_status_phase` for agent Pods (Pending/Running/Failed).
- `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes` for resource usage.
- `kube_persistentvolumeclaim_resource_requests_storage_bytes` — PVC sizing.

## 4. Dashboards

Start with a community Jenkins Grafana dashboard, then build your own panels for:

1. **Capacity** — executor utilization, queue depth, agent count.
2. **Throughput** — builds/hour, average and p95 build duration.
3. **Quality** — failure rate, top-5 failing jobs.
4. **Controller health** — heap usage, GC time, thread count.
5. **Costs** — agent Pod-minutes per team/repo.

A useful "single pane" places: queue depth, controller heap %, agent count, top failing jobs.

## 5. Alerts

Page only on things that matter and require human action.

- **`jenkins_queue_size_value > 50` for 10m** — capacity issue.
- **Controller heap > 85% for 5m** — JVM trouble.
- **`up{job="jenkins"} == 0` for 2m** — controller down.
- **PVC > 80% used** — disk pressure.
- **Build failure rate > 25% in 1h on `main`** — broken release branch.

Suppress noisy alerts (canary tests that are *expected* to be flaky). Add a runbook link to every alert.

## 6. Traces (Optional, Powerful)

The **OpenTelemetry** plugin emits a trace per pipeline build with one span per stage.

Setup:
1. Install the OpenTelemetry plugin.
2. Configure an OTLP endpoint (Tempo, Jaeger, or vendor collectors).
3. Each build appears as a trace; you can drill into stage durations and see which stage in which build was slow.

Bonus: correlate pipeline spans with downstream service spans if your services emit OTel too — one trace from `git push` to "production deployed".

## 7. Troubleshooting Playbook

### Build stuck in queue
Symptoms: queue grows, no agent Pods scheduled.
- `kubectl -n jenkins-builds get pods` — any Pending? Why?
- `kubectl describe pod <pending-pod>` — node selector unsatisfiable? Image pull failure? RBAC?
- `kubectl get events -n jenkins-builds --sort-by=.lastTimestamp | tail -20` — recent failures.
- Cluster autoscaler logs — nodes failing to scale up? Quota?
- Check `containerCap` on the cloud config.

### Build hangs mid-run
- **Pipeline Steps** view: identify the running step.
- `kubectl -n jenkins-builds exec -it <pod> -c <container> -- sh` to peek at processes.
- Add `timeout` blocks to fail-fast.
- Check downstream services (registry, package mirror) for outages.

### Pod OOMKilled
- `kubectl describe pod <pod>` shows `OOMKilled` as the termination reason.
- Raise the container's memory limit.
- Or split the work into smaller steps.
- Watch for memory leaks in your build tools (Maven/Gradle can hoard).

### Controller restart loop
- `kubectl -n jenkins logs jenkins-0 -c jenkins --previous` — what crashed it?
- Out-of-memory? Raise heap; check for a bad plugin.
- Failed plugin install? Remove from `installPlugins` list and reinstall manually.
- Bad JCasC config? `kubectl -n jenkins get cm jenkins-jenkins -o yaml` to inspect.

### Slow controller UI
- `JENKINS_URL/threadDump` for thread state.
- Look for plugins doing heavy work synchronously.
- Heap dump if memory is suspect: `kubectl exec` + `jmap`.
- Generate a **Support Bundle** via the Support Core plugin.

### Agent fails to connect
- Pod is `Running` but Jenkins shows "agent went offline".
- Check the `jnlp` container's log: TLS error? URL wrong? Auth token expired?
- Confirm cluster DNS resolves `jenkins.jenkins.svc.cluster.local`.
- Confirm NetworkPolicy allows the egress to 8080/50000.

### Image pull failures
- `kubectl describe pod` → `Failed to pull image`.
- Wrong registry, missing `imagePullSecret`, expired credential, ImagePullBackOff.
- Verify `kubernetes.io/dockerconfigjson` Secret in the agent namespace.

### PVC contention / IOPS bottleneck
- `kubectl describe pvc` for the controller PVC.
- Cloud provider monitoring for the underlying disk — high latency on `JENKINS_HOME` makes everything slow.
- Move to gp3 / pd-balanced with provisioned IOPS, or move build caches off the controller PVC.

## 8. Health Endpoints to Hit

- `JENKINS_URL/manage/systemInfo` — Jenkins version, Java, OS, plugin counts.
- `JENKINS_URL/computer/api/json?depth=1` — node status JSON for monitoring.
- `JENKINS_URL/api/json?tree=jobs[name,color]` — quick all-jobs status.
- `JENKINS_URL/queue/api/json` — current queue.
- `JENKINS_URL/log/all` — internal logs.

Wrap these in your monitoring or in a small `kubectl exec` debug command.

## 9. Support Bundles

The **Support Core** plugin (often pre-installed) generates a zip containing logs, thread dumps, system info, and config. Use it when filing bugs upstream.

- **Manage Jenkins → Support** → generate. Download. Sanitize secrets before sharing externally.
- Programmatic: `curl -u user:token $JENKINS_URL/support`.

## 10. Hands-On Exercise

1. Install `kube-prometheus-stack` and `loki-stack` (or any logging operator).
2. Wire up the Jenkins Prometheus endpoint via ServiceMonitor.
3. Import a community Jenkins dashboard into Grafana. Add a custom panel for queue size.
4. Set up an alert on controller heap > 80% for 5m, with a Slack channel as the destination.
5. Deliberately trigger an OOMKill: write a pipeline that requests 256Mi memory and runs `stress --vm 1 --vm-bytes 512M`. Confirm the failure shows up in metrics and the logs identify the cause.

## 11. Knowledge Check
1. Where do agent Pod logs go after the build finishes?
2. What's the path Jenkins exposes Prometheus metrics on?
3. Name three metrics worth a Grafana panel.
4. How would you investigate "build hung at step 4 of stage 3"?
5. What does the Support Core plugin produce, and when do you use it?

## What's Next
**Module 12** covers backups, disaster recovery, and upgrades for the K8s controller.
