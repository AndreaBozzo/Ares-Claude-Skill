# Extending Ares

All external dependencies are behind traits in `ares-core`. To add a new implementation, implement the relevant trait and wire it into `ScrapeService` or `WorkerService`.

## Implementing a Custom Fetcher

```rust
use ares_core::error::AppError;
use ares_core::traits::Fetcher;

#[derive(Clone)]
pub struct MyFetcher { /* your fields */ }

impl Fetcher for MyFetcher {
    async fn fetch(&self, url: &str) -> Result<String, AppError> {
        // Return raw HTML as a String
        // Use AppError::HttpError for request failures
        // Use AppError::Timeout for timeouts
        todo!()
    }
}
```

Built-in implementations:
- `ReqwestFetcher` — standard HTTP client (`ares-client::fetcher`)
- `BrowserFetcher` — headless Chrome via chromiumoxide (`ares-client::browser_fetcher`, feature: `browser`)
- `ThrottledFetcher<F>` — wraps any Fetcher with per-domain delays (`ares-core::throttle`)

## Implementing a Custom Cleaner

```rust
use ares_core::error::AppError;
use ares_core::traits::Cleaner;

#[derive(Clone)]
pub struct MyCleaner;

impl Cleaner for MyCleaner {
    fn clean(&self, html: &str) -> Result<String, AppError> {
        // Convert HTML to clean text (Markdown or plain text)
        // The output is what the LLM receives as context
        // Use AppError::CleanerError for failures
        todo!()
    }
}
```

Built-in: `HtmdCleaner` — uses the `htmd` crate for HTML-to-Markdown conversion (`ares-client::cleaner`).

Note: `Cleaner::clean` is **synchronous** (not async), unlike `Fetcher` and `Extractor`.

## Implementing a Custom Extractor

```rust
use ares_core::error::AppError;
use ares_core::traits::Extractor;

#[derive(Clone)]
pub struct MyExtractor { /* api_key, model, etc. */ }

impl Extractor for MyExtractor {
    async fn extract(
        &self,
        content: &str,
        schema: &serde_json::Value,
    ) -> Result<serde_json::Value, AppError> {
        // Send content + schema to an LLM and return structured JSON
        // Use AppError::LlmError { message, status_code, retryable } for API failures
        todo!()
    }
}
```

Built-in: `OpenAiExtractor` — works with any OpenAI-compatible API (`ares-client::llm`).

## Implementing an ExtractorFactory

Required by `WorkerService` to create per-job extractors with different model/base_url:

```rust
use ares_core::error::AppError;
use ares_core::traits::{Extractor, ExtractorFactory};

#[derive(Clone)]
pub struct MyExtractorFactory { /* shared config like api_key */ }

impl ExtractorFactory for MyExtractorFactory {
    type Extractor = MyExtractor;

    fn create(&self, model: &str, base_url: &str) -> Result<Self::Extractor, AppError> {
        Ok(MyExtractor { /* ... */ })
    }
}
```

Built-in: `OpenAiExtractorFactory` (`ares-client::llm`).

## Implementing a Custom ExtractionStore

```rust
use ares_core::error::AppError;
use ares_core::models::{Extraction, NewExtraction};
use ares_core::traits::ExtractionStore;
use uuid::Uuid;

#[derive(Clone)]
pub struct MyStore { /* connection pool, etc. */ }

impl ExtractionStore for MyStore {
    async fn save(&self, extraction: &NewExtraction) -> Result<Uuid, AppError> {
        // Persist and return generated UUID
        todo!()
    }

    async fn get_latest(&self, url: &str, schema_name: &str) -> Result<Option<Extraction>, AppError> {
        // Return most recent extraction for URL + schema pair
        todo!()
    }

    async fn get_history(&self, url: &str, schema_name: &str, limit: usize, offset: usize) -> Result<Vec<Extraction>, AppError> {
        // Return extraction history, newest first, with offset-based pagination
        todo!()
    }
}
```

Built-in:
- `ExtractionRepository` — PostgreSQL via sqlx (`ares-db::repository`)
- `NullStore` — no-op, returns `Uuid::nil()` (`ares-core::traits`)

## Implementing a Custom JobQueue

```rust
use ares_core::error::AppError;
use ares_core::job::{CreateScrapeJobRequest, JobStatus, ScrapeJob};
use ares_core::job_queue::JobQueue;
use chrono::{DateTime, Utc};
use uuid::Uuid;

#[derive(Clone)]
pub struct MyJobQueue { /* ... */ }

impl JobQueue for MyJobQueue {
    async fn create_job(&self, request: CreateScrapeJobRequest) -> Result<ScrapeJob, AppError> { todo!() }
    async fn claim_job(&self, worker_id: &str) -> Result<Option<ScrapeJob>, AppError> { todo!() }
    async fn complete_job(&self, job_id: Uuid, extraction_id: Option<Uuid>) -> Result<(), AppError> { todo!() }
    async fn fail_job(&self, job_id: Uuid, error: &str, next_retry_at: Option<DateTime<Utc>>) -> Result<(), AppError> { todo!() }
    async fn cancel_job(&self, job_id: Uuid) -> Result<(), AppError> { todo!() }
    async fn get_job(&self, job_id: Uuid) -> Result<Option<ScrapeJob>, AppError> { todo!() }
    async fn list_jobs(&self, status: Option<JobStatus>, limit: usize, offset: usize) -> Result<Vec<ScrapeJob>, AppError> { todo!() }
    async fn retry_job(&self, job_id: Uuid) -> Result<Option<ScrapeJob>, AppError> { todo!() }
    async fn release_job(&self, job_id: Uuid) -> Result<(), AppError> { todo!() }
    async fn release_worker_jobs(&self, worker_id: &str) -> Result<u64, AppError> { todo!() }
    async fn count_by_status(&self, status: JobStatus) -> Result<i64, AppError> { todo!() }
}
```

Built-in: `ScrapeJobRepository` — PostgreSQL with `SELECT FOR UPDATE SKIP LOCKED` (`ares-db::job_repository`).

Important: `claim_job` must be atomic — use row-level locking or equivalent to prevent double-claiming.

## Wiring Custom Implementations

```rust
// One-shot scraping
let service = ScrapeService::new(my_fetcher, my_cleaner, my_extractor, "model".into());
let result = service.scrape(url, &schema, "name").await?;

// With persistence
let service = ScrapeService::with_store(my_fetcher, my_cleaner, my_extractor, my_store, "model".into());

// Background worker
let worker = WorkerService::new(
    my_queue,
    my_fetcher,
    my_cleaner,
    my_extractor_factory,
    my_store,
    CircuitBreaker::new("llm", CircuitBreakerConfig::default()),
    WorkerConfig::default(),
);
worker.run(cancel_token, &TracingWorkerReporter).await?;
```

## Feature Gates Pattern

Browser support is behind a Cargo feature flag. Follow this pattern for optional heavy dependencies:

```toml
# In Cargo.toml
[features]
browser = ["chromiumoxide", "futures"]

[dependencies]
chromiumoxide = { version = "0.8", optional = true, ... }
```

Then conditionally compile with `#[cfg(feature = "browser")]`.

## Testing with Mocks

`ares-core::testutil` provides mock implementations for all traits:

- `MockFetcher::new(html)` / `MockFetcher::with_error(err)`
- `MockCleaner::passthrough()` / `MockCleaner::with_error(err)`
- `MockExtractor::new(json)` / `MockExtractor::with_error(err)`
- `MockExtractorFactory::new(json)` / `MockExtractorFactory::with_create_error(err)`
- `MockStore::empty()` / `MockStore::with_latest(extraction)` / `MockStore::with_save_error(err)`
- `MockJobQueue::empty()` / `MockJobQueue::with_job(job)`
- `make_test_extraction(data_hash)`, `make_test_job()`
