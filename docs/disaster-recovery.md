# Disaster Recovery Plan

This covers the Talos and Proxmox homelab cluster and the services it relies on (OpenBao, MinIO on TrueNAS, and the Terraform state). I also touch on personal data backups on TrueNAS since the cluster backups share that same box.

The targets in the table below are just goals for now. I'll fill in the measured column as I actually run the drills, otherwise it stays "not measured".

For external alerting, I have Uptime Kuma running on an external node (the QDevice) that pings Discord via a webhook if the cluster state drops.

---

## 1. Principles

1. Rebuild the cluster and restore the data. Terraform and Git describe the whole cluster, while anything I can't afford to lose lives outside it - databases in S3, secrets in OpenBao, and personal files on TrueNAS.
2. Order matters. Restoring layers out of sequence breaks dependencies, so follow the order in section 3.
3. An untested backup is just a guess. Every layer gets drilled, and every drill gets logged at the bottom of this doc.
4. Be honest about what isn't covered yet, which is all listed in section 7.

---

## 2. Where state lives

| State | Lives in | Protected by | Recovery method |
| :--- | :--- | :--- | :--- |
| Kubernetes manifests, ArgoCD config | GitHub repo | GitHub/Local copy to Forgejo | `terraform apply`, then ArgoCD reconciles |
| VM + Talos + bootstrap definitions | `terraform/` + state on NAS | Git + NAS snapshots/replication | `terraform apply` (if state is lost, see S7) |
| Talos secrets (cluster PKI) | Terraform state | NAS (unencrypted, see limitations) | Regenerated on a fresh rebuild |
| etcd | Control-plane VMs | Not backed up, by design (see section 6) | Rebuild |
| PostgreSQL databases | Node-local disks + S3 (base backup + WAL) | CNPG replicas + MinIO on TrueNAS | CNPG recovery (S5) |
| Application secrets | OpenBao VM | PBS VM backup + cron Raft snapshot | Restore VM or snapshot (S6) |
| Immich photos | NFS on TrueNAS | ZFS snapshots, off-site NAS, B2 | TrueNAS restore (S8) |
| Prometheus data | Cluster | Not backed up | Accepted loss |

---

## 3. Recovery order

Restore in this exact order since each step depends on the one before it.

1. Network, DNS, and the Proxmox hosts
2. TrueNAS (MinIO, NFS, and where the Terraform state sits)
3. OpenBao (needs to be running and unsealed)
4. Terraform state available
5. Cluster (`terraform apply`)
6. ArgoCD → operators (ESO, CNPG) → secrets → databases → apps

---

## 4. Recovery objectives (targets)

| Scenario | RPO target | RTO target | Measured |
| :--- | :--- | :--- | :--- |
| Single worker or control-plane VM lost | 0 | Minutes | not measured |
| Database lost or corrupted | Bounded by WAL archiving (minutes) | Time to restore base backup + replay WAL | not measured |
| Full cluster rebuild | 0 for config (Git); DB per above | Set after drill 1 | not measured |
| OpenBao lost | Last Raft snapshot | Time to restore VM + unseal | not measured |
| Proxmox host with 2 control planes lost | As full rebuild | As full rebuild | not measured |

I left the RTO targets blank on purpose. Drill 1 sets the baseline by measuring from the first `terraform apply` until all ArgoCD apps are healthy and the databases are back, and I'll record it here.

---

## 5. Runbooks

### S1. Single worker lost

1. Run `kubectl get nodes` to confirm which node is `NotReady`.
2. Re-run `terraform apply` to recreate the VM and join it back to the cluster.
3. Check that pods rescheduled. For CNPG, run `kubectl get cluster -A` and make sure a replica got promoted if the primary was sitting on that dead node.

### S2. Single control plane lost

1. From a healthy control plane, run `talosctl -n <healthy-ip> etcd members`.
2. Remove the dead member with `talosctl -n <healthy-ip> etcd remove-member <member-id>`.
3. Run `terraform apply` to spin up a new VM, then confirm it rejoins and `etcd members` shows three again.

### S3. Proxmox host with two control planes lost

The talos-ctl-2 VM, is "HA" within Proxmox so it will move over to the other healthy Proxmox node.

### S4. Full cluster loss (rebuild)

Before starting, make sure MinIO is reachable, OpenBao is running and unsealed, the Terraform state is readable, and the repo is up to date.

1. Commit the `*.restore` variant (or the restore overlay once I build it) for every database before the first sync, otherwise they all start empty with `initdb`.
2. Set a new `serverName` for each recovered cluster's own backups (see S5).
3. Run `cd terraform && terraform apply`.
4. Create the `openbao-approle` secret manually using the bootstrap steps in the README.
5. Watch ArgoCD with `kubectl -n argocd-system get applications` and wait until everything goes Healthy.
6. Verify that `kubectl get externalsecret -A` shows everything `SecretSynced`, `kubectl get cluster -A` shows healthy databases, and spot-check the apps (including making sure Immich photos actually load).
7. Write down the start time, end time, and any hiccups in the drill log.

### S5. Database lost or corrupted

1. Swap the database manifest for its `*.restore` variant (`bootstrap.recovery`, pointing to the S3 `externalClusters` entry).
2. Give the recovered cluster a new `serverName` for its own backups, and keep the old `serverName` only under `externalClusters`. If a recovered cluster archives back into the old path, it can clash with or overwrite the history it just recovered from (still need to verify this exact behaviour in a drill with my CNPG version).
3. If you need point-in-time recovery, add `recoveryTarget.targetTime` under `bootstrap.recovery`.
4. Commit, push, and let ArgoCD sync. Make sure the cluster reaches `Cluster in healthy state` and the app reconnects.
5. Re-point app secrets or credentials if the credential-sync Job needs to run again (wave 3).

### S6. OpenBao lost

1. Restore the OpenBao VM from Proxmox Backup Server, or restore the latest Raft snapshot onto a fresh instance using `bao operator raft snapshot restore <file>`.
2. Unseal it using the unseal keys stored offline (see section 7).
3. Check that the `eso` AppRole and `eso-read` policy are there. If not, run through the bootstrap commands from the README again.
4. Check ESO with `kubectl get clustersecretstore` (should be `Valid`) and `kubectl get externalsecret -A` (should be `SecretSynced`).

### S7. Terraform state lost

The state file lives on the NAS, so try restoring it from a NAS snapshot or replica first. If it's completely gone, don't waste time trying to `import` everything - just tear down whatever is left and do a full rebuild (S4), which will generate fresh Talos secrets anyway.

### S8. TrueNAS lost (MinIO, NFS, state)

1. Restore from the off-site NAS replica (or Backblaze B2 for personal files).
2. Spin the MinIO container back up pointing at the restored dataset and check the `database-backup` bucket.
3. Put the Terraform state back in its path.
4. Keep in mind I haven't tested an off-site NAS restore yet, and I still need to confirm whether the MinIO zvol actually makes it into the off-site and cloud copies (see section 7).

### S9. Site loss

Personal files come back from the off-site NAS and Backblaze B2. For the cluster, I'd need new Proxmox hardware and then run through S4, though that only works if the OpenBao backup and Terraform state are also sitting off-site—which isn't confirmed yet.

---

## 6. Decision: etcd is not backed up

Nothing in etcd is irreplaceable. Manifests live in Git, databases live in S3, and secrets live in OpenBao. Rebuilding is a path I've actually tested, and it saves me from maintaining a second restore workflow.

The trade-off is that losing quorum means doing a full rebuild, which takes longer than restoring a snapshot, and anything created manually in the cluster without being committed to Git is gone, but nothing should ever be manually added to the cluster as this is a GitOps first approach and any cluster changes happen through Git.

I'll revisit this if a full rebuild takes longer than I can live with (drill 1 will show me), or if I start running stateful workloads whose cluster objects aren't in Git. Adding a scheduled `talosctl -n <ip> etcd snapshot <file>` copied to MinIO would be easy enough if I ever need it.

---

## 7. Known gaps

| Gap | Risk | Action |
| :--- | :--- | :--- |
| Off-site NAS restore never tested | Replica might not restore as expected | Drill 3 |
| Unseal keys / root token storage | Cannot unseal after restore | Confirm they are stored offline in two places |
| Terraform state unencrypted | Secret exposure | Move to OpenTofu state encryption |
| Restore variants are swapped by hand | Slow and error-prone during a rebuild | Kustomize `restore` overlay |

---

## 8. Drill program

| # | Drill | Frequency | Status |
| :-: | :--- | :--- | :--- |
| 1 | Full cluster rebuild + all DBs restored, timed | Every 6 months | Done Nov 22 |
| 2 | Single DB point-in-time restore | Every 3 months | Not done |
| 3 | TrueNAS off-site replica restore | Every 6 months | Not done |
| 4 | OpenBao restore from PBS and from Raft snapshot | Every 6 months | Not done |
| 5 | Proxmox host loss simulation (power off one host) | Yearly | Done, not recorded |

### Drill log

| Date | Drill # | Start → end | Result | Problems found | Fixes |
| :--- | :-: | :--- | :--- | :--- | :--- |
| Oct 2026 | 1 | not recorded | Cluster rebuilt, DBs recovered from S3 | No Data | N/A |
| Nov 19 2026 | 2 | 9:41 -> 10:51 | Cluster Rebuilt - fully deleted VM's and rebuilt, restored DB's from S3 | Outdated Talos image on Proxmox host, helm repo not updated on provisioning machine, ESO CRD's missing when trying to sync | Installed new Talos image, helm update, changed ArgoCD Sync wave for ESO CRDs and Kube-prom-stack CRDs |
| Nov 22 2026 | 2 | 17:10 -> 17:43 | Cluster Rebuilt - Clean rebuild, I had to manually sync the Monitoring apps again | Manually synced Monitoring "app" in ArgoCD, otherwise went really smoothly. | |