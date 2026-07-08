# rozectl Workflows

Use this reference when a task involves generating, regenerating, diffing, documenting, testing, or deploying Roze services.

## Command Selection

Inspect or maintain the installed CLI before generation work:

```bash
rozectl env
rozectl upgrade --dry-run
rozectl completion powershell
```

Use `cargo install --git https://github.com/roze-team/roze.git rozectl --force` or `cargo install --path apps/rozectl --force` when the active checkout requires a newer binary. Roze is pre-release; prefer a pinned Git revision for controlled production paths.

Generate REST from `.api` route declarations:

```bash
cargo run -p rozectl -- api generate example/user.api --out apps/roze-example --roze-source path
```

Generate RPC from `.api` RPC method declarations:

```bash
cargo run -p rozectl -- rpc generate example/user.api --out apps/user-rpc --roze-source path
```

Generate RPC from proto3:

```bash
cargo run -p rozectl -- rpc protoc example/user.proto --out apps/user-rpc --roze-source path
```

Generate a stream worker scaffold from event-capable RPC/API contracts:

```bash
cargo run -p rozectl -- stream gen example/events.api --out apps/events-worker --roze-source path
```

Regenerate framework-owned files while preserving application-owned files:

```bash
cargo run -p rozectl -- api generate example/user.api --out apps/roze-example --update --roze-source path
```

Use `--update` for normal regeneration. It preserves:

- REST `src/logic/**.rs`
- RPC `src/logic/*.rs`
- custom declarations in RPC `src/logic/mod.rs`
- stream `src/stream/consumer.rs`
- REST custom `src/middleware/*.rs`
- `config.yaml`
- model extension files `src/model/*_ext.rs`

Use `--force` only for a deliberate full rebuild.

Create starter projects through the same generator path:

```bash
rozectl quickstart user-api --kind api --out services/user-api
rozectl quickstart user-rpc --kind rpc --out services/user-rpc
rozectl api new user-api --out services/user-api
rozectl rpc new user-rpc --out services/user-rpc
```

For teams migrating from goctl, prefer Roze's compatibility surface instead of custom conversion scripts:

```bash
rozectl migrate api --from go-zero --api user.api --out roze.api
rozectl migrate api user.api --check
rozectl migrate api user.api --write
rozectl api go --api example/user.api --dir apps/roze-example
```

`quickstart`, `api new`, `rpc new`, and goctl-compatible generation can read starter templates from `--home`, `--remote`, and `--branch`.

## Dry Runs And Compatibility

Validate and format contracts before generating or reviewing diffs:

```bash
rozectl api validate example/user.api
rozectl api format example/user.api --check
rozectl api format example/user.api --write
```

`api validate` catches duplicate types/fields/routes/RPC methods, invalid generated Rust identifiers, unknown request/response types, path parameter mismatches, middleware name collisions, and other generation-blocking contract errors.

Preview changes without writing to the target:

```bash
rozectl diff api example/user.api --out apps/roze-example --roze-source path
rozectl diff rpc example/user.api --out services/user-rpc --roze-source path
rozectl diff model example/user.sql --out services/user-api --format sql
```

Check API contract compatibility:

```bash
rozectl contract check --old example/user.v1.api --new example/user.v2.api
```

Treat removed routes, removed RPC methods, changed request/response types, removed fields, field source/type changes, and newly required fields as breaking unless the command documents otherwise.

Inspect and maintain local or remote starter templates through built-in commands:

```bash
rozectl template list
rozectl template show api
rozectl template init --home templates
rozectl template diff api --home templates
rozectl template update api --home templates
rozectl template revert api --home templates
```

## Mock, Tests, OpenAPI, And SDKs

Generate a mock REST server:

```bash
rozectl mock gen --api example/user.api --out mock-server
```

Generate HTTP contract tests:

```bash
rozectl test gen --api example/user.api --out contract-tests
```

Generate OpenAPI:

```bash
rozectl openapi generate example/user.api --out openapi.json
```

Generate clients:

```bash
rozectl api client ts example/user.api --out sdk/user.ts
rozectl api client js example/user.api --out sdk/user.js
rozectl api client dart example/user.api --out sdk/user.dart
rozectl api client java example/user.api --out sdk/RozeApiClient.java
rozectl api client kotlin example/user.api --out sdk/RozeApiClient.kt
rozectl api client swift example/user.api --out sdk/RozeApiClient.swift
```

Generate Markdown API docs:

```bash
rozectl api doc --api example/user.api --dir . --out docs/api
rozectl doc service --api example/user.api --out SERVICE.md
```

## Local Environment Commands

Check local tools, config files, ports, and dependency TCP endpoints:

```bash
rozectl doctor --config apps/roze-example/config.yaml --port 3000
rozectl doctor --tcp 127.0.0.1:6379 --tcp 127.0.0.1:9092
rozectl doctor --tool helm --tool etcdctl
```

Start or inspect the local dependency stack:

```bash
rozectl dev up --detach
rozectl dev status
rozectl dev down
```

`rozectl dev` defaults to `docker-compose.integration.yml` unless a compose file is supplied.

## Deployment Artifacts

Generate Docker and Kubernetes assets:

```bash
rozectl docker --binary user-api --port 8080
rozectl kube deploy --name user-api --image registry.example.com/user-api:latest --port 8080
```

Keep generated deployment files aligned with public docs and smoke scripts whenever command flags or defaults change.

## Generator Smoke Checks

For ordinary parser/generator changes:

```bash
cargo test -p rozectl -- --skip postgres --skip mysql --skip mongo
```

Generated REST, RPC, stream, Toasty, and SeaORM templates also have ignored compile-smoke tests that create temporary crates and run `cargo check`, plus `cargo clippy --all-targets -- -D warnings` where applicable:

```bash
cargo test -p rozectl -- --ignored --skip postgres --skip mysql --skip mongo
```

Run database-specific tests only when local credentials or compose dependencies are available.

For release-sensitive generator work, run the explicit compile-smoke slices when relevant:

```bash
cargo test -p rozectl generated_rest_project_compiles_with_model_and_search -- --ignored
cargo test -p rozectl generated_rpc_project_compiles -- --ignored
cargo test -p rozectl generated_stream_project_compiles -- --ignored
```
