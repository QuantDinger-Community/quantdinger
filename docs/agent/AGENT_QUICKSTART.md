# Agent Gateway Quickstart

QuantDinger exposes a tenant-scoped Agent Gateway at `/api/agent/v1`. Agent tokens are separate from human JWT sessions and enforce capability scopes, market/instrument allowlists, rate limits, expiry, and paper-only restrictions.

The machine-readable contract is [agent-openapi.json](agent-openapi.json). MCP setup is documented in [MCP_SETUP.md](MCP_SETUP.md).

## Authenticate

Create an Agent Token from the human admin UI, store the full token when it is shown once, and send it as a bearer token:

```bash
curl -H "Authorization: Bearer $QUANTDINGER_AGENT_TOKEN" \
  http://localhost:8888/api/agent/v1/whoami
```

Scopes are `R` for reads, `W` for saved artifacts and deployment configuration, `B` for backtests, `N` for notification side effects, and `T` for runtime or order mutations. `C` is admin-only. Token permissions never bypass server-side live-trading controls.

Every mutating W/B/N/T request requires a unique `Idempotency-Key` header. Reuse the same key only when retrying the exact same method, route, query, and body. The gateway atomically reserves the key, returns a stored completed response on replay, and rejects concurrent or mismatched reuse.

## Strategy API V2

Executable strategies use Strategy API V2. Code defines `initialize(context)`, declares its universe and subscriptions, and provides `handle_data`, `on_rebalance`, or a scheduled callback. Markets, instruments, frequencies, dependencies, warmup, and leverage policy come from the compiled manifest.

The Agent Gateway exposes the complete source lifecycle:

1. List starter code with `GET /strategy-sources/templates`.
2. Compile code with `POST /strategy-sources/compile`.
3. Save a private source with `POST /strategy-sources`.
4. Inspect or update it through `/strategy-sources/{source_id}`.
5. Review immutable snapshots through `/strategy-sources/{source_id}/versions`.
6. Create a stopped deployment from the saved source id.

Source restoration requires an explicit `confirm=true` request and creates another immutable snapshot.

Create a stopped deployment from a saved source:

```bash
curl -X POST http://localhost:8888/api/agent/v1/strategies \
  -H "Authorization: Bearer $QUANTDINGER_AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: deploy-spy-trend-v1" \
  -d '{
    "name": "spy-trend",
    "sourceId": 12,
    "initialCapital": 10000,
    "executionMode": "signal",
    "leverageEnabled": false,
    "params": {"lookback": 50}
  }'
```

Update the same canonical fields with `PATCH /api/agent/v1/strategies/{id}`. Starting a deployment is intentionally not part of the W-scope configuration endpoint. A T-scope token can stop a running deployment through `/strategies/{id}/stop`.

## Backtests

Backtests accept Strategy API V2 code and run asynchronously:

```bash
curl -X POST http://localhost:8888/api/agent/v1/backtest/run \
  -H "Authorization: Bearer $QUANTDINGER_AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: spy-trend-2025" \
  -d '{
    "code": "def initialize(context):\n    g.symbol = \"USStock:SPY\"\n    context.set_universe([g.symbol])\n    context.subscribe(frequency=\"1d\")\n\ndef handle_data(context, data):\n    pass",
    "startDate": "2025-01-01",
    "endDate": "2025-12-31",
    "initialCapital": 10000,
    "leverageEnabled": false,
    "params": {}
  }'
```

Poll `/api/agent/v1/jobs/{job_id}` or consume `/api/agent/v1/jobs/{job_id}/stream`. Reuse an idempotency key when retrying the same submission.
Cancel a queued or running job with `POST /jobs/{job_id}/cancel` using B scope, confirmation at the MCP layer, and a new idempotency key. A running worker may finish its local computation, but it cannot overwrite the durable cancelled state.

## Indicators

Indicators are chart-only. Fetch `/indicators/authoring-contract`, validate with `/indicators/validate`, and save through `/indicators`. Indicator code cannot be passed to the backtest endpoint; convert the trading idea to Strategy API V2 first.

## Research, broker observations, and notifications

Research tools expose point-in-time universes, factor metadata, and the tenant watchlist under `/research/*`. Broker observation endpoints under `/trading/*` return safe credential metadata, account snapshots, account/strategy positions, pending orders, and cursor-paginated trade ledgers. They never return decrypted API keys, secrets, passphrases, or encrypted credential blobs.

N-scope signal-alert endpoints under `/notifications/signal-alerts` reuse the existing indicator notification service. Immediate evaluation requires explicit MCP confirmation because it may deliver a notification.

## Runtime and orders

`GET /runtime/overview` returns compact tenant runtime state. Quick orders require T scope and an `Idempotency-Key`. Live execution additionally requires a live-capable token, server live-trading enablement, a credential reference, client-side explicit confirmation, and compliance with the token's `max_order_notional` and `max_daily_notional` caps.

The emergency stop at `/quick-trade/kill-switch` requires `confirm=true`. It attempts to cancel open agent-originated live orders, cancels open paper orders, revokes all active T-scope tokens for the tenant, and returns any exchange cancellation failures for mandatory human review.

Rate limiting is shared through Redis across API workers and enforces both token and tenant quotas. Responses include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`; `429` responses also include `Retry-After`.

Never log tokens or credential material. Treat redacted values as terminal and do not attempt to reconstruct them.
