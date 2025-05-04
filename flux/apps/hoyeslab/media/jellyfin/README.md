# Jellyfin

This set of manifests deploys Jellyfin. I chose to define manifests manually over the [official helm chart](https://github.com/jellyfin/jellyfin-helm/tree/master/charts/jellyfin) at the time of writing for a few reasons:

- The helm chart doesn't use the release namespace. I deploy this to my `media` namespace.
- There are a few bugs in the chart

## Volumes

We use 3 volumes for Jellyfin:

- The config volume, which stores things like user accounts, watch history, and settings is configured as a PVC replicated with Longhorn. This ensures that such data can be backed up and moved between nodes as needed.
- The transcoding directory, which stores in-progress transcodes. This uses an `emptyDir` volume, which on my nodes are always backed by NVMe SSDs.
- The data directory, which holds actual media. This is an NFS volume that mounts my NAS.

## Node Selection

At the time of writing, I have 4 nodes in my cluster with the following CPUs and iGPU capabilities:

| CPU       | Code Name   | iGPU    | QSV Version |
|-----------|-------------|---------|-------------|
| i7-13700t | Raptor Lake | UHD 770 | Version 8   |
| i5-11400t | Rocket Lake | UHD 730 | Version 8   |
| i5-10500t | Comet Lake  | UHD 630 | Version 6   |
| i9-9900t  | Coffee Lake | UHD 630 | Version 6   |

Per the [Intel QSV Wikipedia page](https://en.wikipedia.org/wiki/Intel_Quick_Sync_Video), HDR to SDR transcoding capabilities were introduced in QSV version 7, so the i5-11400t and i7-13700t nodes have this capability. Hence, I constrain Jellyfin to run only on those nodes. In reality, the other nodes are probably powerful enough to handle a few streams of HDR to SDR transcoding in with OpenCL VPP tone mapping, so it probably doesn't matter.

The nodes have a `gpu.hoyes.dev/type` label according to which iGPU they have (defined in the [ansible inventory](/ansible/inventory/inventory.yaml)).

My cluster also runs the [Intel Device Plugins for Kubernetes](https://github.com/intel/intel-device-plugins-for-kubernetes), which provides additional labels and resource classes for pods (see [controllers/intel-device-plugins](/flux/infrastructure/controllers/intel-device-plugins/) and [controllers/node-feature-discovery](/flux/infrastructure/controllers/node-feature-discovery/)). The node labels applied only indicate whether there is a GPU and its PCIe address, it doesn't support labelling GPU type for non-enterprise GPUs.


The two of these are used in a node affinity to constrain the deployment:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: gpu.hoyes.dev/type
          operator: In
          values:
            - UHD_730
            - UHD_770
        # Technically redundant with the above, but makes sure that the deployment fails
        # if something is wrong with the intel device plugins
        - key: intel.feature.node.kubernetes.io/gpu
          operator: In
          values:
            - "true"
```

Since I have two such nodes, I have sufficient high availability to keep Jellyfin active while doing maintenance on any node.

For even greater availability, we could combine weighted node affinities with [Kubernetes Descheduler](https://github.com/kubernetes-sigs/descheduler) to move pods back to the preferred nodes once they come back online (otherwise if a UHD_770 / UHD_730 node goes down and the pod moves to a UHD_630 node, it will stay there until manually removed).

The relevant affinities in the deployment would look some like:

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: gpu.hoyes.dev/type
          operator: In
          values:
          - UHD_770
          - UHD_730
    - weight: 50
      preference:
        matchExpressions:
        - key: gpu.hoyes.dev/type
          operator: In
          values:
          - UHD_630

    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: gpu.hoyes.dev/type
          operator: In
          values:
          - UHD_770
          - UHD_730
          - UHD_630
        - key: intel.feature.node.kubernetes.io/gpu
          operator: In
          values:
            - "true"
```

## Hardware GPU Acceleration (Intel)

Jellyfin supports hardware GPU acceleration. As above, all of my nodes have Intel iGPUs that can be used, though I only run on the newest one at the moment.

The jellyfin deployment leverages the [Intel Device Plugins](/flux/infrastructure/controllers/intel-device-plugins/) to get access to GPUs inside the pod(s) without needing to manually mount `/dev/dri` render devices. In my current configuration, multiple deployments can share a GPU if needed.

In the deployment, set the following resource reuqests on the container(s) that need access to a render device. You should also specify a node selector:

```yaml
spec:
  template:
    spec:
      containers:
        - name: ...
          resources:
            requests:
              gpu.intel.com/i915: "1"

      # Ore use a node affinity term like above
      nodeSelector:
        intel.feature.node.kubernetes.io/gpu: "true"
```

Note that this resource type is specifically for Intel UHD graphics. For newer devices with Intel Xe graphics, use `gpi.intel.com/xe`.

See the [Intel GPU plugin docs](https://github.com/intel/intel-device-plugins-for-kubernetes/blob/main/cmd/gpu_plugin/README.md#introduction) for more information.

## Additional Notes

Some additional things to note that helped me get GPU transcoding working in my setup, not all of which may actually be required:
- The [configure-proxmox.yaml ansible playbook](/ansible/configure-proxmox.yaml) configures IOMMU settings in GRUB on the Proxmox hosts
- As described in the [ansible directory readme](/ansible/), a resource mapping is used in Proxmox to make the iGPU PCI device available to the VMs
- A number of packages and kernel updates are applied in the [VM system requirements task](/ansible/roles/system_requirements/tasks/main.yaml)
  - `linux-modules-extra-<kernel version>` is required
  - `linux-generic-hwe-24.04` to get an up to date kernel (for Ubuntu 24.04).
    - There are occasional incompatibilities with the [Intel Compute Runtime](https://github.com/intel/compute-runtime) and the `jellyfin-opencl-intel` linuxserver mod that may necessitate this.
    - In particular, I found that the combination of the latest Intel Compute Runtime ([release 25.13.33276.16](https://github.com/intel/compute-runtime/releases/tag/25.13.33276.16) at the time of writing), kernel 6.8.0 on Ubuntu 24.04.02, and the i7-13700 were incompatible, so I run kernel `6.11.0-25` now.
