---
title: "Quickstart"
description: "Get Simple Relay up and running with a single Docker command or standalone binary in under a minute."
section: "Getting Started"
order: 2
---

Get Simple Relay running and test your first reliable HTTP delivery in under 60 seconds.

## 1. Run Simple Relay

### Option A: Using Docker (Recommended)

Run the container in daemon mode, mounting a local folder for persistence:

```bash
docker run -d \
  --name simple-relay \
  -p 8080:8080 \
  -v $(pwd)/data:/data \
  ghcr.io/simple-relay/simple-relay:latest
```

### Option B: Standalone Binary

Download the compiled static binary from GitHub Releases or install via Go:

```bash
# Using Go 1.22+
go install github.com/simple-relay/simple-relay/cmd/relay@latest

# Run the relay
relay --listen :8080 --db-path ./relay.db
```

### Option C: Build from Source (Git Clone)

Clone the repository and compile the binary with standard Go:

```bash
git clone https://github.com/simple-relay/simple-relay.git
cd simple-relay

# Pure Go static build (no CGO required)
CGO_ENABLED=0 go build -o relay ./cmd/relay

# Run the compiled binary
./relay --listen :8080 --db-path ./relay.db
```

Verify that the relay is healthy:

```bash
curl http://localhost:8080/healthz
# {"status":"ok","version":"1.0.0"}
```

---

## 2. Ingest an Inbound Webhook

Simple Relay provides dedicated route endpoints under `/api/v1/webhook/:route`. Send a test payload:

```bash
curl -i -X POST http://localhost:8080/api/v1/webhook/orders \
  -H "Content-Type: application/json" \
  -d '{"order_id": "ord_99812", "amount": 4200, "currency": "usd"}'
```

Response:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"queued","request_id":"01j9x0m1p00000000000000000"}
```

The raw bytes are durably persisted to storage before the `200 OK` response was sent. The delivery dispatcher is now forwarding this payload to your destination endpoint in the background.

---

## 3. Test Outbound Proxy with Smart Retries

Route an outgoing request to an upstream service through Simple Relay's synchronous retry proxy:

```bash
curl -i "http://localhost:8080/api/v1/proxy/sync?url=https://httpbin.org/status/200"
```

If the upstream server returns a `502 Bad Gateway` or `429 Too Many Requests`, Simple Relay automatically catches the failure and retries using exponential backoff with full jitter until the request succeeds or max attempts are reached.

---

## 4. Open the Embedded Dashboard

Open your browser and navigate to:

```text
http://localhost:8080/ui/
```

You can view:
- Inbound and outbound delivery logs.
- Attempt durations and latency graphs.
- Raw request and response payloads.
- Dead Letter Queue (DLQ) entries with one-click re-delivery.

---

## What's Next?

- Configure **[HMAC Signatures](/docs/stripe-github)** for Stripe or GitHub.
- Understand the **[Retries and Dead Letter Queue](/docs/retries-and-dlq)** mechanism.
- Check out the complete **[REST API Reference](/docs/api-reference)**.
