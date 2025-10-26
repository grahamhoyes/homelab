# Ingress Nginx

I have 3 separate deployments of [ingress-nginx](https://github.com/kubernetes/ingress-nginx) for different purposes. They are described below by ingress class name.

## `nginx`

Source: [release.yaml](./release.yaml)

The `nginx` ingress class is the default. Its controller creates a service of type `LoadBalancer`, which is made available to my local network through MetalLB. This service is accessible through the following DNS records:
- `ingress-nginx.hoyes.dev`: A record pointing to the load balancer IP on my internal services subnet. Created manually, since external-dns doesn't listen to this ingress class (more on that below)
- `*.hoyes.dev`: Wildcard CNAME record pointing to the A record above

External DNS uses an ingress class filter to ignore this ingress class, so it doesn't create individual A records. We rely on the wildcard record for all ingresses in this class.

This is the class I use most of the time. It's accessible on my LAN, and via a Tailscale subnet router.

## `nginx-cloudflare`

Source: [cloudflare-ingress.yaml](./cloudflare-ingress.yaml)

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
      secretName: hoyes-dev-tls  # Wildcard cert for *.hoyes.dev (created by cert-manager). Assumes the secret already exists.
```

If desired, access policies can be configured from the Cloudflare Zero Trust dashboard > Access > Applications. Select Self-hosted, and specify the public hostname used above.

## `nginx-tailscale`

Source: [tailscale-ingress.yaml](./tailscale-ingress.yaml)

The controller service for the `nginx-tailscale` class uses `loadBalancerClass: tailscale`, which creates a Tailscale proxy that advertises the service to the tailnet. The main goal of this is to use custom DNS records with TLS certificates instead of MagicDNS records.

Using external-dns annotations on the controller service, the following A records are automatically created in Cloudflare, pointing to the tailnet IP address of the proxy:
- `ingress-nginx-tailscale.hoyes.dev` - Can be used as a target for CNAME records
- `*.ts.hoyes.dev` - Wildcard record for services

### Option 1: Custom domain with CNAME

To expose a service at a custom domain (e.g., `service.hoyes.dev`), create a CNAME pointing to the ingress controller:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tailscale-service
  annotations:
    external-dns.alpha.kubernetes.io/hostname: service.hoyes.dev
    external-dns.alpha.kubernetes.io/target: ingress-nginx-tailscale.hoyes.dev
spec:
  ingressClassName: nginx-tailscale
  rules:
    - host: service.hoyes.dev
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
        - service.hoyes.dev
      secretName: hoyes-dev-tls  # Wildcard cert for *.hoyes.dev (created by cert-manager). Assumes the secret already exists.
```

This creates a CNAME record at `service.hoyes.dev` pointing to `ingress-nginx-tailscale.hoyes.dev` (which resolves to the Tailscale IP). The existing wildcard certificate for `*.hoyes.dev` covers this domain.

### Option 2: Use the wildcard domain

Without external-dns annotations, the service will be available at `service.ts.hoyes.dev`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tailscale-service
spec:
  ingressClassName: nginx-tailscale
  rules:
    - host: service.ts.hoyes.dev
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
        - service.ts.hoyes.dev
      secretName: ts-hoyes-dev-tls  # Wildcard cert for *.ts.hoyes.dev (should be created similarly)
```

The wildcard DNS record `*.ts.hoyes.dev` will resolve the domain. A wildcard certificate for `*.ts.hoyes.dev` should be created in each namespace (similar to `hoyes-dev-tls`) to avoid individual domains appearing in certificate transparency logs.
