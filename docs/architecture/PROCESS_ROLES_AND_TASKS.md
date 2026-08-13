# Backend Process Roles and Durable Tasks

The production deployment uses one backend image with independent process roles.

| Role | Command | Responsibility |
| --- | --- | --- |
| API | `gunicorn -c gunicorn_config.py run:app` | HTTP, authentication, validation, durable command submission |
| Migration | `python -m app.commands.migrate` | Fail-fast schema application before services start |
| Trading | `python -m app.commands.trading_worker` | Strategy runtimes, pending orders, grid fills, exchange connections |
| Scheduler | `python -m app.commands.scheduler` | Portfolio monitoring, deployment schedules, payment scans, signal alerts |
| Celery Worker | `celery -A app.celery_app:celery_app worker` | AI, backtests, reports, and maintenance jobs |
| Celery Beat | `celery -A app.celery_app:celery_app beat` | Periodic maintenance dispatch |

## Ownership rules

- HTTP processes never start trading or scheduler threads.
- Strategy start, stop, restart, and reconcile requests use `qd_strategy_commands`.
- Trading workers claim commands with PostgreSQL `SKIP LOCKED` semantics.
- A strategy runtime requires a renewable row in `qd_strategy_runtime_leases`.
- Fencing tokens increase when an expired runtime is taken over by another worker.
- Global exchange pollers and scheduler loops use `qd_process_leases` leader ownership.
- Worker health is recorded in `qd_worker_heartbeats`.

## Celery boundary

Celery owns finite jobs that can be serialized, retried, and observed independently:

- fast AI analysis;
- agent backtests;
- reflection and AI calibration;
- market catalog synchronization;
- runtime metadata cleanup.

Celery must not own long-lived strategy loops, exchange polling, broker sessions, or grid runtime state. Those remain in the trading process because they require renewable ownership, reconciliation, and controlled shutdown.

## Redis separation

The cache Redis instance may use an eviction policy. Celery uses `redis-jobs`, which enables AOF persistence and `noeviction`. Queue state must never share an evictable Redis memory policy.

## Deployment sequence

Docker Compose enforces this order:

1. PostgreSQL and both Redis instances become healthy.
2. The migration process exits successfully.
3. API, trading, scheduler, Celery Worker, and Celery Beat start.
4. Health checks require fresh worker heartbeats.

Use these endpoints for operations:

- `/api/health` for liveness;
- `/api/health/ready` for PostgreSQL and Celery Broker readiness;
- `/api/health/workers` for trading, scheduler, and Celery heartbeat summaries.
