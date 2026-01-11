# Home Assistant Kubernetes Integration

This creates the service account and role bindings used by the (third party, via HACS) [Home Assistant Kubernetes Integration](https://github.com/tibuntu/homeassistant-kubernetes), which allows monitoring and controlling Kubernetes from Home Assistant.

My Home Assistant itself runs on a separate Proxmox VM, not in Kubernetes.

Once deployed, extract the service account token:

```bash
kubectl get secret homeassistant-kubernetes-integration-token -n home-automation -o jsonpath='{.data.token}' | base64 -d
```

Then configure the integration in Home Assistant from:
- Settings -> Devices & Services
- Add Integration -> Kubernetes
- Supply the above token in the "API Token" field

Future work:
- Instead of extracting a service account token, use the [Tailscale operator API server proxy](https://tailscale.com/kb/1437/kubernetes-operator-api-server-proxy) to impersonate a group which has the same cluster permissions.
