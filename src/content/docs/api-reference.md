---
title: "REST API Reference"
description: "Comprehensive REST API endpoint specifications for webhook ingestion, proxying, DLQ, and system health."
section: "Reference"
order: 10
---

Simple Relay exposes a lightweight REST API for ingesting webhooks, routing proxies, inspecting deliveries, and replaying failed requests.

---

## 1. Webhook Ingestion

### `POST /api/v1/webhook/:route_name`
Ingests an inbound webhook, verifies cryptographic signatures (if configured), commits raw bytes to storage, and queues background delivery.

- **Status Code:** `200 OK` on durable persistence; `401 Unauthorized` on signature failure; `500 Internal Error` on write failure.
- **Response Format:**
  ```json
  {
    "status": "queued",
    "request_id": "01j9x0m1p00000000000000000"
  }
  ```

---

## 2. Outbound Proxy

### `ANY /api/v1/proxy/sync?url={upstream_url}`
Synchronously forwards incoming HTTP request to `upstream_url`. Retries automatically on `429` and `5xx` with exponential jitter. Cancels immediately if client disconnects.

- **Headers:** Transmits original headers and request body.
- **Returns:** Transparent upstream status code, headers, and body.

### `ANY /api/v1/proxy/async?url={upstream_url}`
Fire-and-forget outbound HTTP request. Commits request to storage and guarantees at-least-once delivery.

- **Status Code:** `202 Accepted`
- **Response Format:**
  ```json
  {
    "status": "accepted",
    "request_id": "01j9x0z87q0000000000000000"
  }
  ```

---

## 3. Deliveries & Dead Letter Queue (DLQ)

### `GET /api/v1/requests`
List recorded requests with pagination and status filters.

- **Query Parameters:**
  - `status`: Filter by `queued`, `delivering`, `delivered`, `failed`.
  - `route`: Filter by route name.
  - `limit`: Number of records (default: `50`, max: `200`).
  - `cursor`: ULID cursor for monotonic pagination.

### `GET /api/v1/requests/:id`
Fetch single delivery record including headers, payload, attempt logs, error details, and round-trip latencies.

### `POST /api/v1/requests/:id/replay`
Manually requeues a failed delivery from the DLQ for immediate re-attempt.

### `POST /api/v1/dlq/replay`
Bulk requeues all failed deliveries in the DLQ matching filter criteria.

---

## 4. System & Health

### `GET /healthz`
Health check endpoint returning binary version and operational status.

```json
{
  "status": "ok",
  "version": "1.0.0"
}
```
