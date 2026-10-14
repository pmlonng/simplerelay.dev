---
title: "Configuration Reference"
description: "Complete list of CLI flags, environment variables, and configuration file directives for Simple Relay."
section: "Reference"
order: 9
---

Simple Relay adheres to the 12-factor application methodology. Configuration options can be supplied via command-line flags, environment variables, or a YAML configuration file.

Environment variables take precedence over configuration files, and command-line flags take precedence over environment variables.

---

## Core Settings Matrix

| CLI Flag | Environment Variable | Default | Description |
| :--- | :--- | :--- | :--- |
| `--listen` | `RELAY_LISTEN_ADDR` | `:8080` | TCP host and port to bind the HTTP server. |
| `--db-path` | `RELAY_DB_PATH` | `./relay.db` | Path to persistent storage database file. |
| `--workers` | `RELAY_WORKERS` | `4` | Number of concurrent background delivery workers. |
| `--max-retries` | `RELAY_MAX_RETRIES` | `5` | Default maximum retry attempts before transitioning to DLQ. |
| `--initial-interval`| `RELAY_INITIAL_INTERVAL_MS`| `1000` | Initial backoff delay in milliseconds. |
| `--max-interval` | `RELAY_MAX_INTERVAL_MS` | `60000` | Maximum backoff cap in milliseconds. |
| `--log-level` | `RELAY_LOG_LEVEL` | `info` | Logging verbosity: `debug`, `info`, `warn`, `error`. |
| `--config` | `RELAY_CONFIG_FILE` | `""` | Path to optional YAML configuration file. |

---

## Configuration File Structure (`relay.yaml`)

```yaml
server:
  listen: ":8080"
  db_path: "/data/relay.db"
  log_level: "info"

workers:
  pool_size: 8
  batch_flush_interval_ms: 50

routes:
  - name: "stripe-inbound"
    path: "/api/v1/webhook/stripe"
    destination: "http://internal-backend:8000/webhooks/stripe"
    signature:
      provider: "stripe"
      secret_env: "STRIPE_WEBHOOK_SECRET"
      tolerance_seconds: 300
    retries:
      max_attempts: 10
      initial_interval_ms: 1000
      max_interval_ms: 60000

proxy:
  retry_on_status: [429, 500, 502, 503, 504]
  timeout_ms: 15000
  total_timeout_ms: 60000
  respect_retry_after: true
```
