# Module 10 — Scaling & Performance

## Learning Objectives
- Right-size the controller and agent Pods.
- Autoscale agent capacity with the cluster autoscaler or Karpenter.
- Use spot/preemptible nodes safely for builds.
- Tune concurrency, queues, and caches for throughput.
- Monitor and optimize cost.

## 1. Two Scaling Problems

There are two things to scale independently:

1. **Controller throughput** — how many agent Pods, jobs, and UI users it can handle.
2. **Agent capacity** — how many concurrent builds can actually run.

The controller is *vertical*: more CPU + memory for one Pod. The agents are *horizontal*: more Pods, possibly on more nodes.

## 2. Controller Sizing

Symptoms it's undersized:
- UI feels laggy.
- Builds spend > a few seconds in the queue when capacity is available.
- GC pauses visible in metrics.

Tune in this order:
1. **Memory** — raise the JVM heap (`-Xmx`) and the Pod memory limit. Start small (4Gi), grow as needed.
2. **CPU** — bump the request and limit. The controller schedules and dispatches; high concurrency wants 4–8 CPU.
3. **Disk** — make sure `JENKINS_HOME` PVC isn't full and isn't on slow storage.
4. **Plugin pruning** — many plugins polling = controller load. Audit and remove.

Helm values:
```yaml
controller:
  resources:
    requests: { cpu: "2", memory: "4Gi" }
    limits:   { cpu: "4", memory: "8Gi" }
  javaOpts: "-Xms3g -Xmx6g -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError"
```

Pre-load the JVM with `-Xms == -Xmx` to avoid heap growth pauses.

## 3. Agent Pod Sizing

Each agent has its own resources. Get them right per-language:

| Build type | jnlp + main tool | Typical limits |
|------------|------------------|----------------|
| Tiny script | jnlp + alpine | 200m CPU, 256Mi RAM |
| Node.js | jnlp + node | 1 CPU, 2 GiB |
| Java/Maven (small repo) | jnlp + maven | 2 CPU, 4 GiB |
| Java/Maven (monorepo) | jnlp + maven | 4 CPU, 8 GiB |
| Container build (Kaniko) | jnlp + maven + kaniko | 4 CPU, 8 GiB |

Always set both `requests` and `limits`. Without requests, the scheduler may overcommit; without limits, one runaway build steals from neighbors.

## 4. Cluster Autoscaler vs Karpenter

Pods sit in `Pending` when no node has room. Two ways to add nodes automatically:

### Cluster Autoscaler
- Works with managed node groups (EKS managed nodegroups, GKE node pools).
- Scales by adding/removing nodes from defined groups.
- Slow-ish (1–3 min per node).
- Predictable; fits well in regulated environments.

### Karpenter (AWS, increasingly multi-cloud)
- Provisions individual EC2 instances based on Pod requirements.
- Picks instance type per workload.
- Faster (seconds to a minute).
- More flexible spot mix.

Either way:
- Tag a "build" node pool/template that agent Pods request via `nodeSelector` and `tolerations`.
- The pool starts at 0 nodes and scales up only when builds queue.

### Spot / preemptible nodes
Builds tolerate restarts (much better than your customer-facing services). Run agents on spot.

Tolerations on the pod template:
```yaml
spec:
  tolerations:
    - key: workload
      value: ci
      operator: Equal
      effect: NoSchedule
    - key: spot
      value: "true"
      operator: Equal
      effect: NoSchedule
  nodeSelector:
    workload: ci
```

Spot interruption strategy:
- Jenkins retries the build on a new agent automatically when the Pod is killed.
- For critical stages (`deploy prod`) — route to on-demand agents via a different label.

## 5. Pod Startup Time

Slow agent startup steals user-perceived throughput.

### Common contributors
- Pulling large images.
- Cold node provisioning.
- Pulling many Pod containers in series.

### Mitigations
- **Pre-pulled images.** Run a DaemonSet that pulls common agent images on every node; or use image-pull policies to cache aggressively.
- **Smaller tool images.** Use distroless or minimal base images.
- **Fewer containers per Pod.** If you don't need Kaniko + Maven + Cosign + Trivy in one Pod, split into stages with different Pods.
- **Warm pool.** Configure the Kubernetes plugin to keep `idleMinutes` > 0 so warm agents stick around briefly between builds — good when build volume is bursty.

## 6. Concurrency Knobs

### Controller-side
- `numExecutors` on the controller itself: keep at **0**. Builds should never run on the controller.
- `containerCap` on the Kubernetes cloud: maximum agent Pods at once. A safety valve.
- Per-pod-template caps: limit a specific template to N concurrent.

### Job-side
- `options { disableConcurrentBuilds() }` per pipeline — for jobs that can't safely run twice in parallel (e.g., deploys to the same env).
- **Throttle Concurrent Builds** plugin or **Lockable Resources** for finer control (e.g., max 1 deploy to staging at a time).

### Queue health
A queue depth > executors briefly is fine. Sustained > 2× executor count for hours means you need more capacity.

## 7. Caches

Caches are the highest-ROI performance change.

### Per-build vs persistent
- Per-build cache (`emptyDir`) — slow first build, slow every build. Bad.
- Per-agent persistent cache (PVC on the same node) — fast, but fragile if Pod schedules elsewhere.
- Remote shared cache (S3-backed, sccache, BuildKit registry cache) — fast and resilient.

### Examples by ecosystem
- **Maven**: `~/.m2` volume; or Nexus/Artifactory pulls.
- **npm/yarn/pnpm**: lockfile-aware cache; pnpm with store path on PVC.
- **Gradle**: build cache server (`org.gradle.caching=true`) plus dependency cache.
- **Go**: `GOMODCACHE` and `GOCACHE` on a volume.
- **Container images**: Kaniko cache or BuildKit registry cache (see Module 07).

Pick **one** caching strategy per ecosystem and apply consistently.

## 8. Pipeline-Level Optimizations

These add up:

- Parallelize independent stages.
- Lightweight checkout (multibranch).
- Skip unchanged work (`changeSets`, path filters).
- Stash only what's needed.
- Use shallow Git clones for huge repos.
- Build smaller container images (smaller = faster push, faster pull next time).
- Cache `mvn dependency:go-offline` (or equivalent) early so later stages don't re-resolve.

## 9. Cost Monitoring

Treat CI as a cost center to manage.

### Tag everything
Label your agent node pool and namespace with a cost-center identifier. Cloud cost tools (CUR, BigQuery exports) can then split CI spend.

### Identify the heavy hitters
- Top jobs by CPU-hours.
- Top jobs by Pod-minutes.
- Builds that run frequently but produce no useful change (dead branches, abandoned PRs).

A small dashboard catches the 20% of jobs that consume 80% of capacity.

### Quick wins
- Tighten `numToKeepStr` (build history) — large histories = bigger PVC = bigger backups.
- Move long-running batch jobs to off-peak.
- Reduce e2e suite run frequency on long-lived branches.
- Right-size — the cluster runs on the *slowest* (largest) requested Pod, so trim oversized requests.

## 10. A Worked Example: Tuning Agent Throughput

Symptoms: queue depth spikes to 40 every morning at 9:00; agents take 3 minutes to start.

Steps taken:
1. **Add a build node pool with Karpenter** — provisioned on demand, scales to 0 at night. 5-minute autoscale latency drops to ~45s.
2. **Switch to spot** — same nominal capacity, 1/4 the cost.
3. **Pre-pull base images via a DaemonSet** — startup drops from 3 min to 50s.
4. **Split monolithic 8-container agent template into two stages** — small "build" Pod, separate "deploy" Pod. Memory request per agent drops from 8Gi to 3Gi → 2× more agents per node.
5. **Move Maven cache to a remote Nexus** — builds run 30% faster.

End result: morning queue clears by 9:10 instead of 10:30, cluster bill down 60%.

## 11. Hands-On Exercise

1. Install the cluster autoscaler (or Karpenter) on your cluster.
2. Add a `workload=ci` taint to a build node pool; configure your agent Pod template with the matching toleration and node selector.
3. Watch `kubectl get nodes -w` while a wave of builds kicks off — nodes should appear and disappear.
4. Implement a remote Maven cache via Nexus or via a PVC mounted into agent Pods. Measure: build times before and after.
5. Reduce one agent template from 6 containers to 3; measure: scheduling time before and after.

## 12. Knowledge Check
1. Why does the controller's `numExecutors` belong at 0?
2. What's the trade-off between Cluster Autoscaler and Karpenter?
3. When should a build run on on-demand vs spot instances?
4. Name three ways to reduce agent startup time.
5. What metric tells you you've finally over-provisioned?

## What's Next
**Module 11** covers observability and troubleshooting Jenkins on Kubernetes.
