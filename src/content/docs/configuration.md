---
title: "Configuration Reference"
description: "Complete list of CLI flags and environment variables for Simple Relay."
section: "Reference"
order: 9
---

Simple Relay adheres to the 12-factor application methodology. Configuration options are supplied via command-line flags and environment variables. Command-line flags take precedence over environment variables, which override default values.

---

## Core Settings Matrix

| CLI Flag | Environment Variable | Default | Description |
| :--- | :--- | :--- | :--- |
| `-host` | `RELAY_HOST` | `0.0.0.0` | Server host address. |
| `-port` | `RELAY_PORT` (or `PORT`) | `8080` | Server HTTP port. |
| `-db` | `RELAY_DB_PATH` | `relay.db` | Path to persistent SQLite database file. |
| `-log-level` | `RELAY_LOG_LEVEL` | `info` | Logging verbosity: `debug`, `info`, `warn`, `error`. |
| `-workers` | `RELAY_WORKER_CONCURRENCY` | `20` | Outbound worker pool concurrency. |
| `-dispatcher-buffer` | `RELAY_DISPATCHER_BUFFER` | `0` (dynamic: `2 * workers`) | Dispatcher jobs channel buffer capacity. |
| `-dispatcher-poll-ms` | `RELAY_DISPATCHER_POLL_MS` | `500` | Dispatcher polling interval in milliseconds. |
| `-cleaner-interval` | `RELAY_CLEANER_INTERVAL` | `1h` | Storage retention cleaner execution interval. |
| `-drain-timeout` | `RELAY_DRAIN_TIMEOUT` | `30s` | Graceful shutdown drain timeout. |
| — | `RELAY_RETENTION_SUCCESS_DAYS` | `7` | Retention period in days for successfully delivered records. |
| — | `RELAY_RETENTION_FAILED_DAYS` | `30` | Retention period in days for failed/dead-lettered records. |
| — | `RELAY_ADMIN_PASSWORD` | `""` | Optional admin authentication secret for API endpoints. |
| — | `RELAY_BATCH_SIZE` | `50` | Maximum requests grouped per SQLite WAL write transaction. |
| — | `RELAY_BATCH_TIMEOUT_MS` | `10` | Maximum batch linger duration before flushing to SQLite. |

---

## Dynamic Route Configuration

Routes in Simple Relay are managed dynamically at runtime via the REST API (`/api/v1/routes`) or the embedded Web UI, avoiding static configuration files and service restarts. Routes are persisted in SQLite and hot-reloaded into memory.

### Route Creation Example (`POST /api/v1/routes`)

```bash
curl -X POST http://localhost:8080/api/v1/routes \
  -H "Content-Type: application/json" \
  -d '{
    "name": "stripe-inbound",
    "type": "inbound",
    "mode": "async",
    "path": "/api/v1/webhook/stripe",
    "destination_url": "http://internal-backend:8000/webhooks/stripe",
    "signature_provider": "stripe",
    "signature_secret": "whsec_...",
    "max_attempts": 10,
    "timeout_ms": 15000,
    "retry_backoff_strategy": "exp"
  }'
```
