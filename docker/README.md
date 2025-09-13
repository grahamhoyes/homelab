# Docker Configs

While most of my homelab services are deployed to Kubernetes and managed through the [Flux directory](../flux/), I run some services using standalone Docker Compose stacks. This is typically done to:

- Run services on specific devices (e.g., Raspberry Pi)
- Access particular hardware directly (e.g., direct storage on my NAS without going through NFS)

Renovate will update image versions in this directory, however I don't have anything that auto-updates the stacks on the end machines.

## Services

- **[versity-gateway](./versity-gateway/)** - S3-compatible API gateway backed by POSIX filesystem storage, running on my NAS
