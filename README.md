# Ares — Claude Code Skill

A [Claude Code Skill](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills) for [Ares](https://github.com/AndreaBozzo/Ares), the LLM-powered web scraper.

## What it does

This skill gives Claude deep knowledge of Ares — its architecture, traits, types, CLI, REST API, schema system, and extension patterns. It helps with:

- Using Ares as a library (building scrapers with `ScrapeService`)
- Running CLI commands and deploying the server
- Creating and managing JSON Schemas for extraction
- Implementing custom `Fetcher`, `Cleaner`, `Extractor`, or `ExtractionStore` traits
- Contributing to the Ares codebase

## Install

1. Download the latest release ZIP, or build it yourself:
   ```bash
   cd ares && zip -r ../ares-skill.zip .
   ```
2. Upload to Claude → **Settings > Capabilities > Custom Skills**

## Structure

```
ares/
├── SKILL.md                        # Entry point — overview, traits, types, quick start
└── references/
    ├── architecture.md             # Pipeline, crate graph, services, error handling
    ├── schemas.md                  # JSON Schema system, registry, versioning
    ├── extending.md                # Implementing custom trait impls
    ├── cli-and-server.md           # CLI commands, REST API, env vars, deployment
    └── contributing.md             # Dev setup, testing, CI, code style
```

## License

Apache-2.0
