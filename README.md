# quick-ding

Docker infrastructure for the cosmoherraje project (salamanca-proxy + InvoiceNinja).

Both modes share the same containers and volumes, so they can coexist on the same machine without conflicts.

## Local development

Starts MariaDB, MinIO, and InvoiceNinja. Salamanca runs on the host via `bun --hot`.

```bash
# 1. Start infra
cd quick-ding
cp .env.example .env          # edit passwords if needed
docker compose up -d           # mariadb + minio + invoiceninja

# 2. Push DB schema
cd ../salamanca-proxy
pnpm db:push

# 3. Run server with hot reload
pnpm dev:server                # http://localhost:3000
```

Services available:
- MariaDB: `localhost:3306` (databases: `salamanca`, `invoiceninja`)
- MinIO API: `localhost:9000`, Console: `localhost:9001`
- InvoiceNinja: `localhost:8080`
- Salamanca: `localhost:3000` (running on host)

Test the webhook locally:
```bash
# should return 401
curl -s -X POST "http://localhost:3000/api/webhooks/invoiceninja?secret=wrong" \
  -H "Content-Type: application/json" -d '{}'

# should return ignored (secret matches .env WEBHOOK_SECRET)
curl -s -X POST "http://localhost:3000/api/webhooks/invoiceninja?secret=local-dev-secret-1234" \
  -H "Content-Type: application/json" \
  -d '{"event_type":"invoice.updated","data":{"id":"test"}}'
```

Configure InvoiceNinja webhook (via admin UI at `:8080`):
```
http://host.docker.internal:3000/api/webhooks/invoiceninja?secret=local-dev-secret-1234
```

## Full stack (production-like)

Adds containerized Salamanca on host port **3002** (configurable via `SALAMANCA_HOST_PORT`).

```bash
cd quick-ding
cp .env.example .env           # fill in all values
docker compose --profile full up -d
```

Services available:
- MariaDB: `localhost:3306`
- MinIO API: `localhost:9000`, Console: `localhost:9001`
- InvoiceNinja: `localhost:8080`
- Salamanca: `localhost:3002` (or `$SALAMANCA_HOST_PORT`)

InvoiceNinja webhook URL (internal Docker network):
```
http://salamanca:3000/api/webhooks/invoiceninja?secret=${WEBHOOK_SECRET}
```

## Running both at once

| Service | Dev (host) | Full (container) |
|---------|-----------|------------------|
| Salamanca | :3000 | :3002 |
| MariaDB | :3306 (shared) | :3306 (shared) |
| MinIO | :9000 (shared) | :9000 (shared) |
| InvoiceNinja | :8080 (shared) | :8080 (shared) |

Both modes read/write the same database and MinIO buckets.

## Teardown

```bash
docker compose --profile full down        # stop all
docker compose --profile full down -v     # stop + delete volumes
```
