# Ingress Nginx

I have 3 separate deployments of [ingress-nginx](https://github.com/kubernetes/ingress-nginx) for different purposes. They are described below by ingress class name.

## `nginx`

The `nginx` ingress class is the default. Its controller creates a service of type `LoadBalancer`, which is made available to my local network through MetalLB. This service is accessible through the following DNS records:
- `ingress-nginx.hoyes.dev`: A record pointing to the load balancer IP on my internal services subnet. Created manually, since external-dns doesn't listen to this ingress class (more on that below)
- `*.hoyes.dev`: Wildcard CNAME record pointing to the A record above

External DNS uses an ingress class filter to ignore this ingress class, so it doesn't create individual A records. We rely on the wildcard record for all ingresses in this class.

This is the class I use most of the time. It's accessible on my LAN, and via a Tailscale subnet router.

## `nginx-cloudflare`

The controller service for the `nginx-cloudflare` class is of type `ClusterIP`, so it is not directly accessible from outside the cluster. Instead, [a cloudflare tunnel points to that service](../cloudflare/release.yaml). This lets me expose a service to the public internet (possibly behind Cloudflare Zero Trust access policies) by using the `nginx-cloudflare` ingress class and some External DNS annotations, as opposed to updating the cloudflare tunnel deployment to point to the backend services directly. This also lets us use other features of ingress-nginx as needed, like path rerouting.

There is a CNAME record at `hoyeslab-tunnel.hoyes.dev` which points to `<tunnel ID>.cfargotunnel.com`, for convenience.

To configure an ingress with this class, set the external-dns target to the above CNAME and configure Cloudflare proxying:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: public-service
  annotations:
    external-dns.alpha.kubernetes.io/target: hoyeslab-tunnel.hoyes.dev
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
spec:
  ingressClassName: nginx-cloudflare
  rules:
    - host: public-service.hoyes.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: public-service
                port:
                  name: http
  tls:
    - hosts:
        - public-service.hoyes.dev
      secretName: hoyes-dev-tls  # Assumes the secret already exists in the namespace
```

If desired, access policies can be configured from the Cloudflare Zero Trust dashboard > Access > Applications. Select Self-hosted, and specify the public hostname used above.

## `nginx-tailscale`

The controller service for the `nginx-tailscale` class uses `loadBalancerClass: tailscale`, which creates a tailscale proxy that advertises the service to the tailnet. The main goal of this is to be able to use our own DNS records with TLS instead of MagicDNS records.

Using External DNS annotations, this device has the following A records that point to the tailnet IP address of the proxy:
- `ingress-nginx-tailscale.hoyes.dev`. Can be used as a target for CNAMEs.
- `*.ts.hoyes.dev`

To configure an ingress at the root of the domain:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tailscale-service
  annotations:
    external-dns.alpha.kubernetes.io/hostname: tailscale-service
    external-dns.alpha.kubernetes.io/target: ingress-nginx-tailscale.hoyes.dev
spec:
  ingressClassName: tailscale-cloudflare
  rules:
    - host: tailscale-service.hoyes.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: tailscale-service
                port:
                  name: http
  tls:
    - hosts:
        - tailscale-service.hoyes.dev
      secretName: hoyes-dev-tls  # Assumes the secret already exists in the namespace
```

Without the external-dns annotations, the service will be available at `tailscale-service.ts.hoyes.dev`. However, the `tls` section will need to be updated to either reference the secret of a wildcard certificate for `*.ts.hoyes.dev`, or add a `cert-manager.io/cluster-issuer: letsencrypt` annotation to generate a certificate.
