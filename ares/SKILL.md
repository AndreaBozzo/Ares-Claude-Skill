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
URL → [ContentCache?] → Fetcher (HTML) → Cleaner (Markdown) → [ExtractionCache?] → Extractor (LLM + JSON Schema) → Validate → Hash → Compare → Store
```

Each stage is a trait, so every component can be swapped or mocked independently. Optional in-memory caches (moka) skip fetch/extraction when content or results are already cached.

After extraction the result is validated against the JSON Schema (`validate_extracted_output`); on mismatch the pipeline returns `AppError::ExtractionValidationError` and nothing is persisted (toggle with `.with_validation(false)`). A heuristic groundedness check (`ungrounded_fields`) then warns — without failing — when short atomic values look absent from the source (a hallucination signal schema validation can't catch). **Valid JSON is not necessarily grounded truth.**

## Crate Map

| Crate | Purpose | Key Exports |
|---|---|---|
| `ares-core` | Business logic, traits, pipeline | `ScrapeService`, `WorkerService`, `CircuitBreaker`, `ThrottledFetcher`, `CrawlConfig`, `ContentCache`, `ExtractionCache`, `CacheConfig`, `ProxyConfig`, `StealthConfig`, `TlsBackend`, `validate_schema`, `validate_extracted_output`, `ungrounded_fields`, traits |
| `ares-client` | HTTP/browser fetchers, cleaner, LLM clients | `ReqwestFetcher`, `BrowserFetcher`, `HtmdCleaner`, `OpenAiExtractor` (+`Factory`), `AnthropicExtractor` (feature `anthropic`), `CandleExtractor` (feature `local-llm`), `Provider`, `ProviderExtractor` (+`Factory`), `HtmlLinkDiscoverer`, `CachedRobotsChecker`, `UserAgentPool` |
| `ares-db` | PostgreSQL persistence | `Database`, `ExtractionRepository`, `ScrapeJobRepository` |
| `ares-api` | Axum REST API | Routes, DTOs, bearer auth, OpenAPI/Swagger, crawl endpoints |
| `ares-cli` | Command-line interface | `scrape`, `history`, `job`, `worker`, `crawl`, `schema`, `model` subcommands, output formats |

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
    // Returns `true` if the URL may be fetched. On fetch/parse errors it
    // defaults to allowing (graceful degradation) — hence plain `bool`, not Result.
    fn is_allowed(&self, url: &str) -> impl Future<Output = bool> + Send;
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
| `ProxyConfig` | `ares_core::proxy` | Proxy pool with rotation (round-robin or random), thread-safe via AtomicUsize |
| `ProxyEntry` | `ares_core::proxy` | Single proxy endpoint (url + optional auth credentials, percent-encoded) |
| `RotationStrategy` | `ares_core::proxy` | Enum: RoundRobin, Random |
| `TlsBackend` | `ares_core::proxy` | Enum: Rustls (default), Native, Random — for TLS fingerprint diversity |
| `StealthConfig` | `ares_core::stealth` | Browser anti-fingerprinting config (all opt-in, default disabled) |
| `UserAgentPool` | `ares_client::user_agent` | 20 realistic browser UA strings, random selection per request |

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

## Providers & Backends

The `Extractor` trait is the seam for inference backends. Three ship in-tree, selected at runtime via `--provider` / `ARES_PROVIDER` (CLI), the `provider` field of `POST /v1/scrape` (API), or `Provider` + `ProviderExtractor`/`ProviderExtractorFactory` dispatch enums (library):

| Provider | Value | Backend | Build | Notes |
|---|---|---|---|---|
| OpenAI-compatible | `openai` (default) | `OpenAiExtractor` | default | OpenAI, Gemini compat endpoint, local OpenAI-style servers (llama.cpp/Ollama/LM Studio) via `--base-url` |
| Anthropic (Claude) | `anthropic` | `AnthropicExtractor` | `--features anthropic` | Native Messages API via forced tool use (not OpenAI-compatible) |
| Local (native) | `local` | `CandleExtractor` | `--features local-llm` | Native CPU inference through Candle; manage weights with `ares model pull/list/remove`. No API key, no per-token cost |

New backends implement `Extractor` + `ExtractorFactory`; nothing else in the pipeline changes.

## Reference Guides

| Topic | File | When to Read |
|---|---|---|
| Architecture deep-dive | `references/architecture.md` | Understanding pipeline internals, crate dependencies, resilience patterns |
| JSON Schema system | `references/schemas.md` | Creating/managing schemas, registry, versioning |
| Extending Ares | `references/extending.md` | Implementing custom Fetcher/Cleaner/Extractor/Store/JobQueue |
| CLI & REST API | `references/cli-and-server.md` | Running CLI commands, calling API endpoints, deploying |
| Contributing | `references/contributing.md` | Dev setup, testing, CI, code style |

## Version Notes

- **Current version:** 0.4.0
- Until crates.io release, use git dependency: `ares-core = { git = "https://github.com/AndreaBozzo/Ares" }`
- Works with any OpenAI-compatible API (OpenAI, Gemini, local servers) out of the box; Anthropic and native local inference are feature-gated.
- Optional features: `browser` (headless Chrome), `anthropic` (native Claude), `local-llm` (native Candle CPU inference).
- **New in 0.3.0:** Provider abstraction with runtime selection (`--provider`/`ARES_PROVIDER`), native Anthropic backend, output validation (`validate_extracted_output`, returned as 422 over HTTP) and groundedness checks (`ungrounded_fields`), `--max-content` cap, additional schemas (`public_tenders`, `tender_list`, `job_board`).
- **New in 0.4.0:** Native local inference via Candle (`local-llm` feature, `--provider local`, `ares model` subcommand).
- **Earlier (0.2.0):** Web crawling, in-memory caching, output formats (json/jsonl/csv/table/jq), proxy rotation, User-Agent rotation, browser stealth mode, TLS backend selection.
