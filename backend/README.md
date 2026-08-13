# QuantDinger Python Backend

This directory contains the Flask API, domain services, background workers,
database migrations, OpenAPI integration, and backend tests for QuantDinger.

Start with the [project README](../README.md) for product installation. This
document is the backend contributor and operator quick reference.

## Runtime model

The production deployment reuses one backend image across independent process
roles:

| Role | Command | Ownership |
| --- | --- | --- |
| API | `gunicorn -c gunicorn_config.py run:app` | HTTP, authentication, validation, and durable command submission. |
| Migration | `python -m app.commands.migrate` | Fail-fast schema application before services start. |
| Trading | `python -m app.commands.trading_worker` | Strategy runtimes, pending orders, broker sessions, and reconciliation. |
| Scheduler | `python -m app.commands.scheduler` | Portfolio, deployment, payment, and signal schedules. |
| Celery worker | `celery -A app.celery_app:celery_app worker` | Finite AI, backtest, experiment, report, and maintenance jobs. |
| Celery beat | `celery -A app.celery_app:celery_app beat` | Periodic task dispatch. |

HTTP processes must not start trading or scheduler threads. Celery must not own
long-lived strategy loops or broker sessions. See
[Process roles and durable tasks](../docs/architecture/PROCESS_ROLES_AND_TASKS.md).

## Storage model

- PostgreSQL 18 is the system of record.
- `redis` is an evictable application cache.
- `redis-jobs` is the durable Celery broker/result tier with AOF and
  `noeviction`.
- Strategy ownership uses PostgreSQL commands, leases, fencing tokens, and
  worker heartbeats.

Queue state must never share the cache Redis eviction policy.

## Directory map

```text
app/
  commands/             Process entry points and operational commands
  config/               Environment-backed configuration
  data_providers/       Aggregated market and global data providers
  data_sources/         Raw market data adapters
  observability/        Metrics and request instrumentation
  openapi/              Human API schemas, registration, and export metadata
  routes/               HTTP facades and compatibility routes
  services/             Domain workflows and integration orchestration
  tasks/                Celery task definitions
  utils/                Small infrastructure helpers
migrations/             PostgreSQL schema and incremental migrations
scripts/                Backend quality, export, and production checks
tests/                  Unit, integration, contract, and release-gate tests
```

Read [Backend architecture](../docs/architecture/ARCHITECTURE.md) and
[Module boundaries](../docs/architecture/MODULE_BOUNDARIES.md) before larger changes.

## Configuration

Create the runtime environment file:

```bash
cp env.example .env
```

At minimum, replace these values before a shared or production deployment:

```dotenv
SECRET_KEY=<independent-random-value-at-least-10-bytes-32-plus-recommended>
CREDENTIAL_ENCRYPTION_KEY=<independent-random-value-at-least-32-bytes>
ADMIN_USER=<initial-admin-name>
ADMIN_PASSWORD=<strong-initial-password>
```

Generate each secret independently:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

Docker-level database, Redis, Grafana, port, image, and resource settings belong
in the repository-root `.env`; application runtime settings belong here.

## Docker workflow

Run these commands from the repository root.

Core development stack:

```bash
docker compose up -d --build
docker compose ps
```

Production-hardened stack with optional monitoring:

```bash
python backend/scripts/check_production_config.py \
  --env-file .env \
  --env-file backend/.env

docker compose \
  -f docker-compose.yml \
  -f docker-compose.production.yml \
  -f docker-compose.observability.yml \
  up -d --build
```

The production overlay uses UID/GID `10001`, a read-only root filesystem,
dropped capabilities, bounded temporary filesystems, and CPU/memory limits.
Remove the observability overlay when monitoring is provided elsewhere.

## Local Python workflow

Prerequisites:

- Python 3.12;
- PostgreSQL 18;
- Redis 8 for cache and Celery-backed workflows.

Create an environment and install development dependencies:

```bash
python -m venv .venv
python -m pip install --upgrade pip
python -m pip install -r requirements.lock -r requirements-dev.txt
```

Apply migrations:

```bash
QD_PROCESS_ROLE=migration python -m app.commands.migrate
```

Windows PowerShell equivalent:

```powershell
$env:QD_PROCESS_ROLE = "migration"
python -m app.commands.migrate
```

Start the API for local debugging:

```bash
python run.py
```

Use the Docker process model when testing trading, scheduler, or Celery ownership
boundaries. A single local API process is not a substitute for production role
separation.

## Health and operations

| Endpoint | Purpose |
| --- | --- |
| `GET /` | Application identity and resolved version. |
| `GET /api/health` | Basic liveness. |
| `GET /api/health/ready` | PostgreSQL and Celery broker readiness. |
| `GET /api/health/workers` | Trading, scheduler, and Celery heartbeat summary. |
| `GET /metrics` | Prometheus metrics. Keep this private. |

Container logs default to structured JSON and include process role and request
ID. The optional monitoring stack is documented in
[Observability](../docs/deployment/OBSERVABILITY.md).

## API contracts

Human API routes use the existing `/api/...` surface. AI agents use the scoped
`/api/agent/v1/...` gateway with a separate contract.

- Human API conventions: [API_CONVENTIONS.md](../docs/architecture/API_CONVENTIONS.md)
- Committed OpenAPI: [openapi.yaml](../docs/api/openapi.yaml)
- Agent contract: [agent-openapi.json](../docs/agent/agent-openapi.json)

Regenerate the human API artifact after schema or route metadata changes:

```bash
python scripts/export_openapi.py
```

With `OPENAPI_ENABLED=true`, interactive documentation is available at:

- Swagger UI: <http://127.0.0.1:5000/api/docs/swagger>
- ReDoc: <http://127.0.0.1:5000/api/docs/redoc>

## Quality checks

Run the normal backend suite:

```bash
python -m compileall -q app scripts tests
ruff check app scripts tests
python scripts/backend_quality_check.py
python scripts/check_requirements_lock.py
python -m pytest -m "not integration and not stress" --ignore=tests/release_gate -q
```

Run release gates independently:

```bash
python -m pytest tests/release_gate/test_cta_backtest_release_gate.py -q
python -m pytest tests/release_gate/test_live_execution_release_gate.py -q
python -m pytest tests/release_gate/test_robot_strategy_unification.py -q
```

Security CI additionally runs `pip-audit`, Bandit, Gitleaks, and CodeQL.

## Contributor rules

- Keep routes focused on parsing, authentication, service calls, and response
  mapping.
- Keep Flask request objects out of domain services.
- Put exchange-specific normalization and errors near the adapter.
- Make state mutations idempotent and define their retry behavior.
- Keep code comments, docstrings, logs, and internal errors in English.
- Preserve existing paths and response fields unless an intentional contract
  change is documented and tested.
- Add finite retryable work to Celery; keep long-lived ownership in the trading
  or scheduler process.

## Versioning

The local fallback version is read from [`VERSION`](VERSION). Release builds can
inject `APP_VERSION` from a Git tag. Both `v5.0.1` and `5.0.1` normalize to the
API display value `5.0.1`.

Update checked-in version declarations from the repository root:

```bash
python scripts/bump_version.py 5.0.1
python scripts/check_version.py
```

## License

Apache License 2.0. See the repository-root [LICENSE](../LICENSE).
