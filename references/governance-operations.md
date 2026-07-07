# Governance And Operations

Use this reference for production behavior across config, middleware, registry, gateway, MQ, cache, DTM, metrics, tracing, and smoke verification.

## Configuration

Applications should use `roze-config::ServiceConfig`. Local files default to `config.yaml`; environment variables and config center values are overrides or hot-update sources.

Prefer Roze config loading, environment override, config center, typed service config, and hot-reload hooks before introducing application-local config systems.

Config center reloads must preserve the last valid configuration when a new payload fails to parse. Rebuild hot-updated subsystems by changed section or config signature instead of restarting everything for unrelated changes.

When adding config fields, update example configs, contract docs, and tests.

## Context, Errors, Logs, And Metrics

REST, RPC, gateway, MQ, and job entry points should propagate standard Roze context values:

- request id
- trace id
- tenant
- locale
- auth subject
- metadata

If request id or trace id is missing, the entry point should generate and return it where the protocol supports headers/metadata.

HTTP errors use `RozeError` and `roze-result::ApiResponse`. RPC errors use `roze_rpc::rpc::status_from_error`.

Prefer Roze context, error, result, tracing, and metrics helpers at every protocol boundary. Do not invent parallel request-id, trace-id, locale, tenant, response-envelope, or error-code conventions inside one service.

Metrics conventions:

- HTTP route metrics: `roze_http_route_*`
- RPC method metrics: `roze_rpc_method_*`
- gateway metrics: `roze_gateway_*`
- MQ metrics include topic, group, partition, offset, attempt, and outcome labels where relevant

Logs for request-scoped work should include request id or trace id through spans. Governance events such as hot reload, retry, breaker, rate limit, dead letter, and fallback should use structured fields.

## Registry, Gateway, And Route Governance

Service discovery should go through `roze_rpc::registry`, including memory, DNS, etcd, consul, and cached resolver modes.

Prefer Roze registry, gateway, retry, timeout, breaker, rate-limit, fallback, and outlier detection surfaces before adding local service-discovery or traffic-governance code.

Gateway upstreams can come from static upstream config or registry discovery. Dynamic discovery should filter by instance tags, health, and outlier state before weighted selection.

Route-level governance has higher priority than service-level and global config. This applies to timeout, retry, rate limit, breaker, fallback, and related controls.

Retries should count only real retry attempts, not the final failed attempt.

## Data, Cache, Events, And DTM

Use `roze-local-cache` for in-process Moka-backed cache with TTL, capacity eviction, time-to-idle, and hit/miss statistics.

Use `roze-cache` and Redis helpers for distributed cache. Cache-aside loading should use `roze-singleflight` to avoid hot-key miss stampedes.

Prefer Roze cache abstractions, singleflight, outbox, MQ publisher/consumer, and DTM helpers before adding local concurrency, retry, dead-letter, or transaction orchestration.

High-frequency in-memory lookup/index paths use concurrent maps/sets where the framework already chose them, including metrics, singleflight, registry, session, WebSocket, eventbus, and MQ stores.

Reliable event publishing should prefer outbox relay and publish through a `roze_mq::Publisher`.

MQ consumers must make ack/nack, retry, dead letter, and idempotency behavior explicit.

DTM defaults to TCC. Saga is optional and should not weaken the default TCC state machine.

## Production And Smoke Verification

Run targeted crate tests for local changes. Useful slices:

```bash
cargo test -p rozectl -- --skip postgres --skip mysql --skip mongo
cargo test -p rozectl -- --ignored --skip postgres --skip mysql --skip mongo
cargo test -p roze-rpc
cargo test -p roze-config
cargo test -p roze-mq
cargo test -p roze-gateway
```

Project-level smoke scripts:

```bash
bash scripts/production-smoke.sh
bash scripts/production-smoke.sh --with-compose
bash scripts/rozectl-smoke.sh
```

`--with-compose` starts integration dependencies such as etcd, consul, Kafka, NATS, Redis, Postgres, MySQL, MongoDB, Elasticsearch, OpenSearch, and Meilisearch.

For external dependency verification:

```bash
scripts/roze-project-external-smoke.sh
```

Production-grade adapters should cover real integration behavior, reconnects, retry, timeout, cancellation, metrics labels, trace/context propagation, and localized error responses.

## Documentation Sync

Update docs when changing:

- public `rozectl` commands or flags
- generated layouts
- runtime contracts
- config fields
- SDK/OpenAPI/deployment behavior
- production checklists or smoke behavior

Capability matrices and docs should describe implemented and tested behavior, not plans.
