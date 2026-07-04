# Roze Skills

AI-optimized knowledge base for working with the [Roze](https://github.com/roze-team/roze) Rust microservice framework.

This repository packages Roze project conventions, `rozectl` workflows, generated service layouts, runtime boundaries, and troubleshooting notes as an Agent Skill. It is intended for coding agents that need to build, modify, review, or debug Roze-based services with less rediscovery and fewer convention mistakes.

## What This Skill Covers

- `rozectl` API/RPC/model/search/OpenAPI/SDK/deployment workflows
- Roze `.api` and `.proto` contract-first development
- Generated REST service structure with Axum, Tower, and Roze middleware
- Generated RPC service structure with tonic/prost and Roze context/error metadata
- Application-owned vs generator-owned file boundaries
- Toasty and SeaORM model generation patterns
- Elasticsearch, OpenSearch, and Meilisearch search generation patterns
- Config, registry, gateway, MQ, cache, DTM, metrics, tracing, and production smoke checks
- Common troubleshooting paths for regeneration, validation, middleware, context, and local dependencies

## Install

Install as a project-level skill:

```bash
git submodule add https://github.com/roze-team/roze-skills.git .codex/skills/roze-skills
```

Or clone directly:

```bash
git clone https://github.com/roze-team/roze-skills.git .codex/skills/roze-skills
```

Install as a personal skill:

```bash
git clone https://github.com/roze-team/roze-skills.git ~/.codex/skills/roze-skills
```

## Usage

Ask your agent to use the skill when working on Roze tasks:

```text
Use $roze-skills to add a new Roze REST endpoint from this .api file.
Use $roze-skills to review this rozectl generator change.
Use $roze-skills to diagnose why --update rewrote a file I expected to preserve.
Use $roze-skills to generate model/search scaffolding guidance for this service.
```

The skill should also trigger naturally when the task mentions `roze-team/roze`, `rozectl`, Roze `.api` files, generated REST/RPC services, or `crates/roze-*` runtime modules.

## Structure

```text
roze-skills/
  SKILL.md
  agents/openai.yaml
  references/
    rozectl-workflows.md
    service-patterns.md
    data-search-patterns.md
    governance-operations.md
    troubleshooting.md
```

`SKILL.md` is the entry point. It gives the agent the core workflow and tells it which reference file to load for the task at hand.

The `references/` files are intentionally split by domain so agents can load only the context they need.

## Source Of Truth

This skill summarizes Roze conventions, but the active Roze checkout remains the source of truth. Agents should inspect the target repository before editing code, especially:

- `README.md`
- `docs/project-standards.md`
- `docs/usage/*`
- `docs/contracts/*`
- `apps/rozectl`
- touched `crates/roze-*` modules

When Roze code and this skill disagree, update the skill after confirming the current behavior in code and tests.

## Maintenance Checklist

Update this skill when Roze changes:

- public `rozectl` commands or flags
- generated REST/RPC/model/search layouts
- ownership rules for generated vs application files
- runtime contracts for context, errors, middleware, registry, gateway, MQ, cache, or DTM
- verification commands, smoke scripts, or production expectations
- docs that are used as agent guidance

Keep `SKILL.md` concise and move detailed patterns into `references/`.
