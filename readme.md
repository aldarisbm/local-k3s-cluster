# LXC Hosts

optiplex (Proxmox): 10.0.0.50
plex-server: 10.0.0.51
downloader: 10.0.0.52
jellyfin: 10.0.0.53

---

# Apps

## external-lxc-services.yaml

Routes external traffic from Tailscale → Traefik → LXC services via Kubernetes.
All resources live in the `external-services` namespace.

For each LXC, there are three resources:
- **Service** — exposes the LXC port inside the cluster
- **EndpointSlice** — maps the Service to the LXC's static IP
- **Ingress** or **IngressRoute** — tells Traefik which domain to route to that Service

| Domain | LXC IP | Port | Protocol |
|---|---|---|---|
| plex.joseberr.io | 10.0.0.51 | 32400 | HTTP |
| downloader.joseberr.io | 10.0.0.52 | 8080 | HTTP |
| jelly.joseberr.io | 10.0.0.53 | 8096 | HTTP |
| prox.joseberr.io | 10.0.0.50 | 8006 | HTTPS (self-signed) |

Proxmox uses a Traefik `IngressRoute` CRD (instead of a standard `Ingress`) with a
`ServersTransport` that skips TLS verification for Proxmox's self-signed cert.
It is served over `websecure` (port 443) so that auth cookies work correctly.

### DNS

DNS (Route53) points each subdomain as A records to all three node Tailscale IPs for redundancy:
- k3s-server-01: 100.86.99.49
- k3s-worker-01: 100.99.132.50
- k3s-worker-02: 100.64.3.28

### ArgoCD note

`discovery.k8s.io/EndpointSlice` is removed from ArgoCD's resource exclusions in
`apps/argocd/argocd-cm.yaml` so that manually-defined EndpointSlices are managed by ArgoCD.

## traefik/helmchartconfig.yaml

Customizes the k3s-managed Traefik deployment:
- 3 replicas (one per node) with preferred pod anti-affinity
- Toleration for the `node-role.kubernetes.io/master=true:NoSchedule` taint so Traefik runs on the server node too

## argocd/

Manages ArgoCD's own config (`argocd-cm`) via GitOps.
Bootstrap once with:
```
kubectl apply -f apps/argocd/argocd-app.yaml
```

## Admin Tools

Headlamp and ArgoCD are accessed via on-demand port-forwards (commands aliased in `~/.bash_profile`).

- Headlamp: `k3s-pf-headlamp` → https://localhost:9090 — token generated via `k3s-gt-headlamp`
- ArgoCD: `k3s-pf-argocd` → https://localhost:9443 — credentials via `k3s-gt-argocd`

Neither is exposed via Ingress — port-forwards are spun up on demand and killed when done.

## nginx.yaml (test service)

Basic nginx deployment used to validate the cluster is working.
Runs in the `nginx-ns` namespace, exposed internally via ClusterIP on port 80.