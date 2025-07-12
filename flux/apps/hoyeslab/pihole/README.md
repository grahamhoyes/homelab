# High Availabitily Pi-hole in Kubernetes

This group of resources creates a high availability [Pi-hole](https://pi-hole.net/) running in Kubernetes, great for homelab use (you probably don't want to do anything like this in production clusters).

We create a single primary instance from which configuration can be edited through the web UI, which is then synced to multiple replicas. The primary instance is exposed as one IP address outside the cluster, and the replicas share a separate IP address. This works well for clients or routers where you can only specify two DNS servers.

## Motivation

Why would you want to deploy multiple Pi-hole instances to Kubernetes? For that matter, why would you want to deploy Pi-hole to Kubernetes in the first place? Mostly because it's fun! But if you want some logical reasons, here are some:
- Deploying all the instances can be done through GitOps, [Flux CD](https://fluxcd.io/) in my case (though this section uses nothing particular to Flux). That's great for if you break things and need to recreate from scratch.
- Having a single point of failure for DNS, even if it is for home use, is not ideal. If you have a Kubernetes cluster running on multiple nodes as I do, we can ensure that the Pi-hole instances are automatically spread across nodes to mitigate the impact of hardware failure.
- Did I mention it's fun?

## Pre-requisites

The way this is setup assumes you have a few things already available in your cluster:
- [Longhorn](https://longhorn.io/) for distributed storage and backups ([my deployment](/flux/infrastructure/controllers/longhorn/)). This is not strictly required, and is easily removed with minimal impact other than losing backups.
- [MetalLB](https://metallb.io/) for services of type `LoadBalancer`. You can use another load balancer provider as long as they have a way to create UDP and TCP services for the same port on the same IP address.
    - kube-vip [seems to support this](https://kube-vip.io/docs/usage/kubernetes-services/#multiple-services-on-the-same-ip) without the need for any special annotations like MetalLB needs
- Some way of getting secrets into the cluster. I use the [1password Operator](https://developer.1password.com/docs/k8s/operator/) ([my deployment](/flux/infrastructure/foundation/1password/)), but [External Secrets Operator](https://external-secrets.io/latest/) or just creating Secrets manually would work just as well.


## The Setup

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

The two load balancers expose two virtual IPs, which are set as the primary and secondary DNS servers on my network respectively (via the WAN settings on my Unifi Cloud Gateway Max).

Most consumer routers only allow setting two DNS servers, so this setup works well. This is the case for Unifi Network when setting DNS servers at the WAN level (which automatically apply to all networks when using auto DNS configuration), though if you want to apply DNS to individual networks you can specify up to four servers there.

### Kustomize

I won't go into detail on the actual kubernetes manifests, as they are relatively straightforward. It is briefly worth discussing how [Kustomize](https://kustomize.io/) is used here so you can apply and explore the manifests yourself.

Between the primary and replica deployments, there are a lot of resources that need to get deployed twice: the StatefulSets, TCP and UDP services, and the headless service used by nebula-sync. Duplicating all of them would be error-prone (forgetting to update environment variables on one StatefulSet for example), so instead Kustomize overlays are used to modify and apply a base configuration.

Let's look at the directory tree for all the kubernetes services (slightly rearranged):

```
.
├── kustomization.yaml
├── certificate.yaml
├── namespace.yaml
├── nebula-sync
│   └── kustomization.yaml
│   ├── deployment.yaml
└── pihole
    ├── base
    │   ├── kustomization.yaml
    │   ├── service-headless.yaml
    │   ├── service-lb-tcp.yaml
    │   ├── service-lb-udp.yaml
    │   └── statefulset.yaml
    ├── kustomization.yaml
    ├── overlays
    │   ├── primary
    │   │   ├── kustomization.yaml
    │   │   ├── ingress.yaml
    │   │   ├── service-headless.patch.yaml
    │   │   ├── service-lb.patch.yaml
    │   │   ├── service-lb-tcp.patch.yaml
    │   │   └── statefulset.patch.yaml
    │   └── replica
    │       ├── kustomization.yaml
    │       ├── service-headless.patch.yaml
    │       ├── service-lb.patch.yaml
    │       └── statefulset.patch.yaml
    ├── pdb.yaml
    └── secret.yaml
```

Each of the `kustomization.yaml` files points to either a list of yaml files to deploy, or to directories containing another `kustomization.yaml`.

The root Kustomization deploys the base resources (the namespace and a certificate for the ingress), then points to the `nebula-sync` and `pihole` Kustomizations. `nebula-sync` is straightforward, having just a deployment.

The `pihole` Kustomization ([pihole/kustomization.yaml](./pihole/kustomization.yaml)) is more complex. It configures the shared resources (secret for the web UI and a pod disruption budget), then points at the kustomizations for the [primary](./pihole/overlays/primary/kustomization.yaml) and [replica](./pihole/overlays/replica/kustomization.yaml) overlays.

These overlays each point to the [base kustomization](./pihole/base/kustomization.yaml) and:
- Add a name suffix to everything (`-primary` or `-replica`)
- Apply a series of [patches](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/) to configure them accordingly.
    - Patches without a `target` match based on the fields defined in the patch file (see the [docs](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/))
    - Those with a `target` also consider the fields defined there in addition to what's in the patch files (this is just to use regex patterns so we don't need separate patch files for the TCP and UDP services).

Patches are responsible for:
- Setting the virtual IP addresses of the respective load balancers
- Setting metadata and selectors (via the `app.kubernetes.io/component` annotation)
- Adding the web UI ports to the TCP service for the primary instance

Lastly, the primary overlay also deploys an ingress for the web UI.

If you're in the same directory as this readme, you can locally render the full kustomization with:

```bash
kubectl apply -k . --dry-run=client -o yaml
```

If you want to look at just the primary configuration, you can run:

```bash
kubectl apply -k pihole/overlays/primary --dry-run=client -o yaml
```

To apply (if you're not using Flux like me):

```bash
kubectl apply -k .
```

### Blocklists

I get my blocklists from [the Firebog list](https://firebog.net/), and set them manually via the UI.

### Router Configuration

Broadly, there are two ways to configure Pi-hole as your DNS server for all devices on your network:

- If your router supports it, configure the two load balancers as primary and secondary *upstream* DNS servers for your router. Let clients treat your router as their DNS server.
    ```
    Client -> Router -> Pi-hole (primary or replica) -> Upstream DNS
    ```
    - Advantage: it should work for all devices automatically, unless they have manually configured their DNS servers (even with static network configurations, devices can be set to use the default gateway as their DNS server).
    - Disadvantage: Query logs will only contain your router's IP
        - See below, depending on the `externalTrafficPolicy` client IPs may not be visible regardless

- Use your router's DHCP settings to have it push the Pi-hole servers to your client devices.

    ```
    Client -> Pi-hole (primary or replica) -> Upstream DNS
    ```

    - Advantage: Query logs will contain your actual devices' IP addresses
        - This may be a disadvantage if you want them obscured for privacy, but if you do you should just turn off query logging in Pi-hole
        - See below, depending on the `externalTrafficPolicy` client IPs may not be visible regardless
    - Disadvantage: Will not work for devices with static network configurations

I actually use both approaches simultaneously:

- My default VLAN (where all my real devices live) and my IoT network has the Pi-hole DNS servers configured via DHCP, so I can debug issues on my devices and see what the IoT devices are up to.
- The rest of my VLANs which mostly rely on static IPs use the router as their DNS server, which has Pi-hole as its upstream.

This gives me a good balance of convenience and functionality. The only place it doesn't work is for devices on my default VLAN that have their DNS servers manually set to Cloudflare (mostly my work devices), which I'm fine with.

See [Pi-hole's documentation on router setup](https://docs.pi-hole.net/routers/ubiquiti-usg/) for configuration instructions for your router.

## Limitations

- Query logs are only saved on the instance where the request lands. The replica servers don't have ingresses, so troubleshooting blocked queries that land on them can be tricky.
    - You can use `kubectl port-forward -n pihole pod/pihole-replica-0 8080:80` to access the first replica at `localhost:8080/admin` and so on.
- I don't use Pi-hole as a DHCP server. In order to do that:
    - Add the `NET_ADMIN` capability to the pod's security context.
    - Add UDP port 67 to the UDP load balancer
    - Set `SYNC_CONFIG_DHCP` in nebula-sync's environment variables to `"true"`
- My entire kubernetes cluster is plugged into the same power socket, so if it goes down I lose DNS. But my router is also on that same socket, so I'll be down anyway. See below for options to improve this.
- To prioritize availability, the statefulsets use a `preferredDuringSchedulingIgnoredDuringExecution` anti-affinity constraint that selects all Pi-hole instances (primary and replica). The fact that this is preferred rather than required means that if you end up with less nodes than you have instances, some will be scheduled on the same node.
    - This seems preferable to me over using `requiredDuringSchedulingIgnoredDuringExecution`, having the node hosting the primary instance go down, and no longer having access to the primary until a new node is available.
    - If you are temporarily in a situation where there are less nodes than replicas and Kubernetes schedules two pods on the same node, it won't re-distribute them automatically until one of the pods is removed (either manually or for some other reason).
- Depending on the external traffic policy, client IPs may not be visible (see below)

### External Traffic Policy

There are two options for the external traffic policy of the MetalLB DNS load balancer service, which have their own advantages / disadvantages.

I run MetalLB in Layer 2 mode, which means at any given time only a single node is advertising the services' virtual IP (via ARP). What happens next depends on the external traffic policy:

- `externalTrafficPolicy: Cluster`: All requests arrive at the advertising node, and `kube-proxy` then load balances between pods throughout the cluster.
    - Advantage: Traffic is balanced across pods
    - Disadvantage: Source IP address is obscured by `kube-proxy`. All requests will appear to come from the node that broadcasted the virtual IP.
- `externalTrafficPolicy: Local`: All requests arrive at the advertising node, and are only allocated to pods that are on *that* node.
    - Advantage: Source IP address is preserved.
    - Disadvantage: There is no active load balancing of traffic. Replicas are only used for failover (at which case a different node will start advertising the service IP).

See MetalLB's [docs on traffic policies](https://metallb.io/usage/index.html) for more details.

Given that this entire set up is about high availability for failover, not load balancing traffic (DNS load is trivial), I use `externalTrafficPolicy: Local` to preserve source IPs. This only impacts the replica statefulset anyway, since the primary statefulset only has a single instance.

If you use BGP for MetalLB instead (Layer 3 mode), `externalTrafficPolicy: Local` will give you full load balancing and is theoretically the most efficient configuration given that we have a pod anti-affinity constraint to keep pods on separate nodes.


## Future Work

- Set up unbound as a recursive DNS resolver, and use that as the upstream for Pi-hole
- Add one more replica outside of kubernetes (on a Raspberry Pi), in case the whole cluster goes down. To work around router limitations of two DNS servers, we would need something like keepalived load balancing between the load balancer and the final replica.
    - The main motivation for me to do this is that my entire kubernetes cluster is connected to the same PDU. However, my router is also on that PDU, so if it gets turned off I lose internet anyway so I won't have much of a need for DNS at that point.
    - Some alternatives to this:
        - Deploy Pi-hole on a separate Raspberry Pi, and switch to using network-level DNS configuration in Unifi Network rather than WAN-level, which would allow specifying a third DNS server.
        - Deploy a Raspberry Pi somewhere else in my network, but make it a k3s node so we can keep using metallb. Figure out how to make sure one Pi-hole replica stays on that node.
- Instead of setting the primary / replicas URL + password configuration string for nebula-sync in 1password as its own field, switch to External Secrets Operator and use templating to build that variable with just the password stored in 1password.
