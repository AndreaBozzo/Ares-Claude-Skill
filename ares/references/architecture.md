# Architecture

## Pipeline Flow

```
ScrapeService::scrape(url, schema, schema_name)
  │
  ├─ 1. Fetcher::fetch(url)              → HTML string
  ├─ 2. Cleaner::clean(html)             → Markdown string
  ├─ 3. Extractor::extract(markdown, schema) → JSON Value
  ├─ 4. compute_hash(markdown)           → content_hash (SHA-256)
  ├─ 5. compute_hash(json.to_string())   → data_hash (SHA-256)
  ├─ 6. ExtractionStore::get_latest()    → compare data_hash for change detection
  └─ 7. ExtractionStore::save()          → persist (skipped if skip_unchanged && !changed)
```

## Crate Dependency Graph

```
ares-cli ──┬── ares-core
           ├── ares-client
           └── ares-db

ares-api ─┬── ares-core
             ├── ares-client
             └── ares-db

ares-client ──── ares-core
ares-db ──────── ares-core
```

`ares-core` has zero external runtime dependencies beyond `serde`, `tokio`, `sha2`, `chrono`, `uuid`, `tracing`, and `thiserror`. All I/O is abstracted behind traits.

## ScrapeService

```rust
// crates/ares-core/src/scrape.rs
pub struct ScrapeService<F, C, E, S>
where F: Fetcher, C: Cleaner, E: Extractor, S: ExtractionStore
{
    fetcher: F,
    cleaner: C,
    extractor: E,
    store: Option<S>,      // None = no persistence
    model_name: String,
    skip_unchanged: bool,  // skip save when data_hash matches previous
}
```

Constructors:
- `ScrapeService::new(fetcher, cleaner, extractor, model_name)` — no persistence
- `ScrapeService::with_store(fetcher, cleaner, extractor, store, model_name)` — with DB
- `.with_skip_unchanged(true)` — builder method to enable change-detection skipping

Use `NullStore` (from `ares_core::traits`) as the type parameter when persistence is not needed.

## WorkerService

```rust
// crates/ares-core/src/worker.rs
pub struct WorkerService<Q, F, C, EF, S>
where Q: JobQueue, F: Fetcher, C: Cleaner, EF: ExtractorFactory, S: ExtractionStore
```

The worker:
1. Polls `JobQueue::claim_job()` with its `worker_id`
2. Creates an `Extractor` via `ExtractorFactory::create(model, base_url)` per job
3. Builds a `ScrapeService` and runs the pipeline through the `CircuitBreaker`
4. On success: `JobQueue::complete_job(job_id, extraction_id)`
5. On retryable failure: `JobQueue::fail_job(job_id, error, Some(next_retry_at))`
6. On permanent failure: `JobQueue::fail_job(job_id, error, None)`
7. On shutdown (CancellationToken): `JobQueue::release_worker_jobs(worker_id)`

Events are decoupled via `WorkerReporter` trait. Default implementation: `TracingWorkerReporter`.

## CircuitBreaker

```rust
// crates/ares-core/src/circuit_breaker.rs
CircuitBreaker::new(name, CircuitBreakerConfig { ... })
```

States: `Closed` (healthy) → `Open` (rejecting) → `HalfOpen` (probing)

Config defaults:
- `failure_threshold`: 5
- `success_threshold`: 2
- `recovery_timeout`: 30s
- `rate_limit_backoff_multiplier`: 2.0 (doubles recovery timeout on 429)
- `max_recovery_timeout`: 300s

Key methods:
- `cb.call(|| async { ... })` — wraps an operation
- `cb.state()` — current state
- `cb.stats()` — monitoring snapshot
- `cb.reset()` — manual reset

## ThrottledFetcher

```rust
// crates/ares-core/src/throttle.rs
let config = ThrottleConfig::new(Duration::from_secs(1)).with_jitter(Duration::from_millis(500));
let fetcher = ThrottledFetcher::new(inner_fetcher, config);
```

Per-domain delay tracking. Thread-safe (Arc<Mutex<HashMap>>). Drops lock during sleep so different domains aren't blocked.

Default: 1s delay, 500ms jitter.

## Error Handling

`AppError` variants (in `ares-core::error`):

| Variant | Retryable | Trips Circuit |
|---|---|---|
| `HttpError(String)` | Only if contains "timeout"/"connect"/"reset" | Only if contains "timeout"/"connect"/"connection" |
| `LlmError { message, status_code, retryable }` | If `retryable` flag | If 429, 5xx, or `retryable` |
| `NetworkError(String)` | Yes | Yes |
| `Timeout(u64)` | Yes | Yes |
| `RateLimitExceeded` | Yes | Yes |
| `CleanerError(String)` | No | No |
| `SchemaValidationError(String)` | No | No |
| `SchemaError(String)` | No | No |
| `SchemaNotFound { name, version }` | No | No |
| `SerializationError(serde_json::Error)` | No | No |
| `ConfigError(String)` | No | No |
| `DatabaseError(String)` | No | No |
| `Generic(String)` | No | No |

## Database Schema

Migration `001_init.sql`:
- `extractions` table: `id UUID PK`, `url`, `schema_name`, `extracted_data JSONB`, `content_hash`, `data_hash`, `model`, `created_at`
- Indexes on `(url)` and `(url, schema_name)`

Migration `002_scrape_jobs.sql`:
- `scrape_jobs` table: full job lifecycle tracking with `status`, `retry_count`, `worker_id`, `next_retry_at`
- 5 indexes: pending jobs, retry scheduling, worker lookup, status filter, URL lookup
- Job claiming uses `SELECT FOR UPDATE SKIP LOCKED` for atomic concurrent access
