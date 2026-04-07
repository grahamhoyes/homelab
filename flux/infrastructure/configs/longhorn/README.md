# Longhorn Storage Configuration

## Drive Setup

Each k3s agent (worker) node VM has a dedicated 200GB virtual disk (`/dev/vdb`) for Longhorn, separate from the 50GB OS root disk. This disk is:

- Provisioned by the [Ansible playbooks](/ansible/tasks/create-vm.yaml) (disk size configured in [`k3s_agents.yaml`](/ansible/inventory/group_vars/k3s_agents.yaml))
- Formatted as XFS and mounted at `/mnt/longhorn-storage/` by [`k3s.yaml`](/ansible/k3s.yaml)
- Configured as Longhorn's data path in the [Helm release](../controllers/longhorn/release.yaml)

Both virtual disks reside on the host's NVMe SSD via Proxmox `local-lvm`. Control plane nodes do not run Longhorn storage.

## Storage Classes

### `longhorn` (default)

- 3 replicas across nodes
  - 3 replicas is the standard recommendation for HA - it ensures volumes remain available when individual nodes are taken offline for maintenance. Longhorn runs on the 4 k3s-worker VMs (spread across pve1-4), not on the control plane VMs.
- Best-effort data locality
- Used by all application PVCs (media stack, paperless-ngx, pihole, etc.)

### [`longhorn-local`](local-storageclass.yaml)

- 1 replica, strict-local data locality
- Used exclusively by CloudNative-PG database clusters
- CNPG [recommends against replicated storage](https://cloudnative-pg.io/docs/1.29/storage#block-storage-considerations-cephlonghorn) to avoid write amplification. Durability comes from WAL archiving and base backups to S3 (via the Barman cloud plugin) rather than storage-level replication

## Backups & Snapshots

### Longhorn Snapshots ([`snapshot-job.yaml`](snapshot-job.yaml))

- **Schedule:** Daily at midnight
- **Retention:** 21 snapshots (~3 weeks)
- Local to the Longhorn volume - fast rollback but no protection against disk/node loss

### Longhorn Backups ([`backup-job.yaml`](backup-job.yaml))

- **Schedule:** Weekly (Sunday at midnight)
- **Retention:** 18 backups (~4 months)
- **Target:** `nfs://ds920.hoyes.dev:/volume1/backups/longhorn`
- Full volume backups stored on my Synology DS920 NAS
- Note: the NFS share must have "Allow connections from non-privileged ports" enabled (see comment in [`release.yaml`](../../controllers/longhorn/release.yaml))

### Restoring a Longhorn Backup

Longhorn backups can be restored through the Longhorn UI or CLI. See:
- [Longhorn: Restoring Volumes from Backups](https://longhorn.io/docs/latest/snapshots-and-backups/backup-and-restore/restore-from-a-backup/)

### CloudNative-PG (Barman) Restores

CNPG databases are not restored via Longhorn. Instead, bootstrap a new CNPG Cluster from S3 backups using point-in-time recovery. See:
- [CloudNative-PG: Recovery](https://cloudnative-pg.io/docs/1.29/recovery)
- Example recovery config is commented out in [`cluster.yaml`](/flux/apps/hoyeslab/paperless/postgres/cluster.yaml)
