# CLI & REST API

## CLI (`ares-cli`)

### `ares scrape` — One-shot extraction

```bash
cargo run --bin ares-cli -- scrape \
  --url https://example.com/blog \
  --schema schemas/blog/1.0.0.json \
  --model gpt-4o-mini \
  --api-key $ARES_API_KEY

# With name@version schema reference
cargo run --bin ares-cli -- scrape -u https://example.com -s blog@latest -m gpt-4o-mini

# With persistence
cargo run --bin ares-cli -- scrape -u https://example.com -s blog@latest --save

# With browser (for SPAs)
cargo run --bin ares-cli --features browser -- scrape -u https://spa.com -s blog@latest --browser

# With proxy rotation
cargo run --bin ares-cli -- scrape -u https://example.com -s blog@latest \
  --proxy http://proxy1:8080 --proxy-rotation round-robin

# With stealth mode (requires --browser)
cargo run --bin ares-cli --features browser -- scrape -u https://spa.com -s blog@latest \
  --browser --stealth --random-ua --tls-backend native

# With proxy rotation
cargo run --bin ares-cli -- scrape -u https://example.com -s blog@latest \
  --proxy http://proxy1:8080 --proxy-rotation round-robin

# With stealth mode (requires --browser)
cargo run --bin ares-cli --features browser -- scrape -u https://spa.com -s blog@latest \
  --browser --stealth --random-ua --tls-backend native

# Skip saving if data unchanged
cargo run --bin ares-cli -- scrape -u https://example.com -s blog@latest --save --skip-unchanged
```

Flags:
- `-u, --url` — Target URL (required)
- `-s, --schema` — Schema file path or `name@version` (required)
- `-m, --model` — LLM model name (default: env `ARES_MODEL` or `gpt-4o-mini`)
- `-b, --base-url` — OpenAI-compatible API base URL (default: env `ARES_BASE_URL`)
- `-a, --api-key` — LLM API key (default: env `ARES_API_KEY`)
- `--save` — Persist to database (requires `DATABASE_URL`)
- `--schema-name` — Override schema name for storage
- `--browser` — Use headless browser (requires `--features browser`)
- `--fetch-timeout` — HTTP fetch timeout in seconds
- `--llm-timeout` — LLM API timeout in seconds
- `--system-prompt` — Custom system prompt for the LLM
- `--skip-unchanged` — Skip saving if data hash matches previous
- `--throttle` — Per-domain throttle delay in milliseconds (0 = disabled)
- `--no-cache` — Disable in-memory caching (content + extraction)
- `--cache-ttl` — Cache TTL in seconds (default: 3600, env: `ARES_CACHE_TTL`)
- `--proxy` — Proxy URL: http, https, or socks5 (env: `ARES_PROXY`)
- `--proxy-file` — Path to file with one proxy URL per line (env: `ARES_PROXY_FILE`)
- `--proxy-rotation` — Rotation strategy: `round-robin` (default) or `random`
- `--random-ua` — Rotate User-Agent with realistic browser strings
- `--stealth` — Browser stealth mode (requires `--browser`): hides webdriver, randomises viewport, spoofs navigator
- `--tls-backend` — TLS backend: `rustls` (default), `native`, `random` (env: `ARES_TLS_BACKEND`)
- `--format` — Output format: `json` (default), `jsonl`, `csv`, `table`, `jq`

### `ares history` — View extraction history

```bash
cargo run --bin ares-cli -- history -u https://example.com -s blog --limit 10
```

Flags:
- `-u, --url` — Filter by URL (required)
- `-s, --schema-name` — Filter by schema name (required)
- `--limit` — Max results (default: 10)
- `--format` — Output format: `json` (default), `jsonl`, `csv`, `table`, `jq`

### `ares job` — Manage persistent jobs

```bash
# Create a job
cargo run --bin ares-cli -- job create \
  -u https://example.com \
  -s blog@latest \
  -m gpt-4o-mini \
  -b https://api.openai.com/v1

# List jobs
cargo run --bin ares-cli -- job list --status pending --limit 20

# Show job details
cargo run --bin ares-cli -- job show <JOB_ID>

# Cancel a job
cargo run --bin ares-cli -- job cancel <JOB_ID>
```

### `ares worker` — Background job processor

```bash
cargo run --bin ares-cli -- worker \
  --poll-interval 5 \
  --api-key $ARES_API_KEY \
  --skip-unchanged

# With browser support
cargo run --bin ares-cli --features browser -- worker --browser
```

Flags:
- `--poll-interval` — Seconds between job queue polls (default: 5)
- `--api-key` — LLM API key (default: env `ARES_API_KEY`)
- `--browser` — Use headless browser for fetching
- `--skip-unchanged` — Skip saving if data unchanged
- `--throttle` — Per-domain throttle delay in milliseconds (0 = disabled)
- `--no-cache` — Disable in-memory caching
- `--cache-ttl` — Cache TTL in seconds (default: 3600, env: `ARES_CACHE_TTL`)
- `--proxy` — Proxy URL: http, https, or socks5 (env: `ARES_PROXY`)
- `--proxy-file` — Path to file with one proxy URL per line (env: `ARES_PROXY_FILE`)
- `--proxy-rotation` — Rotation strategy: `round-robin` (default) or `random`
- `--random-ua` — Rotate User-Agent with realistic browser strings
- `--stealth` — Browser stealth mode (requires `--browser`)
- `--tls-backend` — TLS backend: `rustls` (default), `native`, `random` (env: `ARES_TLS_BACKEND`)

### `ares crawl` — Recursive web crawling

```bash
# Start a crawl session
cargo run --bin ares-cli -- crawl start \
  -u https://example.com \
  -s blog@latest \
  --max-depth 2 \
  --max-pages 20 \
  --allowed-domains example.com,blog.example.com

# Check crawl progress
cargo run --bin ares-cli -- crawl status <SESSION_ID>

# Get extracted results from all crawled pages
cargo run --bin ares-cli -- crawl results <SESSION_ID>
```

Flags (start subcommand):
- `-u, --url` — Seed URL to start crawling from (required)
- `-s, --schema` — Schema path or `name@version` (required)
- `-d, --max-depth` — Maximum crawl depth (default: 1)
- `-m, --model` — LLM model (default: env `ARES_MODEL`)
- `-b, --base-url` — API base URL (default: env `ARES_BASE_URL`)
- `--max-pages` — Maximum total pages to crawl (default: 100)
- `--allowed-domains` — Comma-separated domain allowlist (defaults to seed URL domain)
- `--schema-name` — Override schema name

Note: Crawl creates jobs in the queue. A worker (`ares worker`) must be running to process them.

### `ares schema` — Schema management

```bash
# Validate a JSON Schema file
cargo run --bin ares-cli -- schema validate schemas/blog/1.0.0.json
```

## REST API (`ares-api`)

Server runs on `ARES_SERVER_PORT` (default: 3000). OpenAPI docs at `/swagger-ui`.

### Authentication

All `/v1/*` endpoints require bearer token: `Authorization: Bearer $ARES_ADMIN_TOKEN`

### Endpoints

#### `POST /v1/scrape` — One-shot scrape

```json
// Request
{
  "url": "https://example.com",
  "schema": {"type": "object", "properties": {...}},
  "schema_name": "blog",
  "model": "gpt-4o-mini",        // optional (falls back to ARES_MODEL)
  "base_url": "https://api.openai.com/v1",  // optional (falls back to ARES_BASE_URL)
  "save": true                    // optional (default: true)
}

// Response 200
{
  "extracted_data": {...},
  "content_hash": "abc123...",
  "data_hash": "def456...",
  "changed": true,
  "extraction_id": "uuid"
}
```

#### `POST /v1/jobs` — Create job (202 Accepted)

```json
// Request
{
  "url": "https://example.com",
  "schema_name": "blog",
  "schema": {...},
  "model": "gpt-4o-mini",
  "base_url": "https://api.openai.com/v1",
  "max_retries": 3               // optional
}

// Response 202
{ "job_id": "uuid", "status": "pending" }
```

#### `GET /v1/jobs?status=pending&limit=20&offset=0` — List jobs

```json
// Response 200
{
  "jobs": [...],
  "total": 150,       // total matching count in DB
  "limit": 20,
  "offset": 0
}
```

#### `GET /v1/jobs/{id}` — Get job details
#### `DELETE /v1/jobs/{id}` — Cancel job (204 / 404 / 409)

#### `POST /v1/jobs/{id}/retry` — Retry failed/cancelled job

Resets a `failed` or `cancelled` job back to `pending`. Returns 409 if the job is not in a retryable state, 404 if not found.

```json
// Response 200 — returns the updated job
{ "id": "uuid", "status": "pending", "retry_count": 0, ... }
```

#### `GET /v1/extractions?url=...&schema_name=...&limit=10&offset=0` — Query history

```json
// Response 200
{
  "extractions": [...],
  "total": 42,
  "limit": 10,
  "offset": 0
}
```

#### `GET /v1/schemas` — List all schemas
#### `GET /v1/schemas/{name}/{version}` — Get schema
#### `POST /v1/schemas` — Create schema (201)

```json
// Request
{ "name": "product", "version": "1.0.0", "schema": {...} }

// Response 201
{ "name": "product", "version": "1.0.0" }
```

#### `PUT /v1/schemas/{name}/{version}` — Update schema (200 / 404)

```json
// Request
{ "schema": {"type": "object", "properties": {...}} }

// Response 200
{ "name": "product", "version": "1.0.0", "schema": {...} }
```

#### `DELETE /v1/schemas/{name}/{version}` — Delete schema (204 / 404)

Deletes a specific schema version. If the deleted version was the latest, the registry is updated to point to the next most recent version. If it was the only version, the entry is removed from the registry.

#### `POST /v1/crawl` — Start crawl session (202 Accepted)

```json
// Request
{
  "url": "https://example.com",
  "schema_name": "blog",
  "schema": {...},
  "model": "gpt-4o-mini",
  "base_url": "https://api.openai.com/v1",
  "max_depth": 2,
  "max_pages": 50,
  "allowed_domains": ["example.com"]
}

// Response 202
{
  "session_id": "uuid",
  "seed_job_id": "uuid"
}
```

#### `GET /v1/crawl/{id}` — Crawl status

```json
// Response 200
{
  "session_id": "uuid",
  "total_jobs": 15,
  "pending": 3,
  "running": 1,
  "completed": 10,
  "failed": 1,
  "cancelled": 0,
  "progress": 73.3
}
```

#### `GET /v1/crawl/{id}/results` — Crawl results

Returns extracted data from all completed jobs in the crawl session.

#### `GET /health` — Health check (public, no auth)

```json
{ "status": "healthy", "database": "ok" }
```

## Environment Variables

```bash
# LLM Configuration
ARES_API_KEY=your-api-key          # Required for scrape/worker
ARES_MODEL=gpt-4o-mini             # Default model
ARES_BASE_URL=https://api.openai.com/v1  # OpenAI-compatible endpoint

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/ares

# Server
ARES_SERVER_PORT=3000
ARES_ADMIN_TOKEN=your-secret-token  # Bearer auth for /v1/* endpoints
ARES_SCHEMAS_DIR=schemas            # Schema files directory
ARES_CORS_ORIGIN=*                  # CORS allowed origins

# Rate Limiting
ARES_RATE_LIMIT_BURST=30
ARES_RATE_LIMIT_RPS=1
ARES_BODY_SIZE_LIMIT=2097152        # 2MB

# Caching
ARES_CACHE_TTL=3600                  # In-memory cache TTL in seconds

# Proxy & Anti-Bot Evasion
ARES_PROXY=http://proxy:8080         # Single proxy URL
ARES_PROXY_FILE=/path/to/proxies.txt # Proxy list file (one URL per line)
ARES_PROXY_ROTATION=round-robin      # round-robin or random
ARES_RANDOM_UA=false                 # Rotate User-Agent headers
ARES_STEALTH=false                   # Browser stealth mode (server only)
ARES_TLS_BACKEND=rustls              # rustls, native, or random

# Browser (optional)
CHROME_BIN=/path/to/chromium
```

## Deployment

### Docker Compose (development)

```bash
# Start PostgreSQL + pgAdmin
docker compose up -d

# Run migrations
make migrate

# Start server
cargo run --bin ares-api
```

`compose.yml` provides:
- PostgreSQL 16 with persistent volume
- pgAdmin 4 for database management

### Docker (production)

```bash
# Build image (multi-stage: Rust builder → Debian slim)
make docker-build
# or
docker build -t ares-api:latest .

# Run
docker run -p 3000:3000 --env-file .env ares-api:latest
```

The Dockerfile includes Chromium for browser-based scraping and uses release build with LTO + symbol stripping.
