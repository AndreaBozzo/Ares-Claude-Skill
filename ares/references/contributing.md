# Contributing to Ares

## Dev Environment Setup

**Requirements:**
- Rust 1.88+ (edition 2024)
- PostgreSQL 16 (via Docker or local install)
- Optional: Chromium/Chrome (for `--features browser`)

```bash
# Clone
git clone https://github.com/AndreaBozzo/Ares.git
cd Ares

# Copy env template
cp .env.example .env
# Edit .env with your API key and database URL

# Start PostgreSQL
make docker-up

# Run migrations
make migrate

# Build
cargo build
```

## Makefile Targets

| Target | Description |
|---|---|
| `make all` | fmt + clippy + test |
| `make build` | Debug build |
| `make release` | Release build (LTO + strip) |
| `make test` | All tests |
| `make test-unit` | Unit tests only (`--lib --bins`) |
| `make test-integration` | Integration tests only (requires Docker/PostgreSQL) |
| `make fmt` | Format code |
| `make fmt-check` | Check formatting |
| `make clippy` | Run clippy with `-D warnings` and `--all-features` |
| `make docker-up` | Start PostgreSQL via Docker Compose |
| `make docker-down` | Stop PostgreSQL |
| `make migrate` | Run SQL migrations with tracking |
| `make docker-build` | Build production Docker image |

## Testing Strategy

### Unit Tests
Every module has inline `#[cfg(test)]` tests. Mock implementations in `ares-core::testutil`:

```rust
use ares_core::testutil::*;

// Create mocks
let fetcher = MockFetcher::new("<html>content</html>");
let cleaner = MockCleaner::passthrough();
let extractor = MockExtractor::new(serde_json::json!({"title": "Test"}));
let store = MockStore::empty();

// Inject into ScrapeService
let svc = ScrapeService::with_store(fetcher, cleaner, extractor, store, "model".into());
```

### Integration Tests
Located in `crates/ares-db/tests/integration/` and `crates/ares-api/tests/integration/`.

Require a running PostgreSQL. In CI, a Postgres container is started automatically.

The server integration tests use `testcontainers` for isolated database instances.

## CI Pipeline

GitHub Actions on every push/PR:
1. `cargo fmt --check`
2. `cargo clippy --all-targets --all-features -- -D warnings`
3. `cargo test --lib --bins` (unit tests)
4. Integration tests with Postgres service container
5. `cargo deny check` (dependency security audit)

## Code Style Conventions

- **Trait-based abstraction:** All external I/O behind traits. Never call HTTP/DB directly from business logic.
- **Error handling:** Use `AppError` variants, not `anyhow` in library code. Set `retryable` flag appropriately for LLM errors.
- **Async:** All trait methods use `impl Future<Output = ...> + Send` (not `async fn` in trait) for compatibility.
- **Builder pattern:** Use `with_*` methods for optional configuration (e.g., `with_skip_unchanged`, `with_jitter`).
- **Testing:** Every public function has tests. Use mocks from `testutil`, not real APIs.
- **Logging:** Use `tracing` macros (`tracing::info!`, `tracing::warn!`). Include structured fields.
- **No `unwrap()` in library code** — propagate errors via `?` or map to `AppError`.

## Project Structure

```
crates/
├── ares-core/src/
│   ├── lib.rs              # Module re-exports
│   ├── traits.rs           # Core trait definitions
│   ├── models.rs           # Data types
│   ├── error.rs            # AppError enum
│   ├── scrape.rs           # ScrapeService pipeline
│   ├── job.rs              # ScrapeJob, JobStatus, RetryConfig
│   ├── job_queue.rs        # JobQueue trait
│   ├── worker.rs           # WorkerService
│   ├── circuit_breaker.rs  # CircuitBreaker
│   ├── throttle.rs         # ThrottledFetcher
│   ├── schema.rs           # SchemaResolver
│   └── testutil.rs         # Mock implementations
├── ares-client/src/
│   ├── fetcher.rs          # ReqwestFetcher
│   ├── browser_fetcher.rs  # BrowserFetcher (feature: browser)
│   ├── cleaner.rs          # HtmdCleaner
│   └── llm.rs              # OpenAiExtractor, OpenAiExtractorFactory
├── ares-db/src/
│   ├── config.rs           # DatabaseConfig
│   ├── database.rs         # Database wrapper
│   ├── repository.rs       # ExtractionRepository
│   └── job_repository.rs   # ScrapeJobRepository
├── ares-api/src/
│   ├── main.rs             # Server entry point
│   ├── routes.rs           # Axum handlers
│   ├── dto.rs              # Request/response types
│   ├── state.rs            # AppState
│   ├── auth.rs             # Bearer token middleware
│   ├── error.rs            # ApiError
│   └── openapi.rs          # Swagger config
└── ares-cli/src/
    └── main.rs             # CLI with clap
```
