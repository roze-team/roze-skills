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

Roze is currently pre-release. It is suitable for evaluation, internal pilots, and controlled production paths where teams pin a reviewed Git revision, inspect generated diffs, run smoke checks, and own production configuration, observability, rollback, auth, and dependency governance. Do not describe a module as production-stable unless the active checkout's maturity matrix and production evidence support that claim.

Run targeted crate tests for local changes. Useful slices:

```bash
cargo test -p rozectl -- --skip postgres --skip mysql --skip mongo
cargo test -p rozectl -- --ignored --skip postgres --skip mysql --skip mongo
cargo test -p roze-rpc
cargo test -p roze-config
cargo test -p roze-mq
cargo test -p roze-gateway
cargo test -p roze-service -p roze-bootstrap -p roze-shutdown
cargo test -p roze-job
```

Project-level smoke scripts:

```bash
bash scripts/production-smoke.sh
bash scripts/production-smoke.sh --with-compose
bash scripts/rozectl-smoke.sh
bash scripts/release-gate.sh
bash scripts/production-evidence-smoke.sh
```

`--with-compose` starts integration dependencies such as etcd, consul, Kafka, NATS, Redis, Postgres, MySQL, MongoDB, Elasticsearch, OpenSearch, and Meilisearch.

`scripts/release-gate.sh` is the high-signal local gate for formatting, clippy, Gateway, Config Center, MQ, lifecycle/bootstrap, `roze-job`, `rozectl` smoke, generated compile smoke, production evidence smoke, and production smoke without external Compose dependencies.

For external dependency verification:

```bash
scripts/roze-project-external-smoke.sh
```

Production-grade adapters should cover real integration behavior, reconnects, retry, timeout, cancellation, metrics labels, trace/context propagation, and localized error responses.

## Production Evidence And Soak

Release gates and smoke scripts are not long-run production evidence. Runtime-critical modules need reproducible reports before stable claims.

Use the built-in report scaffold instead of inventing ad hoc report formats:

```bash
bash scripts/production-evidence.sh \
  --area lifecycle \
  --duration 24h \
  --workload "start, drain, shutdown, failed task, timeout hooks" \
  --failure-injection "stuck task, signal shutdown, hook timeout" \
  --command "ROZE_LIFECYCLE_SOAK_SECONDS=86400 bash scripts/production-soak-lifecycle.sh"
```

Short soak entrypoints validate the harness. Increase duration and workload before using them as evidence:

```bash
bash scripts/production-soak-mq.sh 300
bash scripts/production-soak-config-center.sh 300
bash scripts/production-soak-lifecycle.sh 300
```

Evidence reports should record the Roze Git revision, Rust toolchain, OS, command, dependency topology, workload, duration, success criteria, error budget, latency/throughput/resource trends, restart count, leak checks, failure injection timeline, and final verdict.

Keep report verdicts conservative: use `inconclusive` until measurements and artifacts are filled in. Do not fabricate 24h/72h results or call beta/scaffold modules stable based only on short-run smoke checks.

Lifecycle evidence can include a validated summary line:

```bash
bash scripts/production-evidence.sh \
  --area lifecycle \
  --duration 24h \
  --workload "start, drain, shutdown, failed task, timeout hooks" \
  --failure-injection "stuck task, signal shutdown, hook timeout" \
  --command "bash scripts/production-soak-lifecycle.sh" \
  --lifecycle-summary "roze_lifecycle_soak cycles=2 worker_exits=8 stop_hooks=8 running_snapshots=2 stopped_snapshots=2 max_service_count=4"
```

`scripts/production-evidence-smoke.sh` verifies report scaffold generation and rejects lifecycle summaries with missing fields, non-numeric fields, non-lifecycle usage, or inconsistent counts.

## Documentation Sync

Update docs when changing:

- public `rozectl` commands or flags
- generated layouts
- runtime contracts
- config fields
- SDK/OpenAPI/deployment behavior
- production checklists or smoke behavior

Capability matrices and docs should describe implemented and tested behavior, not plans.
