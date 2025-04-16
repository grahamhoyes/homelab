# High Availabitily Pi-hole in Kubernetes

This group of resources creates a high availability [Pi-hole](https://pi-hole.net/) running in Kubernetes, great for homelab use (you probably don't want to do anything like this in production clusters).

We create a single primary instance from which configuration can be edited through the web UI, which is then synced to multiple replicas. The primary instance is exposed as one IP address outside the cluster, and the replicas share a separate IP address. This works well for clients or routers where you can only specify two DNS servers.

## Motivation

Why would you want to deploy mulitple Pi-hole instances to Kubernetes? For that matter, why would you want to deploy Pi-hole to Kubernetes in the first place? Mostly because it's fun! But if you want some logical reasons, here are some:
- Deploying all the instances can be done through GitOps, [Flux CD](https://fluxcd.io/) in my case (though this section uses nothing particular to Flux). That's great for if you break things and need to recreate from scratch.
- Having a single point of failure for DNS, even if it is for home use, is not ideal. If you have a Kubernetes cluster running on multiple nodes as I do, we can ensure that the Pi-hole instances are automatically spread across nodes to mitigate the impact of hardware failure.
- Did I mention it's fun?

## Pre-requisites

The way this is setup assumes you have a few things already available in your cluster:
- [Longhorn](https://longhorn.io/) for distributed storage and backups ([my deployment](/flux/infrastructure/controllers/longhorn/)). This is not strictly required, and is easily removed with minimal impact other than losing backups.
- [MetalLB](https://metallb.io/) for services of type `LoadBalancer`. You can use another load balancer provider as long as they have a way to create UDP and TCP services for the same port on the same IP address.
    - kube-vip [seems to support this](https://kube-vip.io/docs/usage/kubernetes-services/#multiple-services-on-the-same-ip) without the need for any special annotations like MetalLB needs
- Some way of getting secrets into the cluster. I use the [1password Operator](https://developer.1password.com/docs/k8s/operator/) ([my deployment](/flux/infrastructure/foundation/1password/)), but [External Secrets Operator](https://external-secrets.io/latest/) or just creating Secrets manually would work just as well.


# The Setup

There are three components for this setup:
- A primary Pi-hole instance, created with a `StatefulSet` with one replica.
    - This has a service of type `LoadBalancer` for a dedicated IP address that should be set as the primary DNS server in your router / clients.
    - It also has an ingress for reaching the web UI.
- A secondary set of Pi-hole instances, created with a `StatefulSet` with multiple replicas. I use two replicas, but you can have as many as you want.
    - These share a service of type `LoadBalancer`, which should be set as the secondary DNS server in your router / clients.
    - These do not have an ingress for accessing the web UI, and their load balancer services don't expose the web UI ports either.
- [nebula-sync](https://github.com/lovelaze/nebula-sync/tree/main) for syncing configuration from the primary instance to the replicas.

<p align="center" style="text-align:center">
    <img src="img/pihole-kubernetes.svg" alt="Pi-hole Kubernetes Diagram" />
</p>
