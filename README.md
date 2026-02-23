<p align="center">
  <img src="docs/assets/ares-skill-logo.png" alt="Ares Skill Logo" width="800">
</p>

<h1 align="center">Ares — Claude Code Skill</h1>

<p align="center">
  A <a href="https://support.claude.com/en/articles/12512198-how-to-create-custom-skills">Claude Code Skill</a> for <a href="https://github.com/AndreaBozzo/Ares">Ares</a>, the LLM-powered web scraper.
</p>

> **Ares 0.1.0** releases on crates.io on **February 29, 2026**. This skill will be published to the Anthropic Skill Marketplace shortly after.

## What it does

This skill gives Claude deep knowledge of Ares — its architecture, traits, types, CLI, REST API, schema system, and extension patterns. It helps with:

- Using Ares as a library (building scrapers with `ScrapeService`)
- Running CLI commands and deploying the server
- Creating and managing JSON Schemas for extraction
- Implementing custom `Fetcher`, `Cleaner`, `Extractor`, or `ExtractionStore` traits
- Contributing to the Ares codebase

## Install

### Fastest way

Ask Claude to kindly install the skill for you!

### Claude Code (CLI)

```bash
git clone https://github.com/AndreaBozzo/Ares-Claude-Skill.git
mkdir -p ~/.claude/skills
ln -s "$(pwd)/Ares-Claude-Skill/ares" ~/.claude/skills/ares
```

The skill is now available in all Claude Code sessions. Since it's a symlink, `git pull` is all you need to update.

### Claude (Web/Desktop)

1. Build the ZIP:
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
