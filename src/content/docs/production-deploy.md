---
title: "Recipe: Production Deployment"
description: "Deploy Simple Relay to production using Docker Compose, Systemd, and continuous streaming backups."
section: "Recipes & Guides"
order: 8
---

Because Simple Relay is packaged as a single self-contained binary, production deployment is straightforward and requires no complex orchestrators.

---

## 1. Docker Compose Setup

A standard production `docker-compose.yml` file:

```yaml
version: "3.8"

services:
  relay:
    image: ghcr.io/simple-relay/simple-relay:latest
    container_name: simple-relay
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - RELAY_LISTEN_ADDR=:8080
      - RELAY_DB_PATH=/data/relay.db
      - RELAY_WORKERS=8
      - RELAY_LOG_LEVEL=info
    volumes:
      - ./data:/data
    healthcheck:
      # Use relay for zero-dependency health checks (no curl required)
      test: ["CMD", "/usr/local/bin/relay", "health"]
      interval: 10s
      timeout: 3s
      retries: 3
```

> [!TIP] Zero-Dependency Probes
> In minimal container environments where `curl` or `wget` are absent, use `relay health` or `relay queue` to perform health checks and queue depth monitoring. See the **[CLI Reference](/docs/cli-reference)** for Docker and Kubernetes probe configurations.

Start the service:

```bash
docker compose up -d
```

---

## 2. Linux Systemd Service

For bare-metal servers or cloud virtual machines (EC2, Hetzner, DigitalOcean):

1. Place the compiled binary in `/usr/local/bin/relay`.
2. Create dedicated data directory:
   ```bash
   sudo mkdir -p /var/lib/simple-relay
   sudo useradd -r -s /bin/false simplerelay
   sudo chown -R simplerelay:simplerelay /var/lib/simple-relay
   ```
3. Create systemd unit at `/etc/systemd/system/simple-relay.service`:

```ini
[Unit]
Description=Simple Relay - Reliable HTTP Delivery Layer
After=network.target

[Service]
Type=simple
User=simplerelay
Group=simplerelay
WorkingDirectory=/var/lib/simple-relay
ExecStart=/usr/local/bin/relay --listen :8080 --db-path /var/lib/simple-relay/relay.db
Restart=always
RestartSec=5s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now simple-relay
sudo systemctl status simple-relay
```

---

## 3. Continuous Streaming Backups with Litestream

For disaster recovery, pair Simple Relay with [Litestream](https://litestream.io/) to continuously stream point-in-time snapshots to AWS S3, Cloudflare R2, or any S3-compatible storage:

```yaml
# /etc/litestream.yml
dbs:
  - path: /var/lib/simple-relay/relay.db
    replicas:
      - url: s3://my-backup-bucket/simple-relay
        sync-interval: 1s
```

Run Litestream alongside Simple Relay:

```bash
litestream replicate -exec "/usr/local/bin/relay --listen :8080 --db-path /var/lib/simple-relay/relay.db"
```

If the host suffers catastrophic hardware failure, your entire database state can be restored instantly from S3 with zero data loss.
