# Forgejo

This set of manifests deploys [Forgejo](https://forgejo.org/), a self-hosted git forge. It's reachable on my LAN at `https://code.hoyes.dev` (web) and `git@git.hoyes.dev:user/repo.git` (SSH clone). The instance is currently private only via the regular `nginx` ingress class, but it's designed so that exposing it publicly via a Cloudflare tunnel later requires minimal changes.

## Pre-requisites

This setup requires a few things available in the cluster:
- [CloudNative-PG](https://cloudnative-pg.io/) for managed Postgres ([my deployment](/flux/apps/hoyeslab/database/cloudnative-pg/)), along with the [barman-cloud plugin](/flux/apps/hoyeslab/database/cloudnative-pg/barman-cloud/) for S3 backups.
- [Dragonfly Operator](https://www.dragonflydb.io/) for a Redis-compatible cache ([my deployment](/flux/apps/hoyeslab/database/dragonfly/)).
- [MetalLB](https://metallb.io/) (or equivalent) for `LoadBalancer` services.
- [external-dns](/flux/infrastructure/controllers/external-dns/) configured to watch Services. This is what auto-creates the `git.hoyes.dev` A record from a Service annotation.
- [cert-manager](/flux/infrastructure/controllers/cert-manager/) with a `letsencrypt` ClusterIssuer for the wildcard `*.hoyes.dev` certificate.
- An NFS server with an SSD-backed share (see below).
- [1Password Operator](/flux/infrastructure/foundation/1password/) (or any way of getting an admin user secret into the cluster).

## The Setup

The `forgejo` namespace contains three logical pieces:

- **Postgres** (`postgres/`): a CloudNative-PG `Cluster` with 2 instances on `longhorn-local` (local SSD storage), backed up nightly to versitygw (S3) via the barman-cloud plugin. Postgres 18.
- **Dragonfly** (`dragonfly/`): a single-replica Redis-compatible cache, used by Forgejo for the cache, queue, and session backends.
- **Forgejo** (`forgejo/`): the application itself, deployed via the [official Helm chart](https://artifacthub.io/packages/helm/forgejo-helm/forgejo). The PV, PVC, and 1Password secret are managed alongside the `HelmRelease`.

The `kustomization.yaml` at the top of this directory pulls all three together, plus a namespace and a wildcard certificate.

## Storage

Forgejo's `/data` mount holds repositories, LFS objects, attachments, the auto-generated `app.ini`, and SSH host keys. We mount this from the NAS over NFS, specifically from a share which is backed by NVMe SSDs rather than HDDs.

The share is created on my DS920 at `/volume2/data-fast/forgejo`, owned by the `service-access` user (UID 1029, GID 100). NFSv4.1 is enabled in DSM. Squash is disabled, since the pod runs as 1029:100 anyway and we control the only client.

### Why a custom PV instead of an inline `nfs:` volume

Forgejo's documentation [recommends specific mount options for hosted repository data](https://forgejo.org/docs/next/admin/installation/docker/#hosting-repository-data-on-remote-storage-systems), which can't be set with a regular inline `nfs:` volume spec in the pod template:

```
hard,timeo=10,retry=10,vers=4.1
```

The `hard` flag in particular blocks all file operations if the share is unavailable, which prevents partial writes from corrupting repositories during a NAS hiccup. Inline `nfs:` volumes don't expose `mountOptions` to the user, so we use a `PersistentVolume` with an explicit `mountOptions` list and a pre-bound `claimRef`.

### Why `ReadWriteMany`

The PV is declared `ReadWriteMany` even though `replicaCount: 1`. This leaves the door open for running Forgejo in HA later without a storage migration. NFS supports concurrent access natively, so the access mode is just a scheduler hint and costs nothing to set.

Forgejo HA also needs an external database (we have it), Redis-backed queue/cache/session (we have it), and some attention to cron jobs that assume single-instance.

## Auto-Generating Signing Keys

Forgejo derives several signing keys on first install: `SECRET_KEY`, `INTERNAL_TOKEN`, `JWT_SECRET`, `LFS_JWT_SECRET`. These encrypt some database fields and sign JWTs for LFS and OAuth.

The Helm chart's [`config_environment.sh` init script](https://code.forgejo.org/forgejo-helm/forgejo-helm/src/tag/v17.1.0/scripts/config_environment.sh) calls `forgejo generate secret <TYPE>` for each of these on first pod start, writes them to `gitea/conf/app.ini` on the NFS share, and then explicitly refuses to overwrite them on subsequent restarts (it detects the existing `app.ini` and unsets the env vars).

We deliberately don't manage these via 1Password. The keys are generated on first run and live in `app.ini` on the NFS share, which is covered by btrfs snapshots and Hyper Backup. Any normal restore (rolling back to a snapshot, restoring from Hyper Backup, recovering after a NAS failure) pulls `app.ini` along with the repositories, so the keys come back with the data.

## SSH Hostname

The web UI is at `https://code.hoyes.dev`. SSH clones use `git@git.hoyes.dev:user/repo.git`, which is necessarily on a different subdomain / IP address.

The Forgejo SSH server runs on port 22 and is exposed via a `LoadBalancer` service. Sharing an IP address with the `nginx-ingress` controller [using MetalLB](https://metallb.universe.tf/usage/#ip-address-sharing) is not possible, since both would need to use `externalTrafficPolicy: Cluster`. We prefer to use `externalTrafficPolicy: Local` to preserve client IPs, hence we need a separate service and load balancer IP address for SSH.

Creating the separate A record is handled by an External DNS annotation on the load balancer service.

## Rootless Image

The Helm chart sets `image.rootless: true` by default and we keep that. The rootless image runs as a non-root user and ships with its own embedded SSH server (not the system OpenSSH), which sidesteps a [documented NFS issue](https://forgejo.org/docs/next/admin/installation/docker/#hosting-repository-data-on-remote-storage-systems) where SSH host key permissions become `0777` over NFS and break standard OpenSSH.

We override the chart's default UID/GID (1000:1000) to `1029:100` via `podSecurityContext` and `containerSecurityContext`, matching the `service-access` user on the NAS so files on the NFS share have consistent ownership with everything else.

## Bootstrap

Before applying, the following needs to exist:

1. The NFS share `/volume2/data-fast/forgejo` on the NAS, owned by `1029:100`, exported with NFSv4.1 enabled and "No mapping" squash.
2. A 1Password item for admin credentials at `vaults/Infrastructure/items/forgejo admin` with fields named exactly:
    - `username`
    - `password`
    - `email`
3. The 1Password item for the CloudNative-PG S3 credentials for backups (`vaults/Infrastructure/items/cloudnative-pg Versity Gateway`)

After Flux reconciles, give it a few minutes:
- The CNPG cluster bootstraps two instances.
- The Forgejo init container generates the initial `app.ini` and writes it to the NFS share.
- external-dns creates the `git.hoyes.dev` A record within its sync interval (a minute or two).

Then log in at `https://code.hoyes.dev` with the admin credentials. If desired, create a new user account for regular use.

## Limitations and Future Work

- `replicaCount: 1` is enforced even though the storage layer supports HA
  - The chart itself warns against `replicaCount > 1`
  - The default indexer used, bleve, does not support HA. We would need to switch to elasticsearch.
- `externalTrafficPolicy: Local` with a single replica means SSH and HTTPS traffic only land on the node currently running the Forgejo pod. A node failure breaks both until the pod reschedules and MetalLB re-elects. Same property as nginx-ingress today.
- Forgejo Actions are enabled in the config but no runners are deployed yet. Need to deploy a runner with access to this Forgejo instance and a registration token from the admin UI.
- The instance is private to my LAN today. To go public, add an Ingress with the `nginx-cloudflare` class (see [ingress-nginx README](/flux/infrastructure/controllers/ingress-nginx/README.md)), put a Cloudflare Access policy in front, and revisit `REQUIRE_SIGNIN_VIEW` and registration settings. Consider adding NetworkPolicies for namespace isolation at that point.
