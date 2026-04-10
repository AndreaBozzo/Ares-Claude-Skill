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

`ares-core` has zero external runtime dependencies beyond `serde`, `tokio`, `sha2`, `chrono`, `uuid`, `url`, `tracing`, and `thiserror`. All I/O is abstracted behind traits.

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
- `.with_caches(Some(content_cache), Some(extraction_cache))` — enable in-memory caching

Use `NullStore` (from `ares_core::traits`) as the type parameter when persistence is not needed.

## Caching

Two in-memory caches using the `moka` crate (async-aware, TTL-based eviction):

**ContentCache** — caches fetched HTML by URL hash (SHA-256):
- Default: 1,000 entries, 3600s TTL
- Skips `Fetcher::fetch()` on cache hit

**ExtractionCache** — caches LLM extraction results by composite key:
- Key: `SHA256(content_hash:schema_name:schema_hash:model)`
- Default: 10,000 entries, 3600s TTL
- Skips `Extractor::extract()` on cache hit (avoids redundant LLM API calls)

Both caches are optional — pass `None` to disable. CLI flags: `--no-cache`, `--cache-ttl <secs>`.

## Crawling

`CrawlConfig` (in `ares-core::crawl`) drives recursive web crawling:

```rust
CrawlConfig {
    max_depth: u32,           // default: 3
    max_pages: u32,           // default: 100
    allowed_domains: Vec<String>,
    respect_robots_txt: bool, // default: true
    url_pattern: Option<String>,
}
```

Crawl flow:
1. Seed URL creates a `ScrapeJob` with `crawl_session_id` and `depth: 0`
2. Worker processes the job via `ScrapeService`, gets `raw_html` back
3. `LinkDiscoverer::discover_links(html, base_url)` extracts URLs from HTML
4. `RobotsChecker::is_allowed(url)` filters by robots.txt rules (cached per-domain)
5. New child jobs are created with `depth + 1`, respecting `max_depth` and `max_pages`
6. Process repeats until limits are reached or no new URLs are found

Built-in implementations:
- `HtmlLinkDiscoverer` — extracts `<a href>` tags, resolves relative URLs, deduplicates
- `CachedRobotsChecker` — fetches and caches robots.txt per domain, graceful degradation on failures

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

## Proxy Rotation

`ProxyConfig` (in `ares-core::proxy`) manages a pool of proxy endpoints:

```rust
ProxyConfig::new(proxies, RotationStrategy::RoundRobin)
```

- **Round-robin**: cycles through proxies in order via `AtomicUsize`
- **Random**: picks a random proxy each time (uses centralized `ares_core::rand::random_index`)
- Thread-safe: concurrent callers get different proxies without locking
- `ProxyConfig::entries()` returns proxies in insertion order (for building parallel data structures)
- `ProxyEntry::authenticated_url()` embeds percent-encoded credentials into the URL

`ReqwestFetcher::with_proxies(config)` pre-builds one `reqwest::Client` per proxy in stable insertion order, so `clients[i]` always corresponds to `proxies[i]`. `next_index()` selects which client to use per request.

## TLS Fingerprint Diversity

`TlsBackend` (in `ares-core::proxy`):

- **Rustls** (default): pure-Rust TLS — consistent cross-platform fingerprint
- **Native**: platform TLS (OpenSSL / SChannel / SecureTransport)
- **Random**: alternates per client at construction time

`ReqwestFetcher::with_tls_backend(backend)` must be called **before** `with_proxies` so per-proxy clients use the same backend.

## User-Agent Rotation

`UserAgentPool` (in `ares-client::user_agent`):

- 20 curated browser UA strings (Chrome, Firefox, Safari, Edge across Windows/macOS/Linux/mobile)
- `pool.next()` returns a random UA per call
- `ReqwestFetcher::with_random_ua()` overrides the default UA header per request
- `BrowserFetcher` rotates UA per page when stealth is enabled

## Browser Stealth

`StealthConfig` (in `ares-core::stealth`) — all features opt-in (default: disabled):

| Technique | Field | What It Does |
|---|---|---|
| Hide webdriver | `hide_webdriver` | Patches `navigator.webdriver`, fakes `window.chrome`, WebGL, plugins, permissions |
| Rotate User-Agent | `rotate_user_agent` | Per-page UA from `UserAgentPool` |
| Randomize viewport | `randomize_viewport` | 10 common desktop resolutions |
| Spoof platform | `spoof_platform` | `navigator.platform` matching the UA OS |
| Spoof languages | `spoof_languages` | Realistic `navigator.languages` arrays |

`StealthConfig::full()` enables everything. `StealthConfig::disabled()` (the default) disables all.

Stealth injections are applied on a blank page (`about:blank`) **before** navigating to the target URL, so `AddScriptToEvaluateOnNewDocument` hooks fire before any site JavaScript.

## Randomness

`ares_core::rand::random_index(len)` — centralized lightweight RNG:

- xorshift64 seeded from `SystemTime` nanos
- Mixed with a global `AtomicU64` counter to avoid same-nanos collisions under concurrency
- Used by proxy rotation, stealth helpers, and User-Agent selection
- Not cryptographically secure — intended for load balancing / rotation only

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

Migration `003_crawl_support.sql`:
- Adds crawl columns to `scrape_jobs`: `crawl_session_id`, `parent_job_id`, `depth`, `max_depth`, `max_pages`, `allowed_domains`
- New table `crawl_visited_urls`: `session_id` + `url_hash` PK for URL deduplication
- Indexes: `idx_scrape_jobs_crawl_session`, `idx_scrape_jobs_parent`
