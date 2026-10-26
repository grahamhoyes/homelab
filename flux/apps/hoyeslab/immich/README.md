# Immich

Deployed with the [official chart](https://github.com/immich-app/immich-charts), with CNPG for Postgres and Dragonfly instead of the bundled valkey.

## Storage

Immich writes every file as a single user, so per-user file ownership isn't possible. Instead, everything is owned by `service-access` (1029:100) like my other apps, and users get read-only access to their own folder through DSM permissions. Writing to Immich's library from outside Immich isn't supported anyway.

| Mount                   | NAS path                                  | Contents                                                                                 |
|-------------------------|-------------------------------------------|------------------------------------------------------------------------------------------|
| `/data`                 | `/volume1/data/immich`                    | Originals (`library/<storage label>/`), upload staging (`upload/`), avatars (`profile/`) |
| `/data/thumbs`          | `/volume2/data-fast/immich/thumbs`        | Thumbnails and previews                                                                  |
| `/data/encoded-video`   | `/volume2/data-fast/immich/encoded-video` | Transcoded videos                                                                        |
| `/data/backups`         | `/volume1/app-backups/immich`             | Immich's database dumps                                                                  |
| `/external/graham` (ro) | `/volume1/homes/Graham/Photos`            | Existing photos, as an external library                                                  |

`/volume1/data` is HDD-backed, `/volume2/data-fast` is NVMe.

## Postgres

Immich requires [VectorChord](https://docs.vectorchord.ai/vectorchord/admin/kubernetes.html), which isn't in the standard CNPG image. We mount it as an [image volume extension](https://cloudnative-pg.io/docs/1.29/imagevolume_extensions/) instead of using a custom image. pgvector, which VectorChord depends on, is already in the standard image.

The app user is granted superuser through `managed.roles`, as [Immich expects](https://docs.immich.app/administration/postgres-standalone/). This lets Immich create and upgrade its own extensions, instead of needing a manual `ALTER EXTENSION vchord UPDATE` and reindex after VectorChord updates.

## Dragonfly

Immich's job queue is BullMQ, which needs `--default_lua_flags=allow-undeclared-keys` on Dragonfly.

## Configuration

`immich.configuration` is left empty during evaluation. Setting it makes *all* settings read-only in the UI. Once settings have stabilized, export them from Administration → Settings into the release values.

Storage labels are per user and can't be set in config. Set them in Administration → Users.

## Initial Setup

1. On the NAS, create the following, owned by `service-access` (1029:100):
    - `/volume1/data/immich`
    - `/volume1/app-backups/immich`
    - `/volume2/data-fast/immich/thumbs`
    - `/volume2/data-fast/immich/encoded-video`

   These are mounted directly, so the pod won't start until they exist.
2. Grant `service-access` read-only access to `/volume1/homes/Graham/photos`, so it can be accessed as an external library
3. Add a read-only NFS rule for the cluster on the `homes` share.
4. Give each DSM user read-only access to their `immich/library/<username>` folder.
5. After the first login, before uploading anything:
    - Set the user's storage label to their DSM username.
    - Enable the storage template (`{{y}}/{{MM}}/{{filename}}` to match Synology's layout).
    - Add `/external/graham` as an external library.

Optionally, to browse photos from the home folder over SMB, enable "Allow symbolic links across shared folders" in DSM's SMB settings, then:

```sh
ln -s /volume1/data/immich/library/graham /volume1/homes/Graham/Immich
```

Synology Drive doesn't seem to follow these. Keep the link outside `Photos/`, so it isn't picked up by the external library.

## Cutover from Synology Photos

Don't enable mobile backup while the external library exists. External libraries aren't checked when deciding which photos are already on the server, so everything will get re-uploaded.

1. Import `Photos/` into Immich's own library with [immich-go](https://github.com/simulot/immich-go). Use `upload from-google-photos` for the Takeout dumps, so dates and albums are read from the JSON sidecars.
2. Delete the external library in Immich, and remove the `external-graham` mount.
3. Enable mobile backup. Already-imported photos are skipped by checksum.
