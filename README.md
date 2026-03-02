# quick-ding

Docker infrastructure for the cosmoherraje project (salamanca-proxy + InvoiceNinja).

Both modes share the same containers and volumes, so they can coexist on the same machine without conflicts.

## Architecture

InvoiceNinja runs as two containers following the [official Docker setup](https://github.com/invoiceninja/dockerfiles):

- **invoiceninja** — PHP-FPM application (port 9000, internal only)
- **in-nginx** — Nginx reverse proxy that serves static files and forwards PHP requests to the app via FastCGI

This is required because the InvoiceNinja image only runs PHP-FPM and does not include a web server.

## Initial setup

```bash
cd quick-ding
cp .env.example .env
```

### 1. Generate `INVOICENINJA_APP_KEY`

```bash
docker run --rm invoiceninja/invoiceninja:5 php artisan key:generate --show
```

Copy the full output (including `base64:`) into your `.env`:

```
INVOICENINJA_APP_KEY=base64:AbCdEf1234...
```

### 2. Generate `WEBHOOK_SECRET`

```bash
openssl rand -hex 16
```

Paste into `.env`. Must be at least 16 characters.

### 3. Generate `BETTER_AUTH_SECRET`

```bash
openssl rand -base64 32
```

Paste into `.env`. Must be at least 32 characters.

### 4. Start the stack

```bash
docker compose up -d
```

Wait for all containers to be healthy, then open `http://localhost:8080` and log in with the credentials from `.env` (`IN_USER_EMAIL` / `IN_PASSWORD`).

### 5. Generate `INVOICENINJA_API_TOKEN`

This token lets Salamanca call the InvoiceNinja API. Generate it from the UI:

1. Log in to InvoiceNinja at `http://localhost:8080`
2. Go to **Settings** → **Account Management** → **API Tokens**
3. Click **Add Token**, give it a name (e.g. `salamanca`)
4. Copy the token and paste it into `.env`:

```
INVOICENINJA_API_TOKEN=the_generated_token
```

### 6. Facturama credentials (optional for sandbox)

For sandbox testing, the default CFDI issuer values work out of the box with Facturama's test RFC. For production, replace `FACTURAMA_USER`, `FACTURAMA_PASSWORD`, and the `ISSUER_*` values with your real company data.

## Local development

Starts MariaDB, MinIO, InvoiceNinja (app + nginx). Salamanca runs on the host via `bun --hot`.

```bash
# 1. Start infra
docker compose up -d

# 2. Push DB schema
cd ../salamanca-proxy
pnpm db:push

# 3. Run server with hot reload
pnpm dev:server                # http://localhost:3000
```

Services available:
- MariaDB: `localhost:3306` (databases: `salamanca`, `invoiceninja`)
- MinIO API: `localhost:9000`, Console: `localhost:9001`
- InvoiceNinja: `localhost:8080` (nginx -> php-fpm)
- Salamanca: `localhost:3000` (running on host)

Test the webhook locally:
```bash
# should return 401 (bad secret)
curl -s -X POST "http://localhost:3000/api/webhooks/invoiceninja?secret=wrong&event=invoice.sent" \
  -H "Content-Type: application/json" -d '{}'

# should return 400 (missing event param)
curl -s -X POST "http://localhost:3000/api/webhooks/invoiceninja?secret=<WEBHOOK_SECRET>" \
  -H "Content-Type: application/json" -d '{"id":"test"}'

# should return ignored (valid but non-stampable event)
curl -s -X POST "http://localhost:3000/api/webhooks/invoiceninja?secret=<WEBHOOK_SECRET>&event=invoice.updated" \
  -H "Content-Type: application/json" \
  -d '{"id":"test","number":"INV-0001"}'

# should attempt stamp pipeline (invoice.sent is stampable)
curl -s -X POST "http://localhost:3000/api/webhooks/invoiceninja?secret=<WEBHOOK_SECRET>&event=invoice.sent" \
  -H "Content-Type: application/json" \
  -d '{"id":"test","number":"INV-0001"}'
```

Configure InvoiceNinja webhook via the API (recommended):
```bash
curl -X POST "http://localhost:8080/api/v1/webhooks" \
  -H "X-API-TOKEN: <INVOICENINJA_API_TOKEN>" \
  -H "X-Requested-With: XMLHttpRequest" \
  -H "Content-Type: application/json" \
  -d '{
    "target_url": "http://host.docker.internal:3000/api/webhooks/invoiceninja?secret=<WEBHOOK_SECRET>&event=invoice.sent",
    "event_id": 60,
    "rest_method": "post"
  }'
```

Or via the admin UI at `:8080`: **Settings** -> **Account Management** -> **API Webhooks** -> select "Invoice Sent" and set the target URL to:
```
http://host.docker.internal:3000/api/webhooks/invoiceninja?secret=<WEBHOOK_SECRET>&event=invoice.sent
```

## Full stack (production-like)

Adds containerized Salamanca on host port **3002** (configurable via `SALAMANCA_HOST_PORT`).

```bash
docker compose --profile full up -d
```

Services available:
- MariaDB: `localhost:3306`
- MinIO API: `localhost:9000`, Console: `localhost:9001`
- InvoiceNinja: `localhost:8080` (nginx -> php-fpm)
- Salamanca: `localhost:3002` (or `$SALAMANCA_HOST_PORT`)

InvoiceNinja webhook URL (internal Docker network):
```
http://salamanca:3000/api/webhooks/invoiceninja?secret=<WEBHOOK_SECRET>&event=invoice.sent
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
