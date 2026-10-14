---
title: "Recipe: Handling 429 & Rate Limits"
description: "Gracefully survive upstream API rate limits with automatic Retry-After parsing and exponential jitter."
section: "Recipes & Guides"
order: 7
---

Modern SaaS APIs (e.g., OpenAI, Anthropic, Shopify, Twitter, Slack) strictly enforce rate limits. When your application exceeds quota, upstream returns:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 12
```

Failing to respect rate limits or immediately retrying in a tight loop can lead to temporary account suspension or permanent IP throttling.

---

## How Simple Relay Handles 429 Responses

When routing outbound traffic through `/api/v1/proxy/sync` or `/api/v1/proxy/async`:

1. **Automatic Status Detection:** Responses with status code `429` are flagged as transient retryable conditions.
2. **`Retry-After` Header Parsing:**
   - **Integer Seconds:** e.g., `Retry-After: 30` -> Simple Relay sleeps for 30 seconds (plus subtle uniform random jitter to avoid synchronicity).
   - **HTTP-Date:** e.g., `Retry-After: Wed, 21 Oct 2026 07:28:00 GMT` -> Simple Relay calculates the exact duration delta between current clock time and target timestamp.
3. **Fallback to Exponential Backoff:** If the upstream server returns `429` without a `Retry-After` header, Simple Relay automatically falls back to standard exponential backoff with full jitter.
4. **Context Respect:** If the synchronous caller terminates connection or hits its total timeout, Simple Relay cancels the upstream wait immediately.

---

## Configuration Example

```yaml
proxy:
  retry_on_status: [429, 500, 502, 503, 504]
  max_attempts: 8
  initial_interval_ms: 1000
  max_interval_ms: 60000
  respect_retry_after: true
```

---

## Practical Example: Calling OpenAI Through Simple Relay

```bash
curl -X POST "http://localhost:8080/api/v1/proxy/sync?url=https://api.openai.com/v1/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Explain HTTP idempotency in one sentence."}]
  }'
```

If OpenAI returns a `429` due to organization TPM/RPM limits, Simple Relay holds the connection, sleeps for the duration requested by OpenAI's `Retry-After` header, and retries seamlessly. Your backend application receives a clean `200 OK` as if the rate limit never happened.
