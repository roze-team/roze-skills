# Troubleshooting

Use this reference when diagnosing Roze generation, runtime, integration, or verification issues.

## Contract And Generation Issues

Symptom: `rozectl api generate` rejects a contract.

Check whether the `.api` file contains RPC method declarations. REST generation expects REST route blocks. Use `rozectl rpc generate` for RPC method declarations.

Symptom: `rozectl rpc generate` rejects a contract.

Check whether the `.api` file contains REST routes. RPC generation expects RPC method declarations. Use `rozectl api generate` for REST routes.

Symptom: generated logic or config changed unexpectedly.

Prefer `--update`, not `--force`. Confirm the file is application-owned:

- REST logic: `src/logic/**.rs`
- RPC logic: `src/logic/*.rs`
- custom REST middleware: `src/middleware/*.rs`
- config: `config.yaml`
- model extension: `src/model/_ext.rs`

If a generator-owned file was edited by hand, move the intended behavior into templates/generator code or into an application-owned extension point.

## Middleware Issues

Symptom: custom middleware file was not generated.

Check whether the middleware name is a built-in alias. Built-ins such as `auth`, `jwt`, `trace`, `recover`, `stat`, `prometheus`, `metrics`, `cors`, `timeout`, `rate_limit`, `breaker`, `max_conns`, `shedding`, `gunzip`, `body_limit`, and `idempotency` should not generate custom files.

Symptom: request timeouts behave differently than expected.

Check `rest.middlewares.timeout` and route governance overrides. `timeout: true` lets generated route glue enforce service-wide `governance.timeout_ms` and route-specific timeout overrides. `timeout: false` can still propagate timeout metadata without cancelling logic in the HTTP adapter.

## Validation Issues

Symptom: validator tags do not map to derives.

Some Go validator tags are generated as request-level checks instead of native Rust derive attributes. Inspect the generated DTO validation implementation and add/adjust generator tests for the tag behavior.

Symptom: cross-field validation is missing.

Cross-field comparisons should be generated only when both fields map to the same Rust type. Check the generated Rust field types after `.api` type mapping.

## Context And Error Issues

Symptom: trace id or request id missing from logs or responses.

Check the entry middleware/adapter. REST/RPC/gateway/MQ entry points should restore or generate Roze context and attach it to spans or response headers/metadata.

Symptom: RPC errors lack metadata.

Ensure server adapters convert errors through `roze_rpc::rpc::status_from_error(err, &request_ctx)`. Standard metadata includes Roze error code, error kind, request id, trace id, and locale.

## Local Dependency Issues

Symptom: service or smoke test cannot connect to Redis, Kafka, NATS, databases, registry, or search engines.

Run:

```bash
rozectl doctor --tcp 127.0.0.1:6379 --tcp 127.0.0.1:9092
rozectl dev status
```

For full integration dependencies, start the compose profile used by the project:

```bash
rozectl dev up --detach
```

or:

```bash
bash scripts/production-smoke.sh --with-compose
```

## Test Selection

For parser/generator changes:

```bash
cargo test -p rozectl -- --skip postgres --skip mysql --skip mongo
```

For Postgres/MySQL inspect changes, run the database-specific tests with configured URLs:

```bash
cargo test -p rozectl postgres
cargo test -p rozectl mysql
```

For runtime crates, run the affected crate tests and any neighboring generator tests that assert generated integration code.

For shell smoke script edits, at least syntax-check the script with Bash and run it when dependencies are available.
