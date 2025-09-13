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
docker compose run --rm versitygw admin --endpoint-url https://versitygw.hoyes.dev list-users
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

### Reverse Proxy

As demonstrated in the examples above, Versity Gateway can be deployed behind a reverse proxy for both admin and S3 operations. I used Synology's built-in reverse proxy (Control Panel > Login Portal > Advanced > Reverse Proxy) to proxy `https://versitygw.hoyes.dev` (which resolves to a local IP address) to `http://localhost:7070`.

Use discretion if exposing to the public internet.

## Creating Buckets

The `versitygw` CLI cannot create or delete buckets and objects. For that, we need the `aws` CLI.

To create a bucket, either the root user or a user with the `userplus` role is required. If using a regular `user` role, create the bucket using the root user then reassign the bucket owner using the command above.

```shell
 export AWS_ACCESS_KEY_ID=versitygw  # Root user or user with `userplus` role
 export AWS_SECRET_ACCESS_KEY=<secret_key>

aws --endpoint-url https://versitygw.hoyes.dev s3 mb s3://my_bucket
```

To delete a bucket, again using either the root user or a `userplus` role:

```shell
aws --endpoint-url https://versitygw.hoyes.dev s3 rb s3://my_bucket --force
```
