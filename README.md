# Homeserver Docker Setup

These are my Docker Compose files for services running on my personal server.

## Stack

| Service | Image | Purpose |
|---|---|---|
| Caddy | `caddy:latest` | Reverse proxy, automatic HTTPS |
| Nextcloud | `nextcloud:latest` | File cloud |
| MariaDB | `mariadb:10.11` | Nextcloud database |
| Umami | `ghcr.io/umami-software/umami:postgresql-latest` | Website analyticstool |
| PostgreSQL | `postgres:16-alpine` | Umami database |
| Portainer | `portainer/portainer-ce:latest` | Docker web UI |

## Setup

### 1. Create the shared Docker network

All services that go through Caddy share an external network. Create it once before starting any stack:

```bash
docker network create caddy_net
```

### 2. Configure environment variables

Each stack has its own `.env` file. Copy the examples and fill in your values:

```bash
cp nextcloud.env.example nextcloud.env
cp umami.env.example umami.env
```

**nextcloud.env**
```env
MYSQL_ROOT_PASSWORD=your_root_password
MYSQL_PASSWORD=your_user_password
NEXTCLOUD_DOMAIN=cloud.example.com
```

**umami.env**
```env
UMAMI_DB_PASSWORD=secure_password
UMAMI_APP_SECRET=long_random_string  # generate with: openssl rand -hex 32
```

### 3. Configure Caddy

Edit `Caddyfile` and replace the example domains with your actual domains. Caddy handles TLS automatically via Let's Encrypt.

### 4. Create data directories

```bash
# Nextcloud
mkdir -p /home/sasha/services/nextcloud/data/db
mkdir -p /home/sasha/services/nextcloud/data/app

# Umami
mkdir -p /home/sasha/services/umami/data/db
```

### 5. Start the services

Start each stack separately, Caddy first so the network and routing are ready:

```bash
docker compose -f docker-compose-caddy.yml up -d
docker compose -f docker-compose-nextcloud.yml --env-file nextcloud.env up -d
docker compose -f docker-compose-umami.yml --env-file umami.env up -d
docker compose -f docker-compose-portainer.yml up -d
```

### 6. Umami first login

Navigate to `https://analytics.example.com` and log in with the default credentials:

- **Username:** `admin`
- **Password:** `umami`

Change the password immediately after first login.

## Networking

```
Internet
   │  (80/443)
   ▼
 Caddy  ──────────────────────► nextcloud_app  (caddy_net)
   │    ──────────────────────► umami_app      (caddy_net)
   │
   └─ Portainer reachable on :9000 directly (no reverse proxy)
```

- `caddy_net` is an **external** network shared by Caddy and all proxied services.
- `internal_net_cloud` isolates Nextcloud and its MariaDB from everything else.
- `internal_net_umami` isolates Umami and its PostgreSQL from everything else.
- Portainer exposes port `9000` directly and does not go through Caddy.

## Ports

| Port | Service |
|---|---|
| 80 | Caddy (HTTP → redirected to HTTPS) |
| 443 | Caddy (HTTPS + HTTP/3) |
| 9000 | Portainer web UI |


## Volumes

Nextcloud and MariaDB use **bind mounts** into absolute paths so backups are straightforward:

```bash
# Backup
tar -czf backup.tar.gz /home/your-user/services/

# Restore
tar -xzf backup.tar.gz
```

Umami uses a bind mount for its PostgreSQL data at `./data/db`. For database-level backups use `pg_dump`:

```bash
docker exec umami_db pg_dump -U umami umami > umami_backup.sql
```