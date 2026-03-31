---
name: ares
description: Use when working with the Ares web scraper — an LLM-powered Rust tool that extracts structured data from websites using JSON Schemas. Covers library usage, CLI commands, REST API, schema creation, adding custom fetchers/cleaners/extractors, deployment, and contributing to the Ares codebase.
---

# Ares — LLM-Powered Web Scraper

Ares is a Rust library, CLI, and HTTP server that extracts structured data from websites using LLMs and JSON Schemas.

**Repository:** https://github.com/AndreaBozzo/Ares
**License:** Apache-2.0 | **Rust edition:** 2024 | **MSRV:** 1.88+

## Pipeline

```
URL → [ContentCache?] → Fetcher (HTML) → Cleaner (Markdown) → [ExtractionCache?] → Extractor (LLM + JSON Schema) → Hash → Compare → Store
```

Each stage is a trait, so every component can be swapped or mocked independently. Optional in-memory caches (moka) skip fetch/extraction when content or results are already cached.

## Crate Map

| Crate | Purpose | Key Exports |
|---|---|---|
| `ares-core` | Business logic, traits, pipeline | `ScrapeService`, `WorkerService`, `CircuitBreaker`, `ThrottledFetcher`, `CrawlConfig`, `ContentCache`, `ExtractionCache`, `CacheConfig`, `validate_schema`, traits |
| `ares-client` | HTTP/browser fetchers, cleaner, LLM client | `ReqwestFetcher`, `BrowserFetcher`, `HtmdCleaner`, `OpenAiExtractor`, `HtmlLinkDiscoverer`, `CachedRobotsChecker` |
| `ares-db` | PostgreSQL persistence | `Database`, `ExtractionRepository`, `ScrapeJobRepository` |
| `ares-api` | Axum REST API | Routes, DTOs, bearer auth, OpenAPI/Swagger, crawl endpoints |
| `ares-cli` | Command-line interface | `scrape`, `history`, `job`, `worker`, `crawl`, `schema` subcommands, output formats |

## Core Traits (`ares-core::traits`)

```rust
pub trait Fetcher: Send + Sync + Clone {
    fn fetch(&self, url: &str) -> impl Future<Output = Result<String, AppError>> + Send;
}

pub trait Cleaner: Send + Sync + Clone {
    fn clean(&self, html: &str) -> Result<String, AppError>;
}

pub trait Extractor: Send + Sync + Clone {
    fn extract(&self, content: &str, schema: &serde_json::Value)
        -> impl Future<Output = Result<serde_json::Value, AppError>> + Send;
}

pub trait ExtractorFactory: Send + Sync + Clone {
    type Extractor: Extractor;
    fn create(&self, model: &str, base_url: &str) -> Result<Self::Extractor, AppError>;
}

pub trait ExtractionStore: Send + Sync + Clone {
    fn save(&self, extraction: &NewExtraction) -> impl Future<Output = Result<Uuid, AppError>> + Send;
    fn get_latest(&self, url: &str, schema_name: &str) -> impl Future<Output = Result<Option<Extraction>, AppError>> + Send;
    fn get_history(&self, url: &str, schema_name: &str, limit: usize, offset: usize) -> impl Future<Output = Result<Vec<Extraction>, AppError>> + Send;
}
```

`JobQueue` trait: see `ares-core::job_queue` — persistent queue with atomic claiming (`SELECT FOR UPDATE SKIP LOCKED`).

```rust
pub trait LinkDiscoverer: Send + Sync + Clone {
    fn discover_links(&self, html: &str, base_url: &str) -> Result<Vec<String>, AppError>;
}

pub trait RobotsChecker: Send + Sync + Clone {
    fn is_allowed(&self, url: &str) -> impl Future<Output = Result<bool, AppError>> + Send;
}
```

## Key Types

| Type | Module | Purpose |
|---|---|---|
| `Extraction` | `ares_core::models` | Completed extraction (id, url, schema_name, extracted_data, hashes, model, created_at) |
| `NewExtraction` | `ares_core::models` | Insert DTO (no id/timestamps) |
| `ScrapeResult` | `ares_core::models` | Pipeline output (extracted_data, hashes, changed flag, extraction_id) |
| `ScrapeJob` | `ares_core::job` | Queued job with status, retry info, LLM config |
| `JobStatus` | `ares_core::job` | Enum: Pending, Running, Completed, Failed, Cancelled |
| `RetryConfig` | `ares_core::job` | Exponential backoff: 1min → 5min → 30min → 60min (capped) |
| `WorkerConfig` | `ares_core::job` | Worker settings: poll_interval, retry_config, skip_unchanged |
| `AppError` | `ares_core::error` | Error enum with `is_retryable()` and `should_trip_circuit()` |
| `SchemaResolver` | `ares_core::schema` | CRUD for schemas: resolve, create, update, delete + registry management |
| `CircuitBreaker` | `ares_core::circuit_breaker` | Closed → Open → HalfOpen state machine |
| `ThrottledFetcher<F>` | `ares_core::throttle` | Per-domain delay with jitter |
| `CrawlConfig` | `ares_core::crawl` | Crawl settings: max_depth, max_pages, allowed_domains, respect_robots_txt |
| `CacheConfig` | `ares_core::cache` | Cache TTL and capacity limits |
| `ContentCache` | `ares_core::cache` | URL-keyed in-memory HTML cache (moka) |
| `ExtractionCache` | `ares_core::cache` | Content+schema+model-keyed extraction result cache (moka) |
| `OutputFormat` | `ares_cli::output` | Enum: Json, Jsonl, Csv, Table, Jq |

## Quick Start (Library Usage)

```rust
use ares_client::{ReqwestFetcher, HtmdCleaner, OpenAiExtractor};
use ares_core::{ScrapeService, NullStore};

let fetcher = ReqwestFetcher::new()?;
let cleaner = HtmdCleaner::new();
let extractor = OpenAiExtractor::with_base_url(&api_key, "gpt-4o-mini", "https://api.openai.com/v1")?;

let service = ScrapeService::<_, _, _, NullStore>::new(fetcher, cleaner, extractor, "gpt-4o-mini".into());

let schema = serde_json::json!({
    "type": "object",
    "properties": {
        "title": {"type": "string"},
        "author": {"type": "string"}
    },
    "required": ["title", "author"]
});

let result = service.scrape("https://example.com/blog", &schema, "blog").await?;
println!("{}", serde_json::to_string_pretty(&result.extracted_data)?);
```

With persistence, use `ScrapeService::with_store(fetcher, cleaner, extractor, store, model)`.

## Reference Guides

| Topic | File | When to Read |
|---|---|---|
| Architecture deep-dive | `references/architecture.md` | Understanding pipeline internals, crate dependencies, resilience patterns |
| JSON Schema system | `references/schemas.md` | Creating/managing schemas, registry, versioning |
| Extending Ares | `references/extending.md` | Implementing custom Fetcher/Cleaner/Extractor/Store/JobQueue |
| CLI & REST API | `references/cli-and-server.md` | Running CLI commands, calling API endpoints, deploying |
| Contributing | `references/contributing.md` | Dev setup, testing, CI, code style |

## Version Notes

- **Current version:** 0.2.0
- Until crates.io release, use git dependency: `ares-core = { git = "https://github.com/AndreaBozzo/Ares" }`
- Works with any OpenAI-compatible API (OpenAI, Gemini, etc.)
- Browser support requires feature flag: `--features browser`
- **New in 0.2.0:** Web crawling, in-memory caching, output formats (json/jsonl/csv/table/jq), schema validation, 8 built-in schema templates
