# Homeserver Docker Setup

These are my yaml files for docker containers on my personal server.

## Stack

| Service | Image | Purpose |
|---|---|---|
| Caddy | `caddy:latest` | Reverse proxy, automatic HTTPS |
| Nextcloud | `nextcloud:latest` | File Cloud|
| MariaDB | `mariadb:10.11` | Nextcloud database |
| Portainer | `portainer/portainer-ce:latest` | Docker web UI |

## Setup

### 1. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and set your values:

```env
MYSQL_ROOT_PASSWORD=your_root_password
MYSQL_PASSWORD=your_user_password
NEXTCLOUD_DOMAIN=cloud.example.com
```

### 2. Configure Caddy

Edit `Caddyfile` and replace `cloud.example.com` with your actual domain. Caddy handles TLS automatically via Let's Encrypt.

### 3. Create data directories

```bash
mkdir -p data/db data/app
```

### 4. Start the services

Start each stack separately, Caddy first so the network is ready:

```bash
docker compose -f docker-compose-caddy.yml up -d
docker compose -f docker-compose-nextcloud.yml up -d
docker compose -f docker-compose-portainer.yml up -d
```

## Networking

```
Internet
   │  (80/443)
   ▼
 Caddy  ──────────────────────► nextcloud_app (caddy_net)
   │
   └─ Portainer reachable on :9000 directly (no reverse proxy)
```

- `caddy_net` is an **external** network shared by Caddy and Nextcloud.
- `internal_net_cloud` is an **internal** network between Nextcloud and its database, not exposed to Caddy or the outside.
- Portainer exposes port `9000` directly and does not go through Caddy.

## Ports

| Port | Service |
|---|---|
| 80 | Caddy (HTTP -> redirected to HTTPS) |
| 443 | Caddy (HTTPS + HTTP/3) |
| 9000 | Portainer web UI |

## Volumes

Nextcloud and MariaDB use **bind mounts** into `./data/` so backups are straightforward:

```bash
# Backup
tar -czf backup.tar.gz data/

# Restore
tar -xzf backup.tar.gz
```

> **Windows:** The bind mounts in `docker-compose-nextcloud.yml` are Linux-only. Switch to the commented out named volumes at the bottom of that file if deploying on Windows.