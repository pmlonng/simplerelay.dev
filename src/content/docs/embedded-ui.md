---
title: "Embedded Web UI"
description: "Inspect requests, analyze delivery attempts, and replay dead letters directly from the built-in browser dashboard."
section: "Reference"
order: 11
---

Simple Relay packages an administrative Web UI directly inside the Go binary via `go:embed`. It requires zero external Node.js dependencies or separate frontend server processes.

Access the dashboard at:

```text
http://localhost:8080/ui/
```

---

## Key Features of the Dashboard

### 1. Real-time Delivery Log
Inspect all inbound webhooks and outbound proxy calls:
- Chronologically sorted by monotonic ULID timestamp.
- Filter by status (`QUEUED`, `DELIVERING`, `DELIVERED`, `FAILED`).
- Visual latency badges showing duration per delivery attempt.

### 2. Payload & Header Inspector
Clicking any request opens an inspection modal showing:
- Bit-for-bit raw incoming request headers and payload body.
- Forwarded destination URL.
- Upstream response headers and response body (truncated to safe size limits).

### 3. Attempt History & Error Traces
When upstream services fail, inspect:
- Number of attempts made.
- Exact backoff delay calculated with full jitter.
- Error messages (e.g. connection refused, TLS handshake timeout, 502 Bad Gateway).

### 4. One-Click Dead Letter Queue (DLQ) Replay
Any delivery in the `FAILED` state features an interactive **Replay** button. Click to requeue the message instantly without re-sending from the external webhook provider.
