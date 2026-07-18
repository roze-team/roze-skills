---
name: roze-skills
description: Roze Rust microservice framework knowledge for AI agents. Use when working with roze-team/roze, rozectl, Roze .api, .proto, .ent, model/search schemas, generated Rust REST/RPC/stream services, Roze native HTTP/roze-http, Roze crates such as roze-rpc/roze-service/roze-config/roze-gateway/roze-mq/roze-search/roze-query/roze-migration/roze-orm/roze-report, managed upstream RPC clients and roze-service.yaml dependency manifests, Rust ownership/lifetime/error/async/concurrency/performance/unsafe issues inside Roze code, service lifecycle/bootstrap, health/readiness probes, gRPC health, service governance, gateway native HTTP policy, permission/idempotency middleware, persistent outbox, cache consistency, object storage, reporting/charts and ReportDataSource hooks, model/search generation, ent-style query/create/update/delete/projection/aggregate/hook/policy/mixin builders, OpenAPI/Web SDK generation, contract diff, gate checks, generator extensions, API plugins, generated ops assets, generated production reference systems, release gates, production evidence verification/promotion, smoke/soak tests, or troubleshooting Roze project conventions.
---

# Roze Skills

Use this skill to build, modify, review, or troubleshoot projects based on the Roze Rust microservice framework.

Treat the active Roze checkout as the source of truth. This skill gives stable guidance and navigation, but agents must inspect the target repository before changing code, generated templates, docs, tests, or public command behavior.

Roze is IDL-first and convention-driven:

- Define REST routes, RPC methods, DTOs, and annotations in `.api` or `.proto` contracts.
- Use `rozectl` to generate framework-owned boundaries.
- Prefer Roze built-in generators, crates, middleware, config, governance, context, error, health, metrics, tracing, registry, gateway, cache, MQ, DTM, RPC-client, and search/model helpers before introducing custom infrastructure.
- Put business behavior in generated extension points, especially `src/logic/**`, custom middleware, `src/application.rs`, and application-owned modules.
- Preserve generated ownership boundaries during regeneration.
- Validate changes with targeted crate tests and smoke scripts.

## Load References As Needed

Read only the files relevant to the current task:

- [references/rozectl-workflows.md](references/rozectl-workflows.md): use for `rozectl` commands, API/RPC generation, service dependency manifests, update vs force, validate/format/diff, OpenAPI, Web SDK, mock, contract check/diff, gate checks, generator extensions, API plugins, quickstart/template/migrate, docker, kube, env/upgrade/completion, doctor, generated ops assets, and dev stack workflows.
- [references/service-patterns.md](references/service-patterns.md): use for generated REST/RPC/stream layouts, Roze native HTTP handlers/router/extractors/responses, server adapters, managed upstream RPC clients, logic placement, service context, validation, context propagation, permissions, idempotency, errors, middleware, health endpoints, lifecycle/bootstrap, and generated ownership.
- [references/data-search-patterns.md](references/data-search-patterns.md): use for `.ent` model generation, Toasty, SeaORM, SQL/Mongo inspection, model migrations, deterministic model fixtures/seeds, ent-style repository/client/query/mutation/projection/aggregate/hook/policy/mixin builders, decimal/smallint mappings, search DSL/inspection, and Elasticsearch/OpenSearch/Meilisearch support.
- [references/governance-operations.md](references/governance-operations.md): use for config, registry, gateway, MQ, DTM, persistent outbox, cache consistency, object storage, read-model composition, lifecycle, metrics, tracing, release gates, production evidence, smoke/soak scripts, and hot-path expectations.
- [references/rust-engineering.md](references/rust-engineering.md): use for Rust ownership/lifetime, error handling, async/concurrency, resource lifecycle, dependencies, performance, unsafe/FFI, refactoring, and code-review patterns that apply inside Roze crates or generated services.
- [references/troubleshooting.md](references/troubleshooting.md): use when diagnosing generator issues, mixed API/RPC contracts, regeneration overwrites, validator mappings, config failures, dependency stack problems, or tests to run.

## Core Workflow

1. Inspect the target Roze checkout first: read `README.md`, `docs/project-standards.md`, relevant `docs/usage/*`, `docs/contracts/*`, `apps/rozectl`, and the touched `crates/roze-*` modules.
2. Identify whether the task touches generated code, runtime crates, application logic, docs, or tests.
3. Check whether Roze already provides the needed capability through `rozectl`, a generated extension point, or a `roze-*` crate before writing custom framework code.
4. If changing generated output, update the generator/templates/tests in `apps/rozectl` rather than hand-editing generated glue.
5. If changing a generated service, keep application behavior in `src/logic/**`, `src/application.rs`, `src/middleware/*.rs`, `src/model/*_ext.rs`, `src/stream/consumer.rs`, or app-owned modules.
6. Prefer `--update` for regeneration. Use `--force` only for intentional full rebuilds.
7. For Roze 1.x, treat public APIs, CLI commands, generated layouts, config schemas, metrics, and documented runtime ordering as SemVer-governed stable contracts. Keep operational evidence separate: pin the Roze Git revision or signed tag, review generated diffs, run documented smoke/release gates, and do not claim battle-tested 24h/72h behavior unless production evidence exists.
8. Run focused verification. For generator work, include `cargo test -p rozectl -- --skip postgres --skip mysql --skip mongo` unless the task needs real database inspect coverage.
9. Sync docs and this skill when public commands, runtime contracts, generated layouts, config fields, lifecycle behavior, smoke scripts, or production behavior change.

## Roze Principles

- IDL first: `.api` and `.proto` are the source of boundary contracts.
- Rust native: generated services use Roze native HTTP/Tower for REST, tonic/prost for RPC, and Roze crates for framework behavior.
- Generated boundaries stay explicit: route registration, handlers, RPC adapters, protobuf include modules, DTOs, OpenAPI, and deployment files are generator-owned.
- Application logic stays explicit: complex SQL, transactions, authorization, permission checks, domain validation, search ranking, and business orchestration are not invented by the generator.
- Framework behavior should come from Roze first: prefer built-in middleware, `roze_context`, `roze_result`, `roze_error`, `roze_health`, `roze_config`, `roze_rpc`, `roze_gateway`, `roze_http`, `roze_mq`, `roze_cache`, `roze_local_cache`, `roze_singleflight`, `roze_query`, `roze_storage`, `roze_transaction`, `roze_search`, `roze_report`, generated RPC clients, service dependency manifests, and generated repositories before adding local equivalents.
- Generated service entrypoints should use Roze lifecycle/bootstrap primitives where available, especially `roze_service::ServiceGroup`, health/readiness state, graceful shutdown, and generated readiness/draining behavior.
- Context, errors, logs, metrics, config, registry, and middleware behavior should be uniform across REST, RPC, gateway, MQ, and jobs.

## Common Commands

Use local source dependencies inside the Roze repository:

```bash
cargo run -p rozectl -- api generate example/user.api --out apps/roze-example --roze-source path
cargo run -p rozectl -- rpc generate example/user.api --out apps/user-rpc --roze-source path
cargo run -p rozectl -- api generate example/user.api --out apps/roze-example --update --roze-source path
cargo run -p rozectl -- stream gen example/events.api --out apps/user-stream --roze-source path
cargo run -p rozectl -- service dependency add order --project apps/payment --crate shop-order-rpc --path ../shop-order-rpc --endpoint 127.0.0.1:4002
```

Use installed `rozectl` outside the repository:

```bash
rozectl api generate example/user.api --out services/user-api
rozectl rpc protoc example/user.proto --out services/user-rpc
rozectl model generate model/schema.ent --out services/user-api --format ent
rozectl openapi generate example/user.api --out docs/openapi.json
rozectl service sync --project services/user-api --check
```

## Do Not

- Do not put business logic in generated handlers, route files, RPC server/client adapters, protobuf include modules, DTO files, or OpenAPI glue.
- Do not hand-edit generated files when a template or generator test should change instead.
- Do not mix REST routes and RPC methods in one generation path; use the appropriate `api` or `rpc` command.
- Do not construct ad hoc HTTP error JSON or gRPC status metadata outside Roze error helpers.
- Do not expose Axum types from generated services or framework crates; use `roze_http` request, response, router, extractor, middleware, and body APIs.
- Do not replace built-in Roze middleware, context, config, health, registry, cache, MQ, DTM, metrics, tracing, or search/model helpers with local implementations unless the active checkout lacks the needed capability and the gap is documented.
- Do not add public behavior without updating docs and tests.
