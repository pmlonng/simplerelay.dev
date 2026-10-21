---
title: "Recipe: Local Webhook Inspection with Cloudflare Tunnel"
description: "Turn Cloudflare Tunnel (cloudflared) into a durable local webhook inspector with zero-loss server restarts and one-click request replay."
section: "Recipes & Guides"
order: 9
---

When developing webhook integrations locally, engineers frequently rely on **Cloudflare Tunnel (`cloudflared`)** to expose local services to the internet without opening firewall ports.

However, unlike older proprietary tunneling tools (like ngrok's `localhost:4040`), `cloudflared` operates strictly as a network tunnel and does not provide an interactive traffic inspection UI or a replay mechanism. This creates three critical friction points during daily local development:

1. **No Built-in Payload Inspector:** Developers cannot inspect raw request headers, cryptographic signatures, or JSON payloads without adding noisy `console.log` statements to their code.
2. **Dropped Requests on Server Restarts:** When restarting your local backend process (e.g., via `nodemon`, `cargo watch`, or `go run`), any webhooks delivered by the tunnel fail with connection refused errors and are lost.
3. **Inability to Re-run Webhooks:** Testing edge cases or bug fixes requires repeatedly re-triggering events in external vendor dashboards (e.g., repeating test checkout purchases in Stripe).

By placing **Simple Relay** between `cloudflared` and your local development server, you gain a persistent traffic inspector, crash-resilient webhook ingestion, and one-click replay for any historical request.

---

## 1. Traffic Architecture

```
┌─────────────────────────────────┐
│ External Webhook Provider       │ (Stripe, GitHub, Shopify)
└───────────────┬─────────────────┘
                │ Public Internet HTTPS
                ▼
┌─────────────────────────────────┐
│ Cloudflare Tunnel (cloudflared) │ (Exposes 127.0.0.1:8080 to internet)
└───────────────┬─────────────────┘
                │ Loopback HTTP
                ▼
┌─────────────────────────────────┐
│ Simple Relay (:8080)            │ ◄── Embedded Web UI (:8080/ui) & SQLite WAL
└───────────────┬─────────────────┘
                │ Asynchronous Dispatch & Retries
                ▼
┌─────────────────────────────────┐
│ Local Application Server (:3000)│ (Your local dev backend)
└─────────────────────────────────┘
```

In this workflow, Cloudflare Tunnel forwards all incoming public traffic directly to Simple Relay. Simple Relay commits the raw request to local SQLite WAL storage, immediately returns HTTP `200 OK` to the tunnel, and asynchronously delivers the payload to your local development backend with automatic retries.

---

## 2. Step-by-Step Setup

### Step 1: Start Simple Relay

Start Simple Relay in development mode on port `8080`:

```bash
# Using the standalone binary
relay

# Or via Docker
docker run -d --name relay -p 8080:8080 simple-relay:latest
```

The embedded inspection dashboard is immediately available at `http://localhost:8080/ui`.

### Step 2: Configure a Local Inbound Route

Open the dashboard at `http://localhost:8080/ui`, navigate to the **Routes** tab, click **New Route**, and configure an asynchronous inbound route:

| Setting | Value | Description |
| :--- | :--- | :--- |
| **Route Name** | `local-dev` | Route identifier used in URL paths. |
| **Type** | `inbound` | Ingests incoming webhooks and persists before delivery. |
| **Delivery Mode** | `async` | Acknowledges receipt instantly, delivers in background. |
| **Inbound Path** | `/api/v1/webhook/local-dev` | The endpoint Cloudflare Tunnel will route to. |
| **Destination URL** | `http://127.0.0.1:3000/api/webhooks` | Your local application backend port and path. |
| **Max Attempts** | `5` | Automatically retry deliveries if your local server is restarting. |

Alternatively, create the route via REST API:

```bash
curl -X POST http://localhost:8080/api/v1/routes \
  -H "Content-Type: application/json" \
  -d '{
    "name": "local-dev",
    "type": "inbound",
    "mode": "async",
    "path": "/api/v1/webhook/local-dev",
    "destination_url": "http://127.0.0.1:3000/api/webhooks",
    "max_attempts": 5,
    "timeout_ms": 5000
  }'
```

### Step 3: Launch Cloudflare Tunnel

Point `cloudflared` to Simple Relay on port `8080`:

```bash
cloudflared tunnel --url http://127.0.0.1:8080
```

`cloudflared` will generate a public tunnel URL (e.g., `https://random-words.trycloudflare.com`).

### Step 4: Configure Webhook Provider

In your provider's developer dashboard (e.g., Stripe Developer Dashboard or GitHub Webhook settings), configure the webhook destination URL using the tunnel domain and your Simple Relay route path:

```
https://random-words.trycloudflare.com/api/v1/webhook/local-dev
```

---

## 3. Inspecting Local Webhooks

Whenever an event triggers, open `http://localhost:8080/ui` and navigate to the **Requests** tab:

1. **Live Requests Explorer:** See all incoming webhook deliveries in chronological order, with HTTP methods, route names, delivery status (`success`, `failed`, `dlq`), and attempt counts.
2. **Detailed Inspection Drawer:** Click any request row or the **Inspect →** button to open the slide-over drawer:
   - **Headers Audit:** Inspect cryptographic headers (`Stripe-Signature`, `X-Hub-Signature-256`, `User-Agent`).
   - **Raw Payload Body:** View unmodified JSON/text payloads formatted with syntax highlighting.
   - **Copy as cURL:** Export the exact incoming webhook as an executable `curl` command with all original headers and raw body bytes preserved.
   - **Attempt History:** Review execution duration in milliseconds, HTTP response status codes returned by your local backend, and error responses.

---

## 4. Replaying Requests for Local Debugging

A major limitation of local webhook testing is having to manually re-trigger events from external dashboards whenever you modify your local handler code.

Simple Relay allows you to replay **any historical request**—including requests that previously finished with `success` (`200 OK`)—resetting the request back to `pending` and immediately re-dispatching it to your local backend.

### One-Click Replay via Web UI

- **From the Requests Table:** Click the **Replay** button on any table row in the Requests Explorer.
- **From the Inspection Drawer:** Click the **Replay** button in the top header of the drawer while inspecting request payload details.

A success notification will confirm the replay, and the request will immediately redeliver to your local app.

### Replay via Command Line (CLI)

You can also trigger replays directly from your terminal using the embedded CLI:

```bash
# Replay any request (success, failed, or dlq)
relay request replay 01j9y9success001

# Replay using JSON output for scripting
relay request replay 01j9y9success001 -o json
```

---

## 5. Crash-Resilient Server Restarts

When developing locally with hot-reloading tools, your local application process regularly stops and restarts. If a webhook arrives during those seconds, standard proxy tools return a connection refused error, and the webhook provider flags the endpoint as unhealthy.

With Simple Relay in front:
1. `cloudflared` delivers the webhook to Simple Relay.
2. Simple Relay durably commits the raw byte payload to SQLite WAL storage and returns `200 OK` to Cloudflare Tunnel.
3. If your local app is currently down or rebooting, the dispatcher attempt fails, records the failure audit, and schedules an automatic exponential backoff retry.
4. As soon as your local app finishes restarting, Simple Relay delivers the queued webhook without any lost data.
