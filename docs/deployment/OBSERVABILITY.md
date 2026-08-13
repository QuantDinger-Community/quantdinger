# Observability

The backend exposes Prometheus metrics at `GET /metrics` and propagates a validated `X-Request-ID` header through every HTTP response. Application logs use JSON by default in Docker and include the process role and request ID.

Start the monitoring stack together with the application:

```bash
docker compose -f docker-compose.yml -f docker-compose.observability.yml up -d --build
```

The default local endpoints are:

- Prometheus: `http://127.0.0.1:9090`
- Alertmanager: `http://127.0.0.1:9093`
- Grafana: `http://127.0.0.1:3000`

Grafana provisions the Prometheus datasource and the `QuantDinger Runtime Overview` dashboard automatically. Change `GRAFANA_ADMIN_PASSWORD` before exposing Grafana through a reverse proxy.

The bundled alerts cover API error rate, latency, stale workers, PostgreSQL availability, Redis availability, and job Redis memory pressure. Configure a real notification receiver in `ops/alertmanager/alertmanager.yml` before production use.

Do not expose `/metrics`, Prometheus, Alertmanager, or Grafana directly to the public internet. Keep the default loopback bindings or protect them with a private network and authenticated reverse proxy.
