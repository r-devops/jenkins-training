# Module 12 — Backup, DR & Upgrades

## Learning Objectives
- Back up the controller's PVC with Velero or volume snapshots.
- Restore the controller into a fresh cluster.
- Upgrade Jenkins core, plugins, and the Helm chart safely.
- Plan for multi-cluster / multi-region resilience.

## 1. What to Back Up

Three things matter:
1. **Configuration** — already in Git (`jenkins-config` repo).
2. **State** — the controller PVC: jobs, build history, credentials store, identity files.
3. **Secrets that aren't in Git** — Kubernetes Secrets that came from ESO/Sealed Secrets or were hand-applied.

If Git survives but state is lost, you can rebuild — except *build history* and *credential identity files* are gone. That's why state still needs backups.

## 2. Backup Strategies

### Volume snapshots (CSI)
Most cloud StorageClasses support `VolumeSnapshot` CRDs. They're fast (copy-on-write) and consistent enough for Jenkins if taken during a quiet moment.

Manual snapshot:
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: jenkins-home-2026-05-24
  namespace: jenkins
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: jenkins
```

Scheduled snapshots via a `CronJob` or via cloud-native tools (EBS lifecycle, GCP snapshot schedules).

### Velero
The cluster-aware backup tool of choice. Backs up:
- Cluster resources (Namespaces, Deployments, PVCs, Secrets, ConfigMaps).
- PV contents (via restic, kopia, or CSI snapshots).

```bash
velero backup create jenkins-daily \
  --include-namespaces jenkins,jenkins-builds \
  --snapshot-volumes \
  --ttl 720h
```

Schedule:
```bash
velero schedule create jenkins-daily \
  --schedule="0 2 * * *" \
  --include-namespaces jenkins,jenkins-builds \
  --ttl 720h
```

Velero stores to S3/GCS/Azure Blob. Off-cluster storage is non-negotiable for DR.

### Application-level
Optional belt-and-braces:
- Tar the JCasC ConfigMap contents (already in Git).
- Export credentials list (without values) for audit.
- Export job list and configs (`/api/xml`) — handy for forensic recovery if Velero fails.

## 3. Backup Hygiene

- **Daily full snapshot** at low-usage hour.
- **Off-cluster storage** with versioning.
- **Encryption at rest** for snapshot storage.
- **Lifecycle policy** to keep N days online + M months in cold storage.
- **Monthly restore drill** — proven, not theoretical.

## 4. Restore Procedure

Practice this until it's boring.

### From a Velero backup, same cluster
```bash
velero restore create restore-2026-05-24 --from-backup jenkins-daily-20260524020000
```

### From a Velero backup, fresh cluster
1. Install Velero in the new cluster pointing at the same object store.
2. Run the restore command above.
3. Reinstall the Helm chart (if Velero didn't capture Helm release secrets — check your config). Use the same values file.
4. Make sure the chart picks up the restored PVC (StatefulSet `volumeClaimTemplates` will reattach to the existing PVC name).

### From a volume snapshot
1. Provision a new PVC from the snapshot.
2. Install Jenkins with the chart, configured to use that PVC name.
3. Wait for startup.

### Sanity check after restore
- Controller starts and is reachable.
- A representative pipeline runs successfully (an idempotent "hello" job).
- Build history visible.
- Credentials decode correctly (a job using `withCredentials` should succeed).

If the controller starts but credentials decryption fails: the `secrets/master.key` and `secrets/hudson.util.Secret` files from `JENKINS_HOME` are critical. Make sure they were in the backup.

## 5. RTO and RPO

Make these explicit numbers:
- **RTO** — Recovery Time Objective: how long can you be without Jenkins?
- **RPO** — Recovery Point Objective: how much state loss is acceptable?

Common targets:
- Small team: RTO 4 h, RPO 24 h (nightly backup).
- Larger org: RTO 1 h, RPO 6 h (multiple snapshots/day, hot standby cluster).
- Compliance-driven: RTO 30 min, RPO 5 min (synchronous replication, automated failover).

Match backup frequency to RPO; match restore drills to RTO claims.

## 6. Upgrading Jenkins on Kubernetes

Three things to upgrade, ideally not all at once:
1. **Plugins.**
2. **Jenkins core image** (controller image tag).
3. **Helm chart version.**

### Plugins
- Edit `controller.installPlugins` (or your `plugins.txt` if you bake an image).
- Bump versions deliberately; never blanket-update.
- `helm upgrade` — the chart re-runs the plugin install. The controller restarts.
- Verify smoke tests post-upgrade.

### Core
- Bump `controller.tag` to a new LTS line.
- Test in non-prod first.
- `helm upgrade`.
- Plugin compatibility may force you to bump some plugins too.

### Chart
- Bump the `--version` flag.
- Read the chart's release notes — schema changes between major versions break values files.
- `helm diff upgrade` first (with the `helm-diff` plugin) to preview the manifest changes.

### Strategy
- Always have a recent PVC snapshot before any upgrade.
- Maintenance window: ~10 min downtime is typical.
- If you can run a parallel "canary" controller (same chart, new version, dummy data) for compat testing, do it.

### Rolling back
```bash
helm history jenkins -n jenkins
helm rollback jenkins <previous-revision> -n jenkins
```

The PVC survives — your data and plugin folder are intact. The rollback redeploys the previous controller image and ConfigMap.

If a plugin update is the problem: stop the controller, remove the offending `.jpi` from `JENKINS_HOME/plugins`, start. (Easier: roll the `installPlugins` list back and `helm upgrade`.)

## 7. Multi-Cluster Resilience

Options, increasing complexity:

### Cold standby
- Backups shipped to a second region.
- Second cluster ready but no Jenkins running.
- On disaster, run the restore procedure; switch DNS.
- RTO measured in hours.

### Warm standby
- A second cluster runs a Jenkins controller, periodically restored from snapshots.
- Read-only or paused.
- On disaster, promote it; switch DNS.
- RTO minutes-to-hours.

### Hot/hot — not really
Open-source Jenkins doesn't support active-active. Don't fake it.

## 8. Secrets Recovery

If you restore `JENKINS_HOME` but lose `secrets/master.key`, every credential becomes unreadable. Protect the master key:
- Back it up off-cluster, separately, encrypted.
- After a fresh install, copy the key in before the first start.

If you rely on JCasC + ESO for credentials, this matters less — the values are regenerated from Vault. But the controller's identity files still need to survive.

## 9. Disaster Drills

At least quarterly:
- Restore the latest Velero backup into a sandbox cluster.
- Time each phase: provisioning, restore, controller start, first build green.
- Record any manual step. The runbook should be exhaustive.
- Fix surprises before the next drill.

## 10. Upgrade Checklist

- [ ] Backup taken (Velero + PVC snapshot).
- [ ] Release notes read for chart, core, and any plugin major bumps.
- [ ] Smoke pipeline ready (a representative job that exercises common features).
- [ ] Maintenance window scheduled and announced.
- [ ] `helm diff upgrade` preview reviewed.
- [ ] `helm upgrade` executed.
- [ ] Pod restarts cleanly, log clean.
- [ ] Smoke pipeline green.
- [ ] A representative agent build green.
- [ ] Notifications still working (Slack, email).
- [ ] Backup *after* upgrade, before further changes.

## 11. Hands-On Exercise

1. Install Velero in your cluster, pointing at an S3 bucket (or MinIO for local tests).
2. Schedule a daily backup of `jenkins` and `jenkins-builds`.
3. Trigger one backup manually.
4. Restore it into a clean namespace (`jenkins-restore`). Confirm the controller starts and a representative job runs.
5. Document the procedure and time how long it took.
6. Run a Helm upgrade to a slightly newer chart version. Then `helm rollback`. Verify zero data loss.

## 12. Knowledge Check
1. What three things need backing up, and which can be regenerated from Git?
2. What does `helm rollback` do — and what does the PVC do during it?
3. What's the difference between RTO and RPO?
4. Why is `secrets/master.key` so important?
5. How would you upgrade plugins without breaking pipelines?

## What's Next
**Module 13** is about multi-team patterns on a shared Jenkins-on-K8s platform.
