# Kubernetes Ingress for External Services

I have a number of things that run outside of Kubernetes:

- Various Raspberry Pis
    - Octoprint
    - Low-power jump node if everything else breaks
- VMs running on the proxmox cluster that hosts Kubernetes and on my Synology NAS
    - Home Assisant
    - A bunch of random test VMs
    - The VMs running Kubernetes itself (see the [ansible directory](/ansible/) that creates them), which aren't relevant for this discussion
- Docker containers running on my Synology NAS

Most of these I want to be easily accessible on my local network, with the following requirements:

1. Accessible from a subdomain of `hoyes.dev`
2. DNS records don't have to be created for every service.
    - This effectively mandates that there be a reverse proxy that captures `*.hoyes.dev`
    - The main reasons for this are convienence (DNS propagation is annoying) and to avoid leaking the names of services I run over public DNS (I don't want to run my own DNS server)
3. Uses a wildcard SSL certificate
    - Also to avoid leaking the names of services to the certificate transparency logs

I previously accomplished this by:

- Pointing `*.hoyes.dev` to the local IP of my NAS
- Using the Synology reverse proxy application to route hostnames to either ports of docker containers or other devices on the network
- Running [acme.sh](https://github.com/acmesh-official/acme.sh) in a cronjob on the NAS to issue a wildcard SSL cert

I've done something similar in other locations using [Nginx Proxy Manager](https://nginxproxymanager.com/) as well, which is easier to use and handles the SSL directly. I'm sure Caddy would be great too.

Now that I have a bunch of ingresses here in Kubernetes, I want to route `*.hoyes.dev` to the Load Balancer IP of the nginx-ingress controller. That saves me from having to deal with External DNS (which violates requirement #2), and I get greater redundancy by way of the cluster vs doing it only on my NAS.

Note: Obviously configuring these records here violates the constrant of not leaking the names of all services I run. The ones in this repo I don't care about, and the rest are configured via a separate private repository (added to flux [here](/flux/repositories/homelab-private.yaml)).

## Configuring Ingresses

Kubernetes Ingresses are normally backed by Services. Services typically use a selector to select a set of pods, or are of type `ExternalName` which can route to external DNS records (basically a CNAME). To configure IP addresses ourselves, we go one layer deeper and manually create an [EndpointSlice](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) that's normally created automatically be a Service.

We can create the resoures for an external service running at `192.168.1.2` on port `1234` as follows:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myservice-proxy
  namespace: default
spec:
  ports:
    - name: web         # Port name the service exposes. Could also be called `http`.
      port: 80          # Port number the service exposes
      targetPort: http  # Target port in the EndpointSlice (can be name or number)
      protocol: TCP

---

apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: myservice-proxy
  namespace: default
  labels:
    kubernetes.io/service-name: myservice-proxy
addressType: IPv4
ports:
  - name: http          # Port name, referenced by the Service above
    port: 1234          # Actual port of the thing we're proxying to
    protocol: TCP
endpoints:
  - addresses:
      - "192.168.1.2"   # IP address of the thing we're proxying to

---

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myservice-proxy
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: myservice.hoyes.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myservice-proxy
                port:
                  name: web  # Port name in the Service

  # This assumes there's already a TLS secret created by a cert-manager Certificate resource
  tls:
    - hosts:
        - myservice.hoyes.dev
      secretName: hoyes-dev-tls
```
