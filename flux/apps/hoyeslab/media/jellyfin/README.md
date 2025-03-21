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

At the time of writing, I have 3 nodes in my cluster with the following CPUs and iGPU capabilities:

| CPU       | Code Name   | iGPU    | QSV Version |
|-----------|-------------|---------|-------------|
| i5-11400t | Rocket Lake | UHD 730 | Version 8   |
| i5-10500t | Comet Lake  | UHD 630 | Version 6   |
| i9-9900t  | Coffee Lake | UHD 630 | Version 6   |

Per the [Intel QSV Wikipedia page](https://en.wikipedia.org/wiki/Intel_Quick_Sync_Video), HDR to SDR transcoding capabilities were introduced in QSV version 7, so only the i5-11400t node has this capability. For that reason, at the moment, I have Jellyfin constrained to run on that node only. In reality, the other nodes are probably powerful enough to handle a few streams of SDR to HDR transcoding in software (with the decoding/encoding still in hardware), so it probably doesn't matter.

The nodes have a `gpu.hoyes.dev/type` label according to which iGPU they have (defined in the [ansible inventory](/ansible/inventory/inventory.yaml)).

My cluster also runs the [Intel Device Plugins for Kubernetes](https://github.com/intel/intel-device-plugins-for-kubernetes), which provides additional labels and resource classes for pods (see [controllers/intel-device-plugins](/flux/infrastructure/controllers/intel-device-plugins/) and [controllers/node-feature-discovery](/flux/infrastructure/controllers/node-feature-discovery/)). The node labels applied only indicate whether there is a GPU and its PCIe address, it doesn't support labelling GPU type for non-enterprise GPUs.


The two afe these are used in a `nodeSelector` to constrain the deployment:

```yaml
nodeSelector:
    gpu.hoyes.dev/type: "UHD_730"
    intel.feature.node.kubernetes.io/gpu: "true"
```

This is not highly available since there's only one of those nodes. A better way to do it would be to combine weighted node affinities with [Kubernetes Descheduler](https://github.com/kubernetes-sigs/descheduler) to move pods back to the preferred nodes once they come back online (otherwise if a UHD_730 node goes down and the pod moves to a UHD_630 node, it will stay there until manually removed).

The relevant affinities in the deployment would look some like:

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: gpu.hoyes.dev/type
                operator: In
                values:
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
                - UHD_730
                - UHD_630
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

      nodeSelector:
        intel.feature.node.kubernetes.io/gpu: "true"
```

Note that this resource type is specifically for Intel UHD graphics. For newer devices with Intel Xe graphics, use `gpi.intel.com/xe`.

See the [Intel GPU plugin docs](https://github.com/intel/intel-device-plugins-for-kubernetes/blob/main/cmd/gpu_plugin/README.md#introduction) for more information.
