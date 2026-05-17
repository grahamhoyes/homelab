# Versity S3 Gateway

[Versity Gateway](https://github.com/versity/versitygw) is a lightweight S3 translation service that exposes an S3-compatible API backed by a regular POSIX filesystem (and other storage backends).

By default, the POSIX backend requires a filesystem with support for extended attributes (xattrs), though there is an [experimental option](https://github.com/versity/versitygw/wiki/POSIX-metadata) to store metadata out-of-band in a separate directory tree instead of using xattrs.

Since my data is backed by a Synology NAS that only supports NFS 4.1 (xattrs support was added in NFS 4.2), I run Versity Gateway in a container directly on the NAS with volume mounts instead of running in Kubernetes backed by NFS.

## Configuration

### Environment

1. Copy [.env.sample](./.env.sample) to `.env`
2. Change the `ROOT_SECRET_KEY` to something secure
3. Optionally change `ROOT_ACCESS_KEY` (the username)
4. Set `ADMIN_SECRET_KEY` to the same value as `ROOT_SECRET_KEY` to simplify running admin commands in the container

### Reverse Proxy

Versity Gateway can be deployed behind a reverse proxy for both admin and S3 operations. I used Synology's built-in reverse proxy (Control Panel > Login Portal > Advanced > Reverse Proxy) to set up the following proxies (all hostnames resolve to local IPs):

- S3: `https://s3.hoyes.dev` to `http://localhost:7070`.
- Admin API: `https://versitygw-admin.hoyes.dev` to `http://localhost:7071` (port from `VGW_ADMIN_PORT`)
- Web UI: `https://versitygw.hoyes.dev` to `http://localhost:7072` (port from `VGW_WEBUI_PORT`)

Use discretion if exposing to the public internet.

### Web UI

In addition to the S3 interface, Versity Gateway provides an Admin API and Web UI. The web UI requires access to the admin UI, so both must be exposed and accessible.

When using a custom domain, the hostnames for the S3 endpoint, Admin API, and Web UI must be provided to the container:

```yaml
environment:
  # S3 URLs for the Web UI
  VGW_WEBUI_GATEWAYS: "https://s3.hoyes.dev"
  # Admin API URL
  VGW_WEBUI_ADMIN_GATEWAYS: "https://versitygw-admin.hoyes.dev"
  # Web UI URL, for CORS
  VGW_CORS_ALLOW_ORIGIN: "https://versitygw.hoyes.dev"
```

All CLI management commands described below can be performed through the Web UI as well.

### Managing Users

The service is configured to use an internal IAM directory, where user accounts are stored in a `users.json` file. The contents of this file are stored in plaintext, so it isn't the most secure option. Consider using the Vault or LDAP IAM options if security is a concern. In my case, this simple setup just serves to limit bucket access for connecting services.

For more information, see the [Multi-Tenant documentation](https://github.com/versity/versitygw/wiki/Multi-Tenant).

#### Listing Users

With the container running, you can exec into it to list users:

```shell
$ docker compose exec versitygw /app/versitygw admin list-users

Account  Role      UserID  GroupID
-------  ----      ------  -------
myuser   userplus  0       0
myuser2  user      0       0
```

To run from a different host, change the `ADMIN_ENDPOINT_URL` environment variable or specify one with `--endpoint-url`:

```shell
docker compose run --rm versitygw admin --endpoint-url https://s3.hoyes.dev list-users
```

#### Creating Users

To create a new user:

```shell
docker compose exec versitygw /app/versitygw admin create-user -a <user> -s <secret_key> -r user
```

Use the `userplus` role if the user needs permission to create buckets.

#### Managing Buckets

To reassign an existing bucket to a different user:

```shell
docker compose exec versitygw /app/versitygw admin change-bucket-owner --bucket <bucket> --owner <user>
```

## Creating Buckets

The `versitygw` CLI cannot create or delete buckets and objects. For that, we need the `aws` CLI.

To create a bucket, either the root user or a user with the `userplus` role is required. If using a regular `user` role, create the bucket using the root user then reassign the bucket owner using the command above.

```shell
 export AWS_ACCESS_KEY_ID=versitygw  # Root user or user with `userplus` role
 export AWS_SECRET_ACCESS_KEY=<secret_key>

aws --endpoint-url https://s3.hoyes.dev s3 mb s3://my_bucket
```

To delete a bucket, again using either the root user or a `userplus` role:

```shell
aws --endpoint-url https://s3.hoyes.dev s3 rb s3://my_bucket --force
```
