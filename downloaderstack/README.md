# Downloader Stack

Docker Compose stack for automated media downloading. All services route traffic through a Gluetun VPN container.

## Services

| Service | Port | Purpose |
|---|---|---|
| gluetun | — | VPN gateway (PIA / OpenVPN) |
| qbittorrent | 8080 | Torrent client |
| radarr | 7878 | Movie management |
| sonarr | 8989 | TV show management |
| prowlarr | 9696 | Indexer aggregator |
| flaresolverr | 8191 | Cloudflare bypass for indexers |
| port-manager | — | Syncs gluetun's forwarded port → qBittorrent |
| autoheal | — | Auto-restarts unhealthy containers |

## Network

All services except `autoheal` use `network_mode: "container:vpn"`, meaning they share the gluetun network namespace. All traffic exits through the VPN. Ports are exposed on the `gluetun` container only.

## Access

The downloader stack runs on the `downloader` LXC (`10.0.0.52`). Services are exposed externally via Tailscale → Traefik → Kubernetes ingress.

| Domain | Port |
|---|---|
| downloader.joseberr.io | 8080 (qBittorrent) |
| sonarr.joseberr.io | 8989 |
| radarr.joseberr.io | 7878 |
| prowlarr.joseberr.io | 9696 |

DNS (Route53) resolves each subdomain as A records to all three k3s node Tailscale IPs for redundancy:

| Node | Tailscale IP |
|---|---|
| k3s-server-01 | 100.86.99.49 |
| k3s-worker-01 | 100.99.132.50 |
| k3s-worker-02 | 100.64.3.28 |

## Setup

1. Copy `.example.env` to `.env` and fill in your credentials:

```bash
cp .example.env .env
```

| Variable | Description |
|---|---|
| `OVPN_USER` | Private Internet Access OpenVPN username |
| `OVPN_PASSWORD` | Private Internet Access OpenVPN password |
| `QBIT_USER` | qBittorrent Web UI username |
| `QBIT_PASSWORD` | qBittorrent Web UI password |

2. Start the stack:

```bash
docker compose up -d
```

## Health Checks

Each service has a health check configured. Services that depend on the VPN will not start until `gluetun` reports healthy.

| Service | Check | Interval | Start Period |
|---|---|---|---|
| gluetun | `ping -c 1 8.8.8.8` | 45s | 90s |
| qbittorrent | `curl -f http://localhost:8080` | 1m | 30s |
| prowlarr | `curl -f http://localhost:9696` | 1m | 30s |
| radarr | `curl -f http://localhost:7878` | 1m | 30s |
| sonarr | `curl -s http://localhost:8989` | 1m | 30s |
| flaresolverr | `curl -f http://localhost:8191` | 1m | 30s |

**gluetun** uses a ping-based check with a 90s start period to allow time for VPN connection and MTU discovery before checks begin.

## Auto-healing

The `autoheal` container watches all services and automatically restarts any that enter an `unhealthy` state.

- Watches all containers with a configured health check (`AUTOHEAL_CONTAINER_LABEL=all`)
- Checks every 60 seconds
- Waits 5 minutes after stack boot before taking any restart action (`AUTOHEAL_START_PERIOD=300`)

## Volumes

- `gluetun_data` — named volume shared between `gluetun` and `port-manager` for the forwarded port file (`/tmp/gluetun/forwarded_port`)
- `/mnt/knightrider/data` — bind-mounted into qbittorrent, radarr, and sonarr at `/data` so hardlinks work across the download/media directory tree