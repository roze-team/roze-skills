# rozectl Workflows

Use this reference when a task involves generating, regenerating, diffing, documenting, testing, or deploying Roze services.

## Command Selection

Inspect or maintain the installed CLI before generation work:

```bash
rozectl env
rozectl upgrade --dry-run
rozectl completion powershell
```

Use `cargo install --git https://github.com/roze-team/roze.git rozectl --force` or `cargo install --path apps/rozectl --force` when the active checkout requires a newer binary. Roze 1.x is the stable public channel for CLI commands and generated contracts; for production adoption, prefer a pinned Git revision or signed tag and still review the maturity/evidence state for the runtime areas you depend on.

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
cargo run -p rozectl -- stream gen example/events.api --out apps/events-worker --broker rdkafka --roze-source path
```

Regenerate framework-owned files while preserving application-owned files:

```bash
cargo run -p rozectl -- api generate example/user.api --out apps/roze-example --update --roze-source path
```

Use `--update` for normal regeneration. It preserves:

- REST `src/logic/<group>/<method>.rs`
- REST `src/config/mod.rs`
- REST `src/handler/<group>/<method>.rs`
- REST/RPC `src/svc/mod.rs`
- REST/RPC `src/application.rs`
- REST custom middleware files under `src/middleware/<name>.rs`
- RPC `src/config/mod.rs`
- RPC `src/logic/<method>.rs`
- `config.yaml`
- model extension files `src/model/*_ext.rs`

Generated glue such as route registration, handler indexes, DTOs, OpenAPI, RPC server/client adapters, protobuf include modules, `build.rs`, and `proto/service.proto` is refreshed. RPC `--update` removes stale generated logic module declarations for deleted RPC methods while preserving custom declarations in `src/logic/mod.rs`. RPC `--update` also preserves existing generated model composition, including `mod model;`, `ServiceContext::model()`, and Toasty registry wiring when the service already has generated models.

Use `--force` only for a deliberate full rebuild.

API, RPC, stream, model, and search generation is transactional at the project-directory boundary. `rozectl` renders into a same-volume staging project, synchronizes managed service dependencies, formats and validates there, and replaces the target only after every step succeeds. Stream generation runs rustfmt over framework-owned Rust files in create, `--update`, and `--force` modes before committing the staged project; update mode does not rewrite the application-owned consumer or config. Parse, extension, dependency-resolution, or formatting failures should leave the existing project unchanged.

## Service Dependencies

Generated API and RPC services should manage cross-service RPC dependencies through `roze-service.yaml` instead of hand-editing generated client fields or accessors. Add or import a dependency with:

```bash
rozectl service dependency add order --project services/payment --crate shop-order-rpc --path ../shop-order-rpc --endpoint 127.0.0.1:4002
rozectl service dependency list --project services/payment
rozectl service dependency remove order --project services/payment
rozectl service sync --project services/payment --check
```

`dependency add` validates the upstream RPC crate/contract, writes the manifest, and synchronizes the `*-rpc` Cargo path dependency, non-secret defaults in `config/roze-dependencies.yaml`, managed client startup/readiness/accessor sections in `src/svc/mod.rs`, and the generated project kind (`api` or `rpc`). The first dependency add must canonicalize adopted local path dependencies immediately, including defaults and ordering, so an immediate `service sync --check` passes. `service sync --check` writes nothing and fails on drift; run it in release pipelines for every service with `roze-service.yaml`.

`config/roze-dependencies.yaml` is loaded before `config.yaml`; deployment config and `ROZE__...` environment variables override generated dependency defaults. Do not put secrets, tokens, passwords, or certificates in `roze-service.yaml`.

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
rozectl contract diff --old example/user.v1.api --new example/user.v2.api --out contract-diff.md
```

Treat removed routes, removed RPC methods, changed request/response types, removed fields, field source/type changes, and newly required fields as breaking unless the command documents otherwise. Use `contract diff` when reviewers need one semantic report across REST, RPC, OpenAPI, and TypeScript SDK surface changes; it writes the report before failing so CI can retain the artifact.

Run the release-facing semantic gate across API/OpenAPI, search, and SQL schema contracts:

```bash
rozectl gate check --manifest roze-gate.yaml --report target/roze-gate.json --markdown target/roze-gate.md
```

Gate manifests use version `1`, declare `domain: api|search|sql`, and can require checked-in acknowledgement records bound to exact SHA-256 digests for blocking changes. Exit code `0` passes, `1` means an unacknowledged blocking change, and `2` means invalid manifest, acknowledgement, or input.

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

Generated contract tests cover API routes and framework-owned endpoints such as `/healthz`, `/readyz`, `/startupz`, `/metrics`, `/openapi.json`, `POST /reports/exports`, `GET/DELETE /reports/exports/:id`, and `POST /charts/query`. Use `ROZE_E2E_SERVICES=name=http://host:port,...` to run the generated readiness flow against several services from one test command.

For native WebSocket routes, add `@websocket` to a GET route in `.api`. `rozectl api generate --update` refreshes route/handler upgrade glue while preserving application-owned frame logic in `src/logic/**`; WebSocket routes are excluded from OpenAPI and normal HTTP SDK output and cannot use idempotency middleware.

Generate OpenAPI:

```bash
rozectl openapi generate example/user.api --out openapi.json
```

Generate clients:

```bash
rozectl api client ts example/user.api --out sdk/user.ts
rozectl api client js example/user.api --out sdk/user.js
```

Roze currently scopes SDK generation to TypeScript and JavaScript Web clients. Do not document or invoke Dart, Java, Kotlin, Swift, iOS, or Android SDK generation unless the active checkout reintroduces those targets.

Generated TypeScript interfaces preserve the complete API type graph independent of declaration order: custom object fields, nested custom objects, arrays such as `[]Type`/`Vec<Type>`, `Option<T>` nullability, JSON field renames, and `validate:"optional"` / `validate:"omitempty"` optional properties should remain typed instead of collapsing to `unknown`. JavaScript clients should carry the same request-building behavior with JSDoc typedefs.

Generate Markdown API docs:

```bash
rozectl api doc --api example/user.api --dir . --out docs/api
rozectl doc service --api example/user.api --out SERVICE.md
```

Run a custom API plugin after parsing the `.api` once:

```bash
rozectl api plugin --plugin ./tools/api-plugin.sh --api example/user.api --dir generated
```

Plugins receive normalized API JSON through stdin and `ROZECTL_API_SPEC_JSON`; `ROZECTL_API_FILE` and `ROZECTL_OUT_DIR` identify the input and output directories. Plugin outputs are plugin-owned files under the requested directory.

For Rust-side generator extensions, `rozectl` is also a library. Use the versioned extension APIs (`GENERATOR_EXTENSION_API_VERSION` and `MODEL_GENERATOR_EXTENSION_API_VERSION`, currently `1`) instead of depending on private formatting helpers. Command extensions register on `GeneratorRegistry` and run before/after built-in generation; model extensions transform a `ModelGenerationGraph` and may emit generated or application-owned model extension files. Extension outputs must stay in their own unmarked files or documented application-owned extension points; absolute paths, parent traversal, and replacing core model artifacts are rejected.

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

Keep generated deployment files aligned with public docs and smoke scripts whenever command flags or defaults change. Generated Kubernetes/Helm assets expose immutable image digests, Secret/ConfigMap references, Prometheus scrape settings, TLS secret mounts, private registry `image.pullSecrets`, Pod security context defaults, and offline validation. Use `rozectl helm validate` where available to catch malformed values before Helm rendering.

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

Generated production reference systems live under `example/production-systems/` in Roze. Use them when generator changes touch REST/SQL/search, managed REST-to-RPC dependencies, stream workers, repeated updates, dependency synchronization, or generated ops assets:

```bash
bash scripts/generated-reference-systems.sh
bash scripts/reference-systems-integration.sh
bash scripts/production-soak-generated-systems.sh
```

The generated reference-system matrix is compile/regeneration evidence. The integration and soak paths need Linux plus Docker-backed dependencies and are still not a substitute for a completed long-run promoted evidence report.

Before a release, run `bash scripts/release-gate.sh` on Linux or WSL. On Windows, `powershell -ExecutionPolicy Bypass -File scripts/release-preflight.ps1` is useful but non-authoritative because it omits Unix-only rdkafka-enabled targets. Runtime-critical maturity changes also require `bash scripts/production-evidence-gate.sh` and `bash scripts/production-release-audit.sh --json-out target/production-release-audit.json`; model parity work should include `bash scripts/model-parity-gate.sh` when the active checkout provides it.
