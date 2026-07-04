# rozectl Workflows

Use this reference when a task involves generating, regenerating, diffing, documenting, testing, or deploying Roze services.

## Command Selection

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

Regenerate framework-owned files while preserving application-owned files:

```bash
cargo run -p rozectl -- api generate example/user.api --out apps/roze-example --update --roze-source path
```

Use `--update` for normal regeneration. It preserves:

- REST `src/logic/**.rs`
- RPC `src/logic/*.rs`
- REST custom `src/middleware/*.rs`
- `config.yaml`
- model extension file `src/model/_ext.rs` when model generation supports it

Use `--force` only for a deliberate full rebuild.

## Dry Runs And Compatibility

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
