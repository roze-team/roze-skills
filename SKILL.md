---
name: roze-skills
description: Roze Rust microservice framework knowledge for AI agents. Use when working with roze-team/roze, rozectl, Roze .api or .proto contracts, generated Rust REST/RPC services, Roze crates such as roze-http/roze-rpc/roze-config/roze-mq/roze-search, service governance, middleware, model/search generation, OpenAPI/SDK generation, production smoke tests, or troubleshooting Roze project conventions.
---

# Roze Skills

Use this skill to build, modify, review, or troubleshoot projects based on the Roze Rust microservice framework.

Treat the active Roze checkout as the source of truth. This skill gives stable guidance and navigation, but agents must inspect the target repository before changing code, generated templates, docs, tests, or public command behavior.

Roze is IDL-first and convention-driven:

- Define REST routes, RPC methods, DTOs, and annotations in `.api` or `.proto` contracts.
- Use `rozectl` to generate framework-owned boundaries.
- Prefer Roze built-in generators, crates, middleware, config, governance, context, error, health, metrics, tracing, registry, cache, MQ, DTM, and search/model helpers before introducing custom infrastructure.
- Put business behavior in generated extension points, especially `src/logic/**`, custom middleware, and application-owned modules.
- Preserve generated ownership boundaries during regeneration.
- Validate changes with targeted crate tests and smoke scripts.

## Load References As Needed

Read only the files relevant to the current task:

- [references/rozectl-workflows.md](references/rozectl-workflows.md): use for `rozectl` commands, API/RPC generation, update vs force, OpenAPI, SDK, mock, contract, docker, kube, doctor, and dev stack workflows.
- [references/service-patterns.md](references/service-patterns.md): use for generated REST/RPC layouts, handler/server adapters, logic placement, service context, validation, context propagation, errors, middleware, and health endpoints.
- [references/data-search-patterns.md](references/data-search-patterns.md): use for model generation, Toasty, SeaORM, SQL/Mongo inspection, repository boundaries, search DSL/inspection, and Elasticsearch/OpenSearch/Meilisearch support.
- [references/governance-operations.md](references/governance-operations.md): use for config, registry, gateway, MQ, DTM, cache, metrics, tracing, production checks, smoke scripts, and hot-path expectations.
- [references/troubleshooting.md](references/troubleshooting.md): use when diagnosing generator issues, mixed API/RPC contracts, regeneration overwrites, validator mappings, config failures, dependency stack problems, or tests to run.

## Core Workflow

1. Inspect the target Roze checkout first: read `README.md`, `docs/project-standards.md`, relevant `docs/usage/*`, `docs/contracts/*`, `apps/rozectl`, and the touched `crates/roze-*` modules.
2. Identify whether the task touches generated code, runtime crates, application logic, docs, or tests.
3. Check whether Roze already provides the needed capability through `rozectl`, a generated extension point, or a `roze-*` crate before writing custom framework code.
4. If changing generated output, update the generator/templates/tests in `apps/rozectl` rather than hand-editing generated glue.
5. If changing a generated service, keep application behavior in `src/logic/**`, `src/middleware/*.rs`, `src/model/_ext.rs`, or app-owned modules.
6. Prefer `--update` for regeneration. Use `--force` only for intentional full rebuilds.
7. Run focused verification. For generator work, include `cargo test -p rozectl -- --skip postgres --skip mysql --skip mongo` unless the task needs real database inspect coverage.
8. Sync docs and this skill when public commands, runtime contracts, generated layouts, config fields, or production behavior change.

## Roze Principles

- IDL first: `.api` and `.proto` are the source of boundary contracts.
- Rust native: generated services use Axum/Tower for REST, tonic/prost for RPC, and Roze crates for framework behavior.
- Generated boundaries stay explicit: route registration, handlers, RPC adapters, protobuf include modules, DTOs, OpenAPI, and deployment files are generator-owned.
- Application logic stays explicit: complex SQL, transactions, authorization, permission checks, domain validation, search ranking, and business orchestration are not invented by the generator.
- Framework behavior should come from Roze first: prefer built-in middleware, `roze_context`, `roze_result`, `roze_error`, `roze_health`, `roze_config`, `roze_rpc`, `roze_gateway`, `roze_mq`, `roze_cache`, `roze_local_cache`, `roze_singleflight`, `roze_search`, and generated repositories before adding local equivalents.
- Context, errors, logs, metrics, config, registry, and middleware behavior should be uniform across REST, RPC, gateway, MQ, and jobs.

## Common Commands

Use local source dependencies inside the Roze repository:

```bash
cargo run -p rozectl -- api generate example/user.api --out apps/roze-example --roze-source path
cargo run -p rozectl -- rpc generate example/user.api --out apps/user-rpc --roze-source path
cargo run -p rozectl -- api generate example/user.api --out apps/roze-example --update --roze-source path
```

Use installed `rozectl` outside the repository:

```bash
rozectl api generate example/user.api --out services/user-api
rozectl rpc protoc example/user.proto --out services/user-rpc
rozectl openapi generate example/user.api --out docs/openapi.json
```

## Do Not

- Do not put business logic in generated handlers, route files, RPC server/client adapters, protobuf include modules, DTO files, or OpenAPI glue.
- Do not hand-edit generated files when a template or generator test should change instead.
- Do not mix REST routes and RPC methods in one generation path; use the appropriate `api` or `rpc` command.
- Do not construct ad hoc HTTP error JSON or gRPC status metadata outside Roze error helpers.
- Do not replace built-in Roze middleware, context, config, health, registry, cache, MQ, DTM, metrics, tracing, or search/model helpers with local implementations unless the active checkout lacks the needed capability and the gap is documented.
- Do not add public behavior without updating docs and tests.
