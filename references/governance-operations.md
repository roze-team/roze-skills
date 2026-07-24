# Governance And Operations

Use this reference for production behavior across config, middleware, registry, gateway, MQ, cache, DTM, metrics, tracing, and smoke verification.

## Configuration

Applications should use `roze-config::ServiceConfig`. Local files default to `config.yaml`; environment variables and config center values are overrides or hot-update sources.

Prefer Roze config loading, environment override, config center, typed service config, and hot-reload hooks before introducing application-local config systems.

Use built-in secret references for sensitive configuration: `env://NAME`, `${NAME}`, and `file://path`. Relative secret files resolve beside the primary config file, trailing line endings are removed, and `load_with_secret_provider` supports custom providers. JWT key IDs must be unique, resolved HMAC secrets must be at least 32 bytes, and startup fails before listeners are created when secret resolution or validation fails. Debug output must identify references or key IDs only, never secret material.

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
- retry budget

If request id or trace id is missing, the entry point should generate and return it where the protocol supports headers/metadata.

HTTP errors use `RozeError` and `roze-result::ApiResponse`. RPC errors use `roze_rpc::rpc::status_from_error`.

Prefer Roze context, error, result, tracing, and metrics helpers at every protocol boundary. Do not invent parallel request-id, trace-id, locale, tenant, response-envelope, or error-code conventions inside one service.

Context is request metadata, not a global resource container. Keep DB/Redis/RPC clients in `ServiceContext`; keep business user details in explicit request extensions or logic parameters. Do not automatically propagate tokens, cookies, authorization headers, arbitrary user input, or high-cardinality values into context metadata or metrics labels.

Metrics conventions:

- HTTP route metrics: `roze_http_route_*`
- RPC method metrics: `roze_rpc_method_*`
- gateway metrics: `roze_gateway_*`
- MQ metrics include topic, group, partition, offset, attempt, and outcome labels where relevant

Logs for request-scoped work should include request id or trace id through spans. Governance events such as hot reload, retry, breaker, rate limit, dead letter, and fallback should use structured fields.

## Registry, Gateway, And Route Governance

Service discovery should go through `roze_rpc::registry`, including memory, DNS, etcd, consul, and cached resolver modes.

Etcd registry and RPC-client discovery support TLS, mTLS, authentication, token refresh, CA files, client cert/key pairs, and endpoint failover through `RegistryConfig` / `RpcClientEtcdConfig`. Configure cert/key fields together, keep `insecure_skip_verify` false outside controlled diagnostics, and keep credentials in secret references rather than `roze-service.yaml`.

Prefer Roze registry, gateway, retry, timeout, breaker, rate-limit, fallback, and outlier detection surfaces before adding local service-discovery or traffic-governance code.

Gateway upstreams can come from static upstream config or registry discovery. Dynamic discovery should filter by instance tags, health, and outlier state before weighted selection.

Route-level governance has higher priority than service-level and global config. This applies to timeout, retry, rate limit, breaker, fallback, and related controls.

Retries should count only real retry attempts, not the final failed attempt.

`roze_config::GovernanceConfig` is the source of truth for shared policy. At protocol boundaries, resolve one authoritative `GovernancePolicy` with `resolve_policy(key)` or `resolve_policy_for(keys)` instead of merging global and scoped fields independently. Scoped fields override only when present, omitted fields inherit global values, and disabled fallback entries do not silently inherit as enabled.

REST applies timeout, rate limit, breaker, shedding, and fallback. RPC applies timeout, retry budget, rate limit, breaker, shedding, and fallback. MQ `spawn_consumer_with_governance` and job helpers such as `add_governed`, `spawn_with_governance`, and `spawn_once_with_governance` apply timeout, bounded full-jitter retry, retry budget, rate limit, breaker, and shedding before ack/nack or job completion settlement. Fallback is response-oriented; MQ and Job must not reinterpret fallback as ack, success, or suppressed failure.

All governed boundaries should emit `roze_resilience_decisions_total` with bounded `boundary` values such as `rest`, `rpc`, `gateway`, `mq`, or `job`. Keep policy state local to the data path so control-plane availability is not required for each request, message, or job execution.

## Gateway Native HTTP Governance

The native HTTP gateway compiles route policy into an immutable runtime snapshot. Precedence is explicit route fields, then `governance.routes`, then global governance, then gateway defaults. Ordinary HTTP requests enforce method constraints, request-size limits, rate limits, shared breaker state, bounded adaptive shedding, timeout, fallback, and gateway metrics.

Gateway retries are limited to idempotent HTTP methods: `GET`, `HEAD`, `PUT`, `DELETE`, `OPTIONS`, and `TRACE`. Network failures and 500/502/503/504 responses use Roze's exponential full-jitter backoff, `max_backoff_ms`, route/service retry budgets, and the inbound deadline. Do not schedule a retry if its backoff would consume the remaining deadline. Count `attempt` only for calls that actually reach an upstream, and settle the breaker once for the final logical result.

Registry-backed gateway routes should rediscover before every upstream attempt. Merge service and route instance tags with route tags taking precedence; tag mismatches fail closed. Weighted selection uses bounded cursor selection rather than expanding candidates by weight. Connection errors, timeouts, and 5xx responses feed passive outlier ejection, so retries pick again from the remaining healthy set. Active probes use separate healthy/unhealthy thresholds and should end when the gateway runtime is dropped.

Native CORS runs before route matching and before the governance chain. Preflight checks validate origin, method, and headers without consuming rate-limit, breaker, shedding, or upstream capacity; accepted and rejected preflights still produce metrics. Ordinary responses should emit allow-origin only for configured origins.

SSE and WebSocket forwarding must stream without buffering full upstream bodies. The route timeout covers request and response headers; after an SSE or WebSocket stream is established, use `stream_idle_timeout_ms` and `max_stream_connections` with route, service, then gateway precedence. WebSocket upgrades still pass through auth, rate limit, breaker, shedding, registry/tag/health/outlier decisions, validate RFC 6455 version/key and `Sec-WebSocket-Accept`, and reject unrequested subprotocols. `wss`/`https` upstreams use strict TLS validation; plaintext fallback is forbidden.

Config-center reloads build a complete new runtime before atomic replacement. New requests use the new snapshot, in-flight HTTP/SSE/WebSocket work keeps its old snapshot, and rate limits, breaker state, retry budgets, outlier/health state, stream capacity, registry cursors, HTTP client, and TLS state survive replacement. Changes outside `gateway`, `auth`, `governance`, or `registry` should be skipped. Parse errors, invalid policy, registry construction failures, gateway-section removal, and listen-address changes retain the last valid runtime and record structured reload outcomes.

## Data, Cache, Events, And DTM

Use `roze-local-cache` for in-process Moka-backed cache with TTL, capacity eviction, time-to-idle, and hit/miss statistics.

Use `roze-cache` and Redis helpers for distributed cache. Cache-aside loading should use `roze-singleflight` to avoid hot-key miss stampedes.

Prefer Roze cache abstractions, singleflight, outbox, object storage, MQ publisher/consumer, DTM helpers, and `roze-query` read-model composition before adding local concurrency, retry, dead-letter, transaction orchestration, media URL, or fan-out query code.

High-frequency in-memory lookup/index paths use concurrent maps/sets where the framework already chose them, including metrics, singleflight, registry, session, WebSocket, eventbus, and MQ stores.

Reliable event publishing should prefer outbox relay and publish through a `roze_mq::Publisher`.

Generated REST/RPC `ServiceContext` values can expose `Arc<dyn roze_transaction::OutboxStore>`. The default in-memory outbox is for local development and tests. `roze-transaction-sql` is the official PostgreSQL/MySQL `OutboxStore` and `TransactionalOutbox<sea_orm::DatabaseTransaction>` adapter; generated `outbox.store: auto` selects SQL when `database.url` is configured, and production rejects an enabled memory store. Use bundled migrations or an approved migration flow, then rely on `relay_outbox_batch` for claim, lease recovery, retry, dead-letter, replay, cleanup, and `roze_outbox_events_total` metrics.

Generated object-storage integration should keep application logic working with object keys and `FileMetadata`. Use `ServiceContext::media_url` / `resolve_media_url` instead of constructing provider URLs by hand; `issue_upload_token` should return normalized keys, expiration, upload policy, and provider presigned requests.

`roze-storage` supports local storage and an S3-compatible runtime adapter with AWS Signature V4, path-style endpoints, `put_object`, `get_object`, `delete_object`, `stat_object`, and bounded signed PUT/GET URLs. Tenant prefixes, upload validation, metadata, ETags, and endpoint ports in the canonical `Host` header are framework-owned behavior. Qiniu Kodo, Aliyun OSS, and Tencent COS remain provider-SDK boundaries; their mutation methods should fail closed until provider-specific signing exists. Real MinIO/S3 evidence requires the ignored round-trip test with `ROZE_TEST_S3_ENDPOINT`.

Generated model repositories use versioned cache keys via `roze_cache::model_cache_key`, and create/update/delete/soft-delete paths should invalidate every configured lookup key after a successful database write. Use `InvalidationPlan` for writes outside generated repositories. For bounded-staleness read models, use `get_or_load_consistent_option` with `CacheConsistencyPolicy` so stale-on-error cannot outlive the hard TTL window.

Use `roze-query::QueryComposer` for read-model fan-out. Configure total request budget, per-upstream timeout, max fan-out, concurrency, and strict vs partial failure behavior instead of hand-rolling joins across downstream calls.

MQ consumers must make ack/nack, retry, dead letter, and idempotency behavior explicit. Kafka should be constructed through `roze_kafka::build_runtime` or generated stream workers so business code depends only on `roze_mq::{Publisher, Subscriber}`. Use provider `rdkafka` for production consumers; `rdkafka-cmake` is the Windows-friendly production build path, `memory` is for development/testing, and `rust-native`/rskafka is experimental publish-only because it lacks consumer groups and offset commit semantics.

DTM defaults to TCC. Saga is optional and should not weaken the default TCC state machine.

## Production And Smoke Verification

Roze 1.x provides stable public APIs, CLI commands, generated layouts, configuration schemas, metrics/events, and documented runtime ordering under Semantic Versioning. Operational evidence is independent: stable API does not mean every workload, dependency topology, or failure mode is battle-tested. Pin the Roze Git revision or signed tag, inspect generated diffs, run smoke/release gates, and only make long-run production claims when the maturity matrix and production evidence link passing artifacts.

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
cargo test -p roze-storage
```

Project-level smoke scripts:

```bash
bash scripts/production-smoke.sh
bash scripts/production-smoke.sh --with-compose
bash scripts/rozectl-smoke.sh
bash scripts/release-gate.sh
bash scripts/production-evidence-smoke.sh
bash scripts/production-evidence-promotion-smoke.sh
bash scripts/production-release-audit.sh --json-out target/production-release-audit.json
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
bash scripts/production-soak-gateway.sh
bash scripts/production-soak-mq.sh 300
bash scripts/production-soak-config-center.sh 300
bash scripts/production-soak-lifecycle.sh 300
```

For generated service matrices, use:

```bash
bash scripts/generated-reference-systems.sh
bash scripts/reference-systems-preflight.sh
bash scripts/reference-systems-integration.sh
bash scripts/production-soak-generated-systems.sh
```

These exercise the authoritative `example/production-systems/` inputs for REST/SQL/search, managed REST-to-RPC dependencies, stream workers, repeated regeneration, dependency disconnect/recovery, generated operations assets, and Docker-backed registry/storage/broker/search/database dependencies. Set `ROZE_REFERENCE_EVIDENCE_DIR` when retaining machine-readable integration bundles. Direct soak runs should set `ROZE_DIRECT_SOAK_EXPECTED_REVISION` to the release commit so the runner fails closed when `HEAD` differs. Treat compile, preflight, direct dependency probes, or short integration success as generator/runtime coverage, not as long-run production proof.

Competitive benchmark assets under `benchmarks/competitive/` compare generated Roze and go-zero services. They are machine-readable benchmark contracts, not evidence by themselves. Validate structure with `node scripts/competitive-baseline-verify.js` and `node scripts/competitive-input-verify.js`; source builds and echo smoke prove generation/process wiring only. A passing competitive claim requires the fixed runner, digest-bound dependencies, adjacent counterbalanced pair samples, sample/report verifiers, and `scripts/competitive-report-verify.js`; producer-provided pass fields or scores are not trusted.

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

Passing reports must come from fixed-runner artifacts, not from manual report edits. `scripts/production-soak-preflight.sh` validates the runner before long workloads. `scripts/production-evidence-promote.sh` promotes a downloaded artifact only after checking terminal run metadata, elapsed duration, resource samples, boundary summaries, portable SHA-256 manifests, artifact digest, and GitHub provenance. `scripts/production-evidence-report-verify.sh` is the independent predicate used by the maturity evidence gate, including area-specific boundary schemas, counter invariants, percentile ordering, fault counts, sampler coverage, and memory-decline bounds.

`scripts/production-evidence-gate.sh` prevents runtime-critical maturity entries from moving to `stable` without complete passing 24h/72h evidence. Supply-chain gates should run RustSec advisory, dependency license, and registry/source policy checks against `Cargo.lock`, `audit.toml`, and `deny.toml`; exceptions must be narrow, owned, dated, and removable.

`scripts/production-release-audit.sh` is the S6 release predicate over the exact Git revision and maturity matrix. Candidate mode can emit `api_stable_long_run_pending` while Gateway, MQ, Config Center, Lifecycle, or generated services still lack promoted reports. Add `--require-long-run` only for publications that claim battle-tested runtime behavior; any long-run verified report must carry the same full audited revision.

## Documentation Sync

Update docs when changing:

- public `rozectl` commands or flags
- generated layouts
- runtime contracts
- config fields
- SDK/OpenAPI/deployment behavior
- production checklists or smoke behavior

Capability matrices and docs should describe implemented and tested behavior, not plans.
