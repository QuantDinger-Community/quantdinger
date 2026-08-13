# QuantDinger â€” Development Guide

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Docker & Docker Compose | 20+ | required for the default setup |
| Python | 3.10+ | only if running backend outside Docker |
| Node.js | 22 LTS | only if you maintain the private QuantDinger-Vue or mobile source repos |

## Quick Start (Docker)

```bash
# 1. Clone
git clone https://github.com/<your-org>/quantdinger.git
cd quantdinger

# 2. Configure
cp backend/env.example backend/.env
# Edit .env â€” at minimum set SECRET_KEY to a random value:
#   SECRET_KEY=$(python3 -c "import secrets; print(secrets.token_hex(32))")
# Optional: project-root `.env` with `IMAGE_PREFIX` if Docker Hub pulls are slow (see .env.example).

# 3. Launch
docker compose up -d --build

# 4. Open http://localhost:8888
```

The stack includes:

| Service | Port | Description |
|---------|------|-------------|
| `frontend` | 8888 | Nginx serving Vue SPA |
| `backend` | 5000 | Flask API (gunicorn) |
| `postgres` | 5432 | PostgreSQL 18 |
| `redis` | 6379 | Cache layer (LRU, 128 MB) |

## Project Structure

```
quantdinger/
â”œâ”€â”€ backend/          # Flask API
â”‚   â”œâ”€â”€ app/
â”‚   â”‚   â”œâ”€â”€ config/              # Settings, API keys, DB config
â”‚   â”‚   â”œâ”€â”€ data_providers/      # Market data fetchers (crypto, forex, â€¦)
â”‚   â”‚   â”œâ”€â”€ data_sources/        # Exchange/broker adapters (CCXT, yfinance, â€¦)
â”‚   â”‚   â”œâ”€â”€ routes/              # Flask Blueprints (REST endpoints)
â”‚   â”‚   â”œâ”€â”€ services/            # Business logic (strategy, trading, AI, â€¦)
â”‚   â”‚   â””â”€â”€ utils/               # DB helpers, auth, caching, logger
â”‚   â”œâ”€â”€ migrations/              # SQL schema + seed data
â”‚   â”œâ”€â”€ gunicorn_config.py       # Production WSGI config
â”‚   â”œâ”€â”€ run.py                   # App entrypoint
â”‚   â”œâ”€â”€ Dockerfile
â”‚   â””â”€â”€ requirements.txt
â”œâ”€â”€ docs/                        # Changelog, architecture notes
â”œâ”€â”€ docker-compose.yml           # frontend service pulls ghcr.io/.../quantdinger-frontend
â”œâ”€â”€ docker-compose.ghcr.yml      # backend, frontend, and mobile pulled from GHCR (zero-clone deploy)
â””â”€â”€ README.md
```

## Running Backend Locally (without Docker)

```bash
cd backend
python -m venv .venv && source .venv/bin/activate  # or .venv\Scripts\activate on Windows
pip install -r requirements.txt
cp env.example .env   # edit .env
python run.py
```

The dev server starts on `http://localhost:5000` with auto-reload.

## Frontend (private Vue repository)

The open-source tree **does not** contain Vue source or build artefacts. UI work happens in the private **QuantDinger-Vue** repo. Releases are fully automated:

```bash
# In QuantDinger-Vue
git tag v5.0.1
git push origin v5.0.1
```

The `release-frontend.yml` workflow there builds `linux/amd64 + linux/arm64` images via buildx and pushes them to `ghcr.io/openbyteinc/quantdinger-frontend`, tagged with the semver value, `{major}.{minor}`, and `latest`.

This repo's `docker-compose.yml` (and `docker-compose.ghcr.yml`) references that image by default. To pin a version while testing:

```bash
# Project-root .env (sibling of docker-compose.yml)
# Common: bump both sides together
echo "IMAGE_TAG=5.0.1" >> .env

# Or pin only frontend (testing a UI hotfix against stable backend)
# echo "FRONTEND_TAG=5.0.1" >> .env

docker compose pull frontend
docker compose up -d frontend
```

Resolution order: `FRONTEND_TAG` (or `BACKEND_TAG`) â†’ `IMAGE_TAG` â†’ `latest`.

The container reads `BACKEND_URL` at start time and substitutes it into the nginx config via the official `nginx:alpine` envsubst hook, so the same image works for compose, Railway, and direct `docker run`.

### Building frontend from local source

For iterating on Vue source (theme tweaks, debugging, customised UI), drop the source into the gitignored `./QuantDinger-Vue/` slot at the repo root and layer the `docker-compose.build.yml` override on top. The main `docker-compose.yml` only declares `image:` for frontend (so users who only pull never need the directory to exist); the override file adds the `build:` block:

```bash
# Expected layout â€” clone QuantDinger-Vue into ./QuantDinger-Vue/ at this repo root:
#   QuantDinger/
#     QuantDinger-Vue/                <- gitignored; clone goes here
#     backend/
#     docker-compose.yml
#     docker-compose.build.yml        <- enables local frontend build

git clone https://github.com/OpenByteInc/QuantDinger-Vue.git QuantDinger-Vue

# Build frontend from ./QuantDinger-Vue, pull everything else:
docker compose -f docker-compose.yml -f docker-compose.build.yml up -d --build

# Rebuild after editing Vue source:
docker compose -f docker-compose.yml -f docker-compose.build.yml build frontend

# Apply runtime config changes only:
docker compose restart frontend
```

Tip: drop `COMPOSE_FILE=docker-compose.yml:docker-compose.build.yml` into your root `.env` and plain `docker compose up --build` will pick up both files.

Key behaviour:

- **Without the override**: Compose pulls `ghcr.io/.../quantdinger-frontend:<tag>` as usual; `./QuantDinger-Vue/` does not need to exist.
- **With the override**: Compose builds from `./QuantDinger-Vue/` (or `FRONTEND_SRC_PATH` if set) and tags the result with whatever `FRONTEND_TAG` / `IMAGE_TAG` resolves to. The locally built image then satisfies plain `docker compose up` for subsequent runs until you `docker compose pull frontend` to overwrite it.

Source path override (keep the Vue clone somewhere else than `./QuantDinger-Vue/`):

```bash
FRONTEND_SRC_PATH=/abs/path/to/QuantDinger-Vue \
  docker compose -f docker-compose.yml -f docker-compose.build.yml up -d --build
```

Default backend behaviour is unaffected â€” it still builds from this repo's `backend/`. Only the frontend service uses the image/override-build split.

## Adding a New Data Source

1. Create `backend/app/data_sources/<name>.py` implementing a class
   with `get_ticker(symbol)` and `get_kline(symbol, timeframe, limit)`.
2. Register it in `data_sources/factory.py`.
3. If it serves the global market dashboard, add a fetcher in
   `data_providers/` and wire it into the fallback chain.

## Adding a New Exchange (Live Trading)

1. Create `backend/app/services/live_trading/<exchange>.py`
   inheriting from `BaseLiveTrading`.
2. Implement `place_order`, `cancel_order`, `get_balance`, etc.
3. Register in `live_trading/factory.py`.

## Environment Variables

See `backend/env.example` for the full list.  Key variables:

| Variable | Required | Description |
|----------|----------|-------------|
| `SECRET_KEY` | **yes** | JWT signing key â€” must be changed from default |
| `ADMIN_USER` / `ADMIN_PASSWORD` | yes | Initial admin credentials |
| `TWELVE_DATA_API_KEY` | no | Twelve Data for forex/commodities |
| `ADANOS_API_KEY` | no | Optional Adanos Market Sentiment for US stock tickers |
| `OPENAI_API_KEY` or `OPENROUTER_API_KEY` | no | AI analysis features |
| `CACHE_ENABLED` | no | Set `true` to use Redis (auto-set in Docker) |

## Testing

```bash
cd backend
pip install pytest
pytest tests/ -v
```

## Troubleshooting

- **First-time Docker install or pull failures** - see the bilingual [Installation Troubleshooting](docs/deployment/INSTALL_TROUBLESHOOTING.md) guide for Docker Desktop proxy setup, Docker Hub pull errors, and Postgres data-version issues.

- **"apikey parameter is incorrect"** from Twelve Data â€” verify `TWELVE_DATA_API_KEY` in `.env`; Chinese stock data requires a paid plan.
- **Heatmap "æš‚æ— æ•°æ®"** â€” usually caused by NaN in yfinance data; the global JSON encoder now sanitises all NaN/Inf to `null`.
- **Redis connection refused** â€” ensure `redis` service is running (`docker compose up -d redis`); set `CACHE_ENABLED=false` to fall back to in-memory cache.
