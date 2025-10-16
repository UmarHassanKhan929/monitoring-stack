## Monitoring Stack (Prometheus, Grafana, Loki, Promtail, Node Exporter)

This repository provides a ready-to-run observability stack using Docker Compose:

- Prometheus for metrics collection
- Grafana for visualization
- Loki for log aggregation
- Promtail to ship container logs to Loki
- Node Exporter for host metrics

All services are pre-wired with sane defaults, provisioning, and volumes for persistence.

### Components and Ports

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (username: `admin`, password: `admin`)
- Loki (API): http://localhost:3100
- Promtail: http://localhost:9080 (metrics)
- Node Exporter: http://localhost:9100/metrics

### Repository Layout

```
docker-compose.yml
grafana/
  dashboards/
    node-exporter-full.json
  provisioning/
    dashboards/dashboard.yml
    datasources/datasource.yml
prometheus/
  prometheus.yml
promtail/
  promtail-config.yml
```

Grafana is auto-provisioned with:
- A Prometheus data source at `http://prometheus:9090`
- A Loki data source at `http://loki:3100`
- Any JSON dashboards placed in `grafana/dashboards` (e.g., `node-exporter-full.json`)

---

## Quick Start

Prerequisites: Docker Desktop (Windows/macOS) or Docker Engine + Compose.

```bash
docker compose up -d
```

Then open:
- Grafana: http://localhost:3000 (admin/admin)
- Prometheus: http://localhost:9090

To stop:

```bash
docker compose down
```

Data is persisted in Docker volumes (`prometheus_data`, `grafana_data`, `loki_data`).

---

## Integrating Your Backend (Python, Node.js, Go)

There are two parts to integrate:
1) Expose metrics for Prometheus to scrape
2) Ship logs to Loki (via Promtail or directly)

On Windows with Docker Desktop, use `host.docker.internal` for Prometheus to reach apps running on your host. If your app runs as a container on the same Compose network, use the container service name instead.

### 1) Expose Metrics for Prometheus

Update `prometheus/prometheus.yml` to add a job for your app and point it at your metrics endpoint. Example for an app running on your host on port 8000:

```yaml
# prometheus/prometheus.yml
- job_name: 'my-backend'
  static_configs:
    - targets: ['host.docker.internal:8000']
```

If your app runs as a service in the same Compose network, use `service-name:PORT`.

#### Python (FastAPI/Flask/any WSGI)

Install:

```bash
pip install prometheus-client
```

Expose `/metrics` (example using a standalone HTTP server on port 8000):

```python
from prometheus_client import start_http_server, Counter
import time

requests_total = Counter("requests_total", "Total HTTP requests")

if __name__ == "__main__":
    start_http_server(8000)  # Exposes /metrics on :8000
    while True:
        requests_total.inc()
        time.sleep(5)
```

For FastAPI or Flask, you can use `prometheus_client` middleware or frameworks like `prometheus-fastapi-instrumentator`.

#### Node.js (Express)

Install:

```bash
npm i prom-client express
```

Expose `/metrics`:

```javascript
const express = require('express');
const client = require('prom-client');

const app = express();
client.collectDefaultMetrics();

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.listen(8000, () => console.log('metrics on :8000/metrics'));
```

#### Go (net/http)

Install:

```bash
go get github.com/prometheus/client_golang/prometheus/promhttp
```

Expose `/metrics`:

```go
package main

import (
    "log"
    "net/http"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
    http.Handle("/metrics", promhttp.Handler())
    log.Println("metrics on :8000/metrics")
    log.Fatal(http.ListenAndServe(":8000", nil))
}
```

After your app exposes metrics, add the job in `prometheus/prometheus.yml` (as above), then restart Prometheus:

```bash
docker compose restart prometheus
```

In Grafana, import/create dashboards and select the "Prometheus" data source.

---

### 2) Ship Logs to Loki

This stack already ships Docker container logs via Promtail using:

```yaml
# promtail/promtail-config.yml
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: containers
    static_configs:
      - targets: [localhost]
        labels:
          job: containerlogs
          __path__: /var/lib/docker/containers/*/*log
    pipeline_stages:
      - docker: {}
      - labelmap:
          regex: __meta_docker_container_label_(.+)
      - match:
          selector: '{grafana_job=""}'
          action: drop
```

So if your backend runs as a container in the same Docker Engine, logging to stdout/stderr is enough. Promtail will forward those logs to Loki, and you can query them in Grafana using the "Loki" data source.

If your backend runs on the host (outside Docker), you have two options:

- Run a local Promtail to tail your app's log files and push to `http://localhost:3100`
- Use a Loki client library and push logs directly to `http://localhost:3100/loki/api/v1/push`

#### Python logging to Loki (direct)

```bash
pip install python-logging-loki
```

```python
import logging
from logging_loki import LokiHandler

handler = LokiHandler(
    url="http://localhost:3100/loki/api/v1/push",
    tags={"app": "my-python-app"},
)

logger = logging.getLogger("my-python-app")
logger.setLevel(logging.INFO)
logger.addHandler(handler)
logger.info("hello from python")
```

#### Node.js logging to Loki (direct)

Winston example:

```bash
npm i winston winston-loki
```

```javascript
const winston = require('winston');
const LokiTransport = require('winston-loki');

const logger = winston.createLogger({
  transports: [
    new LokiTransport({ host: 'http://localhost:3100', labels: { app: 'my-node-app' } })
  ]
});

logger.info('hello from node');
```

#### Go logging to Loki (direct)

```bash
go get github.com/grafana/loki-client-go/loki
```

```go
package main

import (
    "context"
    "log"
    "time"
    "github.com/grafana/loki-client-go/loki"
)

func main() {
    cfg := loki.Config{URL: "http://localhost:3100/loki/api/v1/push"}
    client, err := loki.New(cfg)
    if err != nil { log.Fatal(err) }
    defer client.Close()

    labels := map[string]string{"app": "my-go-app"}
    err = client.Handle(context.Background(), time.Now(), labels, "hello from go")
    if err != nil { log.Fatal(err) }
}
```

---

## Common Tasks

- Restart a single service:

```bash
docker compose restart grafana
```

- View logs for a service:

```bash
docker compose logs -f promtail
```

- Add another dashboard: drop a JSON file into `grafana/dashboards/` and refresh Grafana.

---

## Troubleshooting

- No metrics from your app:
  - Verify your app exposes `/metrics` locally
  - Ensure `prometheus/prometheus.yml` has a job pointing to `host.docker.internal:PORT` (Windows/macOS) or the correct container service name
  - Restart Prometheus after changing the config

- No logs from your app:
  - If containerized, ensure it writes to stdout/stderr
  - If on host, either tail files with a Promtail agent or use a Loki client library
  - Check Promtail logs: `docker compose logs -f promtail`

- Grafana empty panels:
  - Confirm the correct data source is selected (Prometheus or Loki)
  - Check data source health under Grafana > Connections > Data sources

---

## Security Notes

- Grafana admin credentials are set via environment variables in `docker-compose.yml` and default to `admin/admin`. Change them in production.
- Consider network/ingress controls and TLS for external exposure.


