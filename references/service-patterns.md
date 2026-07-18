# Service Patterns

Use this reference for generated REST/RPC service structure, ownership, Roze native HTTP, middleware, validation, context, errors, health behavior, and lifecycle behavior.

## REST Layout

Generated REST projects use this stable shape:

```text
config.yaml
Cargo.toml
src/
  main.rs
  application.rs
  config/mod.rs
  route/
  handler/
  logic/
  middleware/
  openapi/mod.rs
  svc/mod.rs
  types/mod.rs
```

Ownership rules:

- `src/logic/**.rs` is the default place for business logic.
- `src/application.rs` is the preserved async service-context configuration hook.
- `src/handler/**` is generator-owned HTTP adaptation: parse request, extract context, call logic, wrap response.
- `src/route/**` is generator-owned route registration.
- `src/types/mod.rs` is generated DTO code from `.api`.
- `src/openapi/mod.rs` is generated OpenAPI glue.
- `src/middleware/mod.rs` is generated aggregation.
- `src/middleware/*.rs` custom middleware files are application-owned and preserved by `--update`.
- `src/svc/mod.rs` owns service dependencies; do not place business workflows there.
- `config.yaml` is local/deployment config and is preserved by `--update`.

REST responses should use `roze-result::ApiResponse` through `roze_http::IntoResponse`. HTTP errors should use Roze error types and helpers instead of custom JSON.

Prefer generated REST adapters and Roze HTTP helpers for routing, request extraction, response wrapping, validation, timeout metadata, context propagation, middleware ordering, health, metrics, and OpenAPI exposure. Add custom Tower layers only after confirming the generated middleware/config surface cannot express the behavior.

Generated services and framework crates should not expose Axum types. Roze native HTTP owns the public REST surface: `roze_http::Router`, method routers, `MethodFilter`, `roze_http::body`, typed extractors such as `Path`, `RawPathParams`, `Query`, `Json`, `Form`, `State`, `Extension`, `OriginalUri`, `MatchedPath`, and `NestedPath`, `IntoResponse`, `Response`, `ErrorResponse`, and Tower-compatible middleware helpers.

Prefer `Router::nest`, `nest_service`, `merge`, `route_service`, method service helpers, `method_not_allowed_fallback`, `route_layer`, `into_make_service`, and `into_make_service_with_connect_info` over introducing Axum just for composition or serving. `Router::route_layer` applies only to matched routes and leaves 404/405 fallbacks untouched; call it only after registering routes. Nested routers receive prefix-stripped current URIs while `OriginalUri`, `MatchedPath`, and `NestedPath` preserve external routing context for middleware and observability.

## REST Operational Endpoints

Generated REST services expose:

- `GET /healthz`
- `GET /readyz`
- `GET /startupz`
- `GET /metrics`
- `POST /reports/exports`
- `GET /reports/exports/:id`
- `DELETE /reports/exports/:id`
- `POST /charts/query`
- `GET /openapi.json`

The generated `ServiceContext` owns a `roze_health::HealthRegistry`. Register dependency checks there when startup-created clients need readiness reporting.

`HealthRegistry` executes registered dependency checks concurrently while preserving report order. Checks have a default timeout and panics are isolated into unhealthy probe results; use `HealthRegistry::with_check_timeout(Duration)` when a service needs a stricter budget.

Generated entrypoints should run under `roze_service::ServiceGroup` when the active checkout supports it. On shutdown, generated lifecycle glue marks the shared `HealthRegistry` as draining so `/readyz` stops reporting ready while the process exits through the unified shutdown path.

`/reports/exports` and `/charts/query` are stable framework-owned interface contracts. Application logic should back them with `roze_report` and Roze query primitives for bounded chart queries, asynchronous CSV/XLSX exports, tenant/auth binding, cancellation, expiry, object storage, and audit behavior instead of inventing parallel report protocols. Register the whitelisted `Arc<dyn roze_report::ReportDataSource>` or `ReportCatalog` from `src/application.rs::configure_context`; an unconfigured source should fail closed with `503`, not return fabricated empty data.

## RPC Layout

Generated RPC projects use this stable shape:

```text
config.yaml
Cargo.toml
build.rs
proto/service.proto
src/
  main.rs
  application.rs
  client/mod.rs
  config/mod.rs
  pb/mod.rs
  server/mod.rs
  svc/mod.rs
  types/mod.rs
  logic/
```

Ownership rules:

- `src/logic/*.rs` is application-owned RPC business behavior and is preserved by `--update`.
- `src/application.rs` is the preserved async service-context configuration hook.
- `src/server/mod.rs` is generator-owned tonic server adaptation.
- `src/client/mod.rs` is generator-owned client code.
- `src/pb/mod.rs`, `build.rs`, and `proto/service.proto` are generator-owned.
- `src/svc/mod.rs` owns service dependencies only.

RPC servers should restore request context with `roze_rpc::rpc::request_context`. RPC clients should accept `&roze_context::Context` as the first business context parameter. RPC errors should convert through `roze_rpc::rpc::status_from_error(err, &request_ctx)` and include standard metadata such as error code, kind, request id, trace id, and locale.

Prefer `roze_rpc` server/client scaffolding, registry integration, timeout/retry/breaker metadata, and error metadata helpers over direct tonic-only wiring when building Roze services. Custom tonic interceptors should preserve Roze context and status metadata contracts.

`rozectl rpc protoc --update` should preserve custom module declarations in `src/logic/mod.rs` as well as application-owned files under `src/logic/**`.

Generated RPC entrypoints should use the same `roze_service::ServiceGroup` lifecycle behavior as REST services when available. Avoid creating separate shutdown channels or ad hoc readiness flags unless the active checkout lacks lifecycle support.

Generated RPC servers register the standard `grpc.health.v1.Health` service. The overall server and generated protobuf service report `NOT_SERVING` during startup, dependency failure, and draining, and `SERVING` when the shared `HealthRegistry` is ready. The generated `grpc-health-sync` lifecycle task refreshes status and publishes `NOT_SERVING` before shutdown.

`roze-config::ServiceConfig` supports a default `rpc_client` and named `rpc_clients.<name>` entries. Use `ServiceConfig::rpc_client_config(name)` or `rpc_client_config_ref(name)` when generated services call multiple upstream RPC services. Each client config must choose exactly one connection mode: `target`, `endpoints`, or `etcd`.

The preferred Roze 1.x workflow manages generated API/RPC service dependencies through `roze-service.yaml` and `rozectl service dependency add/remove/list` plus `rozectl service sync`. The manifest is the source of truth for cross-service RPC dependencies; sync updates the `*-rpc` Cargo path dependency, generated non-secret defaults in `config/roze-dependencies.yaml`, and managed client fields/startup/readiness/accessors in `src/svc/mod.rs`. `service sync --check` should run in CI to catch drift.

Generated `ServiceContext` connects each declared upstream RPC client once at startup, stores the cloneable client in a framework-owned field, registers `rpc:<name>` readiness after connection succeeds, and exposes a `<name>()` accessor. Existing projects can be adopted by `dependency add`, which imports usable local `*-rpc` path dependencies and named `rpc_clients` config; it refuses migration when connection configuration is missing. Do not hand-wire tonic channels in application logic when the generated surface can own them.

The manual `Cargo.toml` plus `rpc_clients.<name>` flow remains available for projects that have not adopted `roze-service.yaml`, but it is no longer the preferred path.

Generated RPC clients pass the inbound `roze_context::Context` into Roze's shared retry executor. Retryable failures use exponential full-jitter backoff, service/method retry budgets, remaining-deadline checks before sleeping, and cancellation checks before and after sleep. `roze_resilience_decisions_total` records `attempt` only immediately before a real retry call; budget, deadline, and cancellation exhaustion use bounded decisions such as `budget_exhausted`, `deadline_exhausted`, and `cancelled`.

## Stream Worker Layout

Generated stream workers use Roze MQ primitives and are beta-level scaffolds. They generate producer, consumer, envelope, config, type, and README files from event-capable contracts.

Ownership rules:

- `src/stream/consumer.rs` owns business message handling and is preserved by `--update`.
- generated envelope/type/config glue is generator-owned.
- stream consumers should use `roze_mq::{Delivery, Subscriber}` and make `ack`, `nack`, retry, dead-letter, and idempotency behavior explicit.
- stream handlers should preserve Roze tracing/context conventions and avoid blocking async worker tasks.
- generated stream worker entrypoints should run under `roze_service::ServiceGroup`, propagate shutdown into consumer tasks, and stop workers before returning.

## API Contract Syntax

Roze `.api` files support:

- `syntax = "v1"`
- `info (...)`
- `type Name { ... }` and grouped `type (...)`
- `service name { ... }`
- `@server`, `@handler`, `@doc`, and `@middleware`
- `@permission` before REST routes or RPC methods
- imports
- HTTP methods `get`, `post`, `put`, `patch`, and `delete`
- route signatures with optional request and response types

Routes without request types receive `EmptyReq`. Routes without response types receive `EmptyResp`.

Do not generate REST from contracts containing RPC-only definitions, or RPC from contracts containing REST-only route definitions.

## Permissions And Idempotency

Use `@permission` before REST routes or RPC methods to declare required permissions. Generated REST handlers call `roze_middleware::enforce_permissions` after authentication; generated RPC servers call `roze_rpc::rpc::enforce_permissions`. All declared permissions are required. Permission values come from Roze request context metadata and propagate as `x-roze-meta-permissions`; REST OpenAPI operations expose them as `x-roze-permissions`.

Generated application logic should read identity and authorization state through stable helpers such as `current_subject`, `current_user_id`, `current_admin_id`, `current_tenant`, `current_roles`, `current_permissions`, and `current_scope`. Applications still own authentication and must populate subject, tenant, roles, scopes, and permissions in `roze_context::Context`.

Use `@middleware idempotency` before mutating REST routes or RPC methods to opt into duplicate-request handling. REST requires an `Idempotency-Key` header and RPC requires `idempotency-key` metadata. Generated `ServiceContext` holds `Arc<dyn roze_middleware::IdempotencyStore>` and provides `with_idempotency_store` for a persistent Redis or database adapter; the in-memory default is only for local development and tests.

Idempotency records include key scope, canonical request fingerprint, processing lease, and completed JSON response. Matching completed requests replay, live leases conflict, expired leases can be reclaimed, different requests with the same key conflict, and failed logic releases unfinished records. Preserve stable error codes such as `IDEMPOTENCY_MISSING_KEY`, `IDEMPOTENCY_IN_FLIGHT`, `IDEMPOTENCY_KEY_REUSED`, `IDEMPOTENCY_STORAGE_UNAVAILABLE`, and `IDEMPOTENCY_REPLAY_INVALID`.

## Field Sources And Types

Fields can bind from `path`, `query`, `form`, `header`, or JSON tags. Without a source tag, `rozectl` infers path parameters from `:name` route segments and otherwise treats fields as JSON body fields.

Generated DTO fields use Rust `snake_case` and `serde(rename = "...")` when wire names differ.

Common scalar mappings:

- `string` or `String` to `String`
- `int` to `i64`
- `uint` to `u64`
- `bool` to `bool`
- explicit Rust-like numeric types to the same Rust type
- `[]T` or `Vec<T>` to `Vec<T>`
- `map[K]V` or `HashMap<K,V>` to `std::collections::HashMap<K,V>`

## Validation

`rozectl` reads Go validator-style tags. Prefer native Rust `validator` derive attributes where possible, then generated request-level checks for rules Rust derive cannot express.

Important mappings:

- `required` on strings/containers maps to non-empty length checks.
- `min`, `max`, `len`, `gt`, `gte`, `lt`, and `lte` map to length/range checks depending on type.
- `email`, `url`, `ip`, `ipv4`, `ipv6`, `contains`, and `excludes` map to native validator support.
- `oneof`, `startswith`, `endswith`, `alpha`, `alphanum`, `ascii`, `numeric`, case rules, cross-field comparisons, and conditional required rules are request-level checks.
- `optional` and `omitempty` skip generated validation.

Cross-field comparisons should be generated only when both fields map to the same Rust type.

## Middleware

Service-wide REST middleware lives under `rest.middlewares` in `config.yaml`. Built-ins include recover, trace, stat, prometheus, cors, timeout, max connections, shedding, gunzip, and request body limits.

Route-scoped middleware declared in `.api` is resolved as built-in first. Unknown names generate custom application middleware files.

Built-in names include `auth`, `jwt`, `trace`, `recover`, `stat`, `prometheus`, `metrics`, `cors`, `timeout`, `rate_limit`, `breaker`, `max_conns`, `shedding`, `gunzip`, `body_limit`, and `idempotency`.

Prefer built-in middleware names in `.api` annotations and `config.yaml` before generating custom middleware files. Custom middleware should focus on product-specific behavior and should still preserve Roze request context, tracing spans, metrics labels, error shape, and timeout/cancellation behavior.

Business logic should log with `tracing` macros directly. Do not pass or construct trace ids manually; Roze middleware carries trace ids in the request span.

Generated REST, RPC, and stream entrypoints emit structured lifecycle logs for configuration readiness, dependency setup, registry or subscription readiness, shutdown, stop, and failure. Native HTTP logs request start/completion; RPC governance logs method start/completion/cancellation. Safe `RUST_LOG=debug` framework logs may include route matches, middleware plans, retry decisions, stream ack/nack, model query kinds, and ServiceGroup phase changes, but must not include request/message bodies, auth values, SQL arguments, fallback payloads, or dependency error messages.

## Service Context And Lifecycle

`src/svc/mod.rs` is the place for shared clients and framework state, not business workflows. Use `src/application.rs::configure_context` for application-specific attachment of report data sources and other resources after the generated `ServiceContext` is constructed. Prefer generated `ServiceContext` slots and Roze crates for:

- health registry and dependency readiness checks
- optional cache clients
- optional NATS JetStream/MQ clients
- outbox relay state and persistent outbox stores
- optional object storage clients
- idempotency stores
- managed upstream RPC clients discovered by `rozectl`
- model/search clients generated by `rozectl`
- registry/config/gateway clients where the generated surface supports them

`roze_service::ServiceGroup` provides `RuntimeService`, `FnService`, `ServiceGroupConfig`, `ServiceGroupHandle`, and `LifecycleState` with `Starting`, `Running`, `Draining`, `Stopped`, and `Failed` phases. Use it for HTTP/RPC/consumer/job/background services that must share one shutdown signal and stop-hook timeout. When jobs are involved, prefer the `roze-job::JobService` adapter instead of hand-rolled loops.

Do not keep `/readyz` green during shutdown. If adding custom background services, wire them into the shared lifecycle so readiness, draining, stop hooks, lifecycle snapshots, and task timeouts remain observable and testable.

## Generated Ops Assets

Generated REST and RPC services may include `ops/production-evidence.md`, `ops/governance-baseline.yaml`, Prometheus/Grafana/SLO assets, failure-injection, rollout, incident-response, capacity, security, runtime-hardening, error, communication, cache, data-access, and interface-governance plans, plus `ops/production-verify.ps1`, `ops/production-verify.sh`, `ops/ci-evidence-policy.yaml`, `ops/evidence-manifest.yaml`, and `.github/workflows/roze-production-verify.yml`.

Treat those files as generated production gates and review artifacts, not business logic. In CI or release checks, run:

```powershell
powershell -ExecutionPolicy Bypass -File ops\production-verify.ps1
```

On Linux CI, run `bash ops/production-verify.sh`. Treat CI success as a precondition, not a replacement, for soak and failure-injection evidence.

Keep these generated assets aligned with the service boundary whenever `.api`, `.proto`, config, generated layout, or production-readiness rules change.
