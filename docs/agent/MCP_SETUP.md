# MCP Setup

QuantDinger's MCP server wraps the Agent Gateway and keeps the REST API as the source of truth.

## Install

```bash
pip install "quantdinger-mcp==0.5.0"
```

For repository development, use `pip install -e ./mcp_server` instead.

Set these environment variables in the MCP client process:

```text
QUANTDINGER_BASE_URL=http://localhost:8888
QUANTDINGER_AGENT_TOKEN=qd_agent_xxx
```

Run `quantdinger-mcp` for the default `stdio` transport. For network use, set `QUANTDINGER_MCP_TRANSPORT=sse` or `streamable-http`, plus optional `QUANTDINGER_MCP_HOST` and `QUANTDINGER_MCP_PORT`.

Non-loopback network listeners are denied by default unless `QUANTDINGER_MCP_AUTH_TOKEN` is set to a distinct secret of at least 32 characters. Configure `QUANTDINGER_MCP_PUBLIC_URL` to the externally reachable HTTPS base URL and send the inbound token as a Bearer token from the MCP client. The inbound token protects the MCP listener; `QUANTDINGER_AGENT_TOKEN` remains the separate upstream tenant credential.

## Client configuration

A typical stdio client entry is:

```json
{
  "mcpServers": {
    "quantdinger": {
      "command": "quantdinger-mcp",
      "env": {
        "QUANTDINGER_BASE_URL": "http://localhost:8888",
        "QUANTDINGER_AGENT_TOKEN": "qd_agent_xxx"
      }
    }
  }
}
```

Keep the configuration private. Prefer an environment-secret facility when the client supports one.

## Expected workflow

- Use market tools for discovery and OHLCV reads.
- Use universe, factor, and watchlist tools for research workspace inputs.
- Use indicator tools only for chart artifacts.
- Compile Strategy API V2 code before saving it.
- Save private Strategy API V2 sources and inspect version history.
- Create stopped deployments from saved V2 source IDs.
- Use bounded job polling or SSE streaming for backtest results.
- Use broker and execution observation tools only for reads; credential secrets are never exposed.
- Use N-scope signal-alert tools for notification automation.
- Supply a unique `idempotency_key` to every mutating W/B/N/T tool.
- Require explicit confirmation before restoring source versions, cancelling jobs, deleting alerts, delivering notifications, stopping runtime state, emergency-stopping trading, or placing any order.

The strategy manifest owns market, instrument, frequency, warmup, dependency, and leverage scope. Do not pass those as alternate backtest fields.

For the exact tool signatures, see [the MCP package README](../../mcp_server/README.md) and [Agent OpenAPI](agent-openapi.json).
