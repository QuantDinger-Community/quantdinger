# QuantDinger Cloud Deployment Guide

This guide describes the current production-style Docker deployment for QuantDinger on a cloud server. It covers the recommended GHCR image deployment, optional source deployment, Nginx, HTTPS, upgrades, and common troubleshooting.

For first-time Docker pull and Postgres startup issues, also see [Installation Troubleshooting](INSTALL_TROUBLESHOOTING.md).

## Recommended Architecture

Use one public domain with a host-level Nginx reverse proxy:

- Public web URL: `https://app.example.com`
- Optional mobile H5 URL: `https://m.example.com`
- Host Nginx listens on `80/443`
- Docker `frontend` binds to `127.0.0.1:8888`
- Docker `mobile` binds to `127.0.0.1:8889`
- Docker `backend` binds to `127.0.0.1:5000`
- Docker `postgres` and `redis` bind to localhost only

Only expose `80` and `443` to the public internet. Keep `5000`, `5432`, and `6379` private.

## 1. Prepare the Server

Recommended baseline:

- Ubuntu 22.04 / 24.04 or Debian 12
- 2 vCPU / 4 GB RAM minimum; 4 vCPU / 8 GB RAM is better for AI-heavy use
- 30 GB+ disk space
- Security group or firewall allows `22`, `80`, and `443`
- A domain such as `app.example.com`

Create DNS records:

```text
app.example.com -> your server public IP
m.example.com   -> your server public IP  # optional mobile H5 domain
```

Verify DNS:

```bash
ping app.example.com
```

## 2. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
```

Use Compose v2 commands: `docker compose ...`.

## 3. Choose a Deployment Mode

### Recommended: GHCR prebuilt images

Use this mode for normal cloud deployment. It pulls backend, web frontend, and mobile H5 images from GHCR. No local Python or Node build is required.

```bash
mkdir -p ~/quantdinger
cd ~/quantdinger
curl -O https://raw.githubusercontent.com/OpenByteInc/QuantDinger/main/docker-compose.ghcr.yml
curl -o backend.env https://raw.githubusercontent.com/OpenByteInc/QuantDinger/main/backend/env.example
```

Edit `backend.env` before first start:

```ini
ADMIN_USER=your_admin_user
ADMIN_PASSWORD=your_strong_password
FRONTEND_URL=https://app.example.com,https://m.example.com
ALLOW_LOCAL_DESKTOP_BROKERS=false
```

The GHCR backend entrypoint can generate `SECRET_KEY` on first start and write it back to `backend.env`. You may also set `SECRET_KEY` manually to a long random string.

Create an optional project-root `.env` for Compose orchestration:

```ini
FRONTEND_PORT=127.0.0.1:8888
MOBILE_PORT=127.0.0.1:8889
BACKEND_PORT=127.0.0.1:5000
DB_PORT=127.0.0.1:5432
REDIS_PORT=127.0.0.1:6379

# Pin a release instead of floating latest, for example:
# IMAGE_TAG=5.0.1

# Use a Docker Hub mirror for postgres/redis when needed:
# IMAGE_PREFIX=docker.m.daocloud.io/library/
```

Start:

```bash
docker compose -f docker-compose.ghcr.yml pull
docker compose -f docker-compose.ghcr.yml up -d
docker compose -f docker-compose.ghcr.yml ps
```

### Optional: full repository deployment

Use this mode only when you need to build the backend from local source.

```bash
git clone https://github.com/OpenByteInc/QuantDinger.git
cd QuantDinger
cp backend/env.example backend/.env
./scripts/generate-secret-key.sh
```

Edit `backend/.env`:

```ini
ADMIN_USER=your_admin_user
ADMIN_PASSWORD=your_strong_password
FRONTEND_URL=https://app.example.com,https://m.example.com
ALLOW_LOCAL_DESKTOP_BROKERS=false
```

Optionally create project-root `.env` with the same port settings shown above.

Start:

```bash
docker compose pull
docker compose up -d --build
docker compose ps
```

## 4. Understand the Two Env Files

Keep these files separate:

| File | Used by | Purpose |
|------|---------|---------|
| `backend.env` | `docker-compose.ghcr.yml` backend container | Runtime app config: admin account, `SECRET_KEY`, LLM keys, OAuth, broker keys |
| `backend/.env` | full repository backend container | Same runtime app config when building from source |
| project-root `.env` | Docker Compose | Ports, image tags, image paths, Postgres image/data options, image mirrors |

Do not put secrets such as exchange API keys into the project-root `.env` unless Compose explicitly needs them.

## 5. Configure Nginx

Install Nginx:

```bash
sudo apt update
sudo apt install -y nginx
```

Create `/etc/nginx/sites-available/quantdinger.conf`:

```nginx
server {
    listen 80;
    server_name app.example.com;

    client_max_body_size 20m;

    location / {
        proxy_pass http://127.0.0.1:8888;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name m.example.com;

    client_max_body_size 20m;

    location / {
        proxy_pass http://127.0.0.1:8889;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

If you do not need a separate mobile domain, omit the second server block. Users can still access the mobile H5 service through the bound port or a path/domain you configure yourself.

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/quantdinger.conf /etc/nginx/sites-enabled/quantdinger.conf
sudo nginx -t
sudo systemctl reload nginx
```

If using UFW:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

## 6. Enable HTTPS

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d app.example.com -d m.example.com
sudo certbot renew --dry-run
```

If you only configured `app.example.com`, request only that domain.

Open:

```text
https://app.example.com
https://m.example.com
```

## 7. Optional API Subdomain

The recommended setup keeps API traffic same-origin through the frontend container:

```text
Browser -> https://app.example.com -> host Nginx -> frontend container -> /api -> backend:5000
```

If you need a separate `api.example.com`, expose only the host-local backend through Nginx:

```nginx
server {
    listen 80;
    server_name api.example.com;

    client_max_body_size 20m;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Also set `FRONTEND_URL` in the backend runtime env to include every public frontend origin.

## 8. Operations

For GHCR deployment:

```bash
docker compose -f docker-compose.ghcr.yml ps
docker compose -f docker-compose.ghcr.yml logs -f backend
docker compose -f docker-compose.ghcr.yml logs -f postgres
docker compose -f docker-compose.ghcr.yml restart backend
```

Update GHCR images:

```bash
docker compose -f docker-compose.ghcr.yml pull
docker compose -f docker-compose.ghcr.yml up -d
```

For full repository deployment:

```bash
git pull
docker compose pull
docker compose up -d --build
```

Back up Postgres before major upgrades:

```bash
docker exec quantdinger-db pg_dump -U quantdinger quantdinger > quantdinger_backup.sql
```

## 9. Postgres 18 and Existing Data

The current default Postgres image is `postgres:18.3-alpine`, with `PGDATA=/var/lib/postgresql/18/docker`.

If you already have a Postgres 16 data volume, do not start it with Postgres 18 directly. Either:

- keep using a matching Postgres 16 image until you migrate;
- export/import with `pg_dump` and `pg_restore`;
- run a proper `pg_upgrade` migration.

For a disposable development database only, use the command that matches your deployment mode.

GHCR deployment:

```bash
docker compose -f docker-compose.ghcr.yml down -v
docker compose -f docker-compose.ghcr.yml up -d
```

Full repository deployment:

```bash
docker compose down -v
docker compose up -d
```

Do not use `down -v` on production data.

## 10. Troubleshooting

### Image pull failures

If `redis`, `postgres`, or Docker Hub images fail to pull, set an image mirror in project-root `.env`:

```ini
IMAGE_PREFIX=docker.m.daocloud.io/library/
```

Then retry:

```bash
docker compose -f docker-compose.ghcr.yml pull
```

If GHCR images fail:

```bash
docker pull ghcr.io/openbyteinc/quantdinger-backend:latest
docker pull ghcr.io/openbyteinc/quantdinger-frontend:latest
docker pull ghcr.io/openbyteinc/quantdinger-mobile:latest
```

Common causes include network blocks, private package visibility, or a pinned tag that does not exist.

### Backend exits immediately

Check:

```bash
docker compose -f docker-compose.ghcr.yml logs --tail=100 backend
```

Common causes:

- invalid backend env syntax;
- missing or placeholder `SECRET_KEY` in full repository mode;
- database not healthy;
- wrong `DATABASE_URL` override.

### Nginx 502 or blank page

Check local services first:

```bash
curl http://127.0.0.1:8888/health
curl http://127.0.0.1:8889/health
curl http://127.0.0.1:5000/api/health
sudo nginx -t
```

Then inspect containers:

```bash
docker compose -f docker-compose.ghcr.yml ps
docker compose -f docker-compose.ghcr.yml logs --tail=100 frontend
docker compose -f docker-compose.ghcr.yml logs --tail=100 backend
```

### AI streaming stops after about 50â€“60 seconds

Typical symptoms:

- `/api/ai/chat/message/stream` returns partial content and then becomes a failed request in the browser after roughly 50â€“60 seconds;
- the client subsequently falls back to the non-streaming `/api/ai/chat/message` endpoint;
- the backend container did not restart or run out of memory, and `chat_message_stream failed` may be absent from backend logs.

This usually means an outer reverse proxy, such as host Nginx or 1Panel OpenResty, is still using default proxy timeouts and response buffering. A 600-second timeout inside the Docker frontend does not help when the outer proxy closes the SSE connection first.

Add an exact SSE location to every public domain that serves AI chat. Mobile H5 normally proxies to port `8889`; use port `8888` for the Web frontend:

```nginx
location = /api/ai/chat/message/stream {
    proxy_pass http://127.0.0.1:8889;

    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Connection "";

    proxy_buffering off;
    proxy_cache off;
    proxy_request_buffering off;

    proxy_connect_timeout 75s;
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;
    send_timeout 600s;

    add_header X-Accel-Buffering "no" always;
    add_header Cache-Control "no-cache, no-transform" always;
}
```

1Panel OpenResty commonly mounts site configuration and logs at:

```text
/opt/1panel/www/sites/<domain>/proxy/
/opt/1panel/www/sites/<domain>/log/access.log
/opt/1panel/www/sites/<domain>/log/error.log
```

Validate before reloading; replace the container name with the value from your installation:

```bash
docker exec <openresty-container> /usr/local/openresty/nginx/sbin/nginx -t
docker exec <openresty-container> /usr/local/openresty/nginx/sbin/nginx -s reload
```

If the stream still fails, search the proxy error log for `upstream timed out`, `upstream prematurely closed connection`, `499`, `502`, and `504`. Also inspect the backend container's `RestartCount` and `OOMKilled` state.

### Exchange or LLM network requests need a proxy

For backend runtime outbound requests, set `PROXY_URL` in `backend.env` or `backend/.env`.

Inside Docker, do not use `127.0.0.1` for a host proxy unless the proxy is running inside the same container. Use a reachable host address, for example:

```ini
PROXY_URL=socks5h://host.docker.internal:10808
```

On Linux, you may need to expose your proxy on a private interface or configure Docker host gateway support.

### Public ports

Do not expose these publicly:

- `5000`
- `5432`
- `6379`

Publicly expose only:

- `80`
- `443`
