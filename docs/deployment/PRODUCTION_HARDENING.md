# Production Hardening

## Preflight

Create the project-root `.env` and `backend/.env`, then replace every default credential. Validate them before deployment:

```bash
python backend/scripts/check_production_config.py \
  --env-file .env \
  --env-file backend/.env
```

The guard rejects default database, administrator, Grafana, JWT, and credential-encryption secrets.

## Locked runtime

The production override runs backend processes as UID/GID `10001`, drops Linux capabilities, makes the root filesystem read-only, constrains memory/CPU, and mounts the backend environment file read-only:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.production.yml \
  -f docker-compose.observability.yml \
  up -d --build
```

Prepare both backend secrets before the first locked start because the non-root containers cannot generate or persist them. Run migrations through the bundled migration service; do not run migrations concurrently from API workers.

## Redis tiers

`redis` is a disposable cache with `allkeys-lru`. `redis-jobs` is a durable Celery broker/result store with AOF, periodic snapshots, and `noeviction`. They use separate passwords (`REDIS_PASSWORD` and `CELERY_REDIS_PASSWORD`), separate databases, and separate memory limits.

Back up the `celery_redis_data` volume when queued work must survive a host loss. Cache data does not require backup. Monitor job Redis memory and increase `REDIS_JOBS_MAXMEMORY` before it reaches the limit; `noeviction` intentionally rejects new writes instead of silently losing queued work.

Keep PostgreSQL and Redis ports on their default loopback bindings. Public access should terminate at a TLS reverse proxy in front of the frontend and backend only.
