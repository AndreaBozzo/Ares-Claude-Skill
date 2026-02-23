# JSON Schema System

## Overview

Ares uses JSON Schemas to instruct the LLM on what structured data to extract. Schemas are passed to the `Extractor` trait and used for structured output from the LLM API.

## SchemaResolver

Located in `ares-core::schema`. Resolves schema references to loaded JSON:

```rust
let resolver = SchemaResolver::new("schemas/");

// Option 1: Direct file path
let schema = resolver.resolve("schemas/blog/1.0.0.json")?;

// Option 2: name@version
let schema = resolver.resolve("blog@1.0.0")?;

// Option 3: name@latest (uses registry.json)
let schema = resolver.resolve("blog@latest")?;
```

Returns a `ResolvedSchema { path, name, schema }`.

## Directory Structure

```
schemas/
├── registry.json              # Maps schema names to latest versions
├── blog/
│   ├── 1.0.0.json
│   └── 2.0.0.json
└── github_repo/
    └── 1.0.0.json
```

## Registry Format

`registry.json` maps each schema name to its latest version:

```json
{
  "blog": "1.0.0",
  "github_repo": "1.0.0"
}
```

Updated automatically when creating schemas via `SchemaResolver::create_schema()` or the `POST /v1/schemas` API endpoint.

## Writing a Schema

Schemas follow the JSON Schema standard. The LLM receives the schema as its structured output format.

Example (`schemas/blog/1.0.0.json`):

```json
{
  "title": "BlogPage",
  "type": "object",
  "properties": {
    "title": { "type": "string" },
    "author": { "type": "string" },
    "publish_date": { "type": "string" },
    "summary": { "type": "string" },
    "tags": {
      "type": "array",
      "items": { "type": "string" }
    },
    "hero_image": { "type": "string" },
    "url": { "type": "string" }
  },
  "required": ["title", "author", "publish_date", "summary", "tags", "hero_image", "url"]
}
```

## Best Practices

- Use descriptive property names — the LLM uses these to understand what to extract
- Mark all expected fields as `required` to avoid missing data
- Use `"type": "array"` with `"items"` for lists (tags, categories, links)
- Keep schemas focused — one schema per page type (blog, product, profile)
- Use `title` at the root level for clarity (e.g., `"title": "BlogPage"`)
- Version schemas with semver: `name/version.json`

## Creating Schemas Programmatically

```rust
let resolver = SchemaResolver::new("schemas/");
resolver.create_schema("product", "1.0.0", &schema_json)?;
// Creates schemas/product/1.0.0.json and updates registry.json
```

## Server Schema Endpoints

- `GET /v1/schemas` — List all schemas with versions
- `GET /v1/schemas/{name}/{version}` — Get a specific schema
- `POST /v1/schemas` — Create a new schema version

```bash
# Create via API
curl -X POST http://localhost:3000/v1/schemas \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "product", "version": "1.0.0", "schema": {"type": "object", ...}}'
```

## Helper Functions

- `derive_schema_name(path)` — Extracts name from file path: `"schemas/blog.json"` → `"blog"`
- `SchemaResolver::list_schemas()` — Returns `Vec<SchemaEntry>` with name, latest_version, versions
