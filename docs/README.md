# QuantDinger Documentation

This directory contains maintained documentation for the current QuantDinger
release. Start with the [project README](../README.md) or the
[Chinese project README](README_CN.md).

## Architecture and contracts

| Document | Purpose |
| --- | --- |
| [Architecture](architecture/ARCHITECTURE.md) | Backend ownership map and contributor design rules. |
| [Module boundaries](architecture/MODULE_BOUNDARIES.md) | Dependency direction and package responsibilities. |
| [Concurrency model](architecture/CONCURRENCY_MODEL.md) | Database, worker, and thread ownership rules. |
| [Process roles](architecture/PROCESS_ROLES_AND_TASKS.md) | API, trading, scheduler, Celery, and migration boundaries. |
| [API conventions](architecture/API_CONVENTIONS.md) | Human API envelopes, authentication, and stability classes. |
| [Extension guide](architecture/EXTENSION_GUIDE.md) | How to add routes, services, adapters, and tasks safely. |

## Deployment and operations

| Document | Purpose |
| --- | --- |
| [Production hardening](deployment/PRODUCTION_HARDENING.md) | Locked runtime and production preflight. |
| [Observability](deployment/OBSERVABILITY.md) | Prometheus, Grafana, Alertmanager, and exporters. |
| [Installation troubleshooting](deployment/INSTALL_TROUBLESHOOTING.md) | Docker, mirrors, ports, and PostgreSQL problems. |
| [Cloud deployment (English)](deployment/CLOUD_DEPLOYMENT_EN.md) | Reverse proxy and cloud deployment. |
| [云部署（中文）](deployment/CLOUD_DEPLOYMENT_CN.md) | 反向代理与云部署。 |
| [Multi-user setup](deployment/MULTI_USER_SETUP.md) | Roles and multi-user deployment. |
| [OAuth (English)](deployment/OAUTH_CONFIG_EN.md) | Google and GitHub OAuth configuration. |
| [OAuth（中文）](deployment/OAUTH_CONFIG_CN.md) | Google 与 GitHub OAuth 配置。 |
| [USDT payment](deployment/USDT_PAYMENT_GUIDE.md) | Optional USDT billing configuration. |

Notification configuration:

- [Email (English)](deployment/NOTIFICATION_EMAIL_CONFIG_EN.md) /
  [邮件（中文）](deployment/NOTIFICATION_EMAIL_CONFIG_CN.md)
- [SMS (English)](deployment/NOTIFICATION_SMS_CONFIG_EN.md) /
  [短信（中文）](deployment/NOTIFICATION_SMS_CONFIG_CN.md)
- [Telegram (English)](deployment/NOTIFICATION_TELEGRAM_CONFIG_EN.md) /
  [Telegram（中文）](deployment/NOTIFICATION_TELEGRAM_CONFIG_CN.md)

## Trading and research

| Document | Purpose |
| --- | --- |
| [Indicator guide](trading/INDICATOR_DEV_GUIDE.md) | Chart-only Python indicator contract. |
| [指标指南](trading/INDICATOR_DEV_GUIDE_CN.md) | 图表指标开发契约。 |
| [Strategy guide](trading/STRATEGY_DEV_GUIDE.md) | Strategy API V2, risk, backtest, and live execution. |
| [策略指南](trading/STRATEGY_DEV_GUIDE_CN.md) | 脚本策略、风控、回测与实盘。 |
| [Public universes and fundamentals](trading/PUBLIC_UNIVERSE_AND_FUNDAMENTALS_CN.md) | 股票池来源、时点数据和已知限制。 |
| [IBKR guide](trading/IBKR_TRADING_GUIDE_EN.md) | IBKR connectivity and trading workflow. |

Runnable examples are in [`examples/`](examples/).

## API and AI agents

- [`api/`](api/) contains the generated human OpenAPI contract and ReDoc viewer.
- [`agent/`](agent/) contains Agent Gateway, MCP, security, and integration docs.
- [`mcp_server/`](../mcp_server/) contains the standalone MCP package.

## Maintenance policy

- Keep one maintained document for each operational or architectural concern.
- Put implementation decisions in architecture or contract documents, not in
  one-off planning files.
- Do not commit temporary audits, validation snapshots, generated screenshots,
  or completed roadmaps. Git history already preserves them.
- Update links and the relevant index in the same change when a document moves.
- Keep generated OpenAPI artifacts committed because CI checks them.
