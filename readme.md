# LXC Hosts

optiplex: 10.0.0.50
plex-server: 10.0.0.51
downloader: 10.0.0.52
jellyfin: 10.0.0.53

---

# Apps

## external-lxc-services.yaml

Routes external traffic from Tailscale → Traefik → Proxmox LXC containers via Kubernetes Ingress.
All resources live in the `external-services` namespace.

For each LXC, there are three resources:
- **Service** — exposes the LXC port inside the cluster
- **EndpointSlice** — maps the Service to the LXC's static IP
- **Ingress** — tells Traefik which domain to route to that Service

| Domain | LXC IP | Port |
|---|---|---|
| plex.joseberr.io | 10.0.0.51 | 32400 |
| downloader.joseberr.io | 10.0.0.52 | 8080 |
| jelly.joseberr.io | 10.0.0.53 | 8096 |

DNS (Route53) points each subdomain via CNAME to the k3s server's Tailscale MagicDNS name.
Traefik handles routing based on the `Host` header.

## Admin Tools

Headlamp and ArgoCD are accessed via on-demand port-forwards (commands aliased in `~/.bash_profile`).

- Headlamp: `k3s-pf-headlamp` → https://localhost:9090 — token generated via `k3s-gt-headlamp`
- ArgoCD: `k3s-pf-argocd` → https://localhost:9443 — credentials via `k3s-gt-argocd`

Neither is exposed via Ingress — port-forwards are spun up on demand and killed when done.

## nginx.yaml (test service)

Basic nginx deployment used to validate the cluster is working.
Runs in the `nginx-ns` namespace, exposed internally via ClusterIP on port 80.
