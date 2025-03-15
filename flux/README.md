# Deployment with Flux

I use [Flux](https://fluxcd.io/) to manage deploying almost everything to the k3s cluster cleated with the [Ansible playbooks](../ansible/). The exception is cilium, kube-vip, and MetalLB, which are installed by the playbook as part of the cluster creation process.

## Bootstrapping

See the [Flux GitHub bootstrapping documentation](https://fluxcd.io/flux/installation/bootstrap/github/). To avoid storing a PAT in the cluster directly, Flux can be configured to generate a deploy key (which, for a homelab, doesn't make much of a difference).

```bash
# Set the Github PAT, in my case fetching from 1password
export GITHUB_TOKEN=$(op read "op://<vault>/<secret>/<field>")

flux bootstrap github \
  --token-auth=false \
  --read-write-key=true \
  --owner=grahamhoyes \
  --repository=homelab \
  --branch=main \
  --path=flux/clusters/hoyeslab \
  --personal \
  --author-email <optional email address>
```

After bootstrapping, a secret for 1Password must also be added to the cluster. Follow the readme in [infrastructure/controllers/1password/](infrastructure/controllers/1password/).
