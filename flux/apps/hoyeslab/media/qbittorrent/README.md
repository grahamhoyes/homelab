# qBittorrent

This deployment runs three containers in a single pod:

- **Gluetun** (sidecar): WireGuard VPN tunnel via ProtonVPN. All network traffic from the pod goes through the VPN.
- **qBittorrent** — Download client. Gluetun automatically updates the listen port when the VPN forwarded port changes.
    - Available within the cluster at `http://qbittorrent.media.svc`
    - On first run, the configuration file is loaded from [this configmap](./configmap.yaml). After loading, the configuration is copied into the config PVC and changes to the configmap will have no effect.
- **Prowlarr** — Indexer manager. Shares the VPN connection so indexer traffic is also tunneled.
    - Available within the cluster at `http://prowlarr.media.svc`

## First Setup

1. Check the qbittorrent container logs for the temporary password:
   ```
   kubectl logs -n media deployment/qbittorrent -c qbittorrent | grep "The WebUI administrator"
   ```
2. Log into the Web UI at https://qbittorrent.hoyes.dev with the username and password from the logs.
3. Go to **Options > WebUI** and set a permanent username and password.
4. Visit Prowlarr's Web UI at https://prowlarr.hoyes.dev. Set Authentication Method to `Forms (Login Page)`, and set a permanent username and password.
    - Note: Set Authentication Required to `Enabled`. The `Disabled for Local Addresses` mode includes *all* RFC 1918 private subnets, including LAN addresses where you probably want authentication. It can't currently be limited to just cluster addresses.
5. Configure Sonarr and Radarr to use qBittorrent as their download client.
