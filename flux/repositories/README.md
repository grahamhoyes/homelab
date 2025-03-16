# Other Repositories

Flux isn't limited to pulling from just one repository. Here I have `GitRepository` references to other projects that themselves contain multiple things to deploy (if a repository has just one thing to deploy, it goes in `flux/apps`).

The main use case is for my `homelab-private` repository, which is structured identically to this one and has some private / experimental deployments.

Repositories are configured here as a separate Kustomization rather than directly in `flux/clusters/<cluster>` only because they depend on 1Password being provisioned from the `infrastructure-controllers` Kustomization to provide repository access secrets.
