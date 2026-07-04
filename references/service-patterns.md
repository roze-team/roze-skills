# Service Patterns

Use this reference for generated REST/RPC service structure, ownership, middleware, validation, context, errors, and health behavior.

## REST Layout

Generated REST projects use this stable shape:

```text
config.yaml
Cargo.toml
src/
  main.rs
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
- `src/handler/**` is generator-owned HTTP adaptation: parse request, extract context, call logic, wrap response.
- `src/route/**` is generator-owned route registration.
- `src/types/mod.rs` is generated DTO code from `.api`.
- `src/openapi/mod.rs` is generated OpenAPI glue.
- `src/middleware/mod.rs` is generated aggregation.
- `src/middleware/*.rs` custom middleware files are application-owned and preserved by `--update`.
- `src/svc/mod.rs` owns service dependencies; do not place business workflows there.
- `config.yaml` is local/deployment config and is preserved by `--update`.

REST responses should use `roze-result::ApiResponse`. HTTP errors should use Roze error types and helpers instead of custom JSON.

## REST Operational Endpoints

Generated REST services expose:

- `GET /healthz`
- `GET /readyz`
- `GET /startupz`
- `GET /metrics`
- `GET /openapi.json`

The generated `ServiceContext` owns a `roze_health::HealthRegistry`. Register dependency checks there when startup-created clients need readiness reporting.

## RPC Layout

Generated RPC projects use this stable shape:

```text
config.yaml
Cargo.toml
build.rs
proto/service.proto
src/
  main.rs
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
- `src/server/mod.rs` is generator-owned tonic server adaptation.
- `src/client/mod.rs` is generator-owned client code.
- `src/pb/mod.rs`, `build.rs`, and `proto/service.proto` are generator-owned.
- `src/svc/mod.rs` owns service dependencies only.

RPC servers should restore request context with `roze_rpc::rpc::request_context`. RPC clients should accept `&roze_context::Context` as the first business context parameter. RPC errors should convert through `roze_rpc::rpc::status_from_error(err, &request_ctx)` and include standard metadata such as error code, kind, request id, trace id, and locale.

## API Contract Syntax

Roze `.api` files support:

- `syntax = "v1"`
- `info (...)`
- `type Name { ... }` and grouped `type (...)`
- `service name { ... }`
- `@server`, `@handler`, `@doc`, and `@middleware`
- imports
- HTTP methods `get`, `post`, `put`, `patch`, and `delete`
- route signatures with optional request and response types

Routes without request types receive `EmptyReq`. Routes without response types receive `EmptyResp`.

Do not generate REST from contracts containing RPC-only definitions, or RPC from contracts containing REST-only route definitions.

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

Business logic should log with `tracing` macros directly. Do not pass or construct trace ids manually; Roze middleware carries trace ids in the request span.
