# Rust Engineering Patterns

Use this reference for Rust language and ecosystem decisions inside Roze crates, `rozectl` generators, generated REST/RPC services, and application-owned Roze code.

This guidance is adapted for Roze from useful patterns in `actionbook/rust-skills`. Keep Roze behavior first: if Roze has a generated extension point or `roze-*` helper for the problem, use that before adding generic Rust infrastructure.

## Ownership And Lifetimes

Before fixing borrow-checker errors, ask who should own the data and how long it should live.

Common decision points:

- Use owned values for domain entities, configuration snapshots, generated DTOs crossing async boundaries, and values stored in `ServiceContext`.
- Use `&T` or `&str` for read-only views inside one call.
- Use `&mut T` for local, exclusive mutation.
- Use `Arc<T>` for shared application state across async tasks, Axum handlers, tonic services, registry clients, caches, pools, and background workers.
- Use `Cow<'_, str>` only when borrowed data may need mutation or normalization.
- Clone value objects when duplication is domain-correct. Do not add `.clone()` only to silence ownership errors.

Error-oriented checks:

- `E0382` moved value: decide whether ownership should transfer, be borrowed, or be shared.
- `E0597` borrowed value does not live long enough: check whether a reference is crossing request/task boundaries where owned or `Arc` data is needed.
- `E0506` or `E0507`: check whether mutation should happen in a narrower scope or via a different data shape.
- `E0716`: bind temporaries when references need a named owner.

Roze-specific notes:

- Generated DTOs can usually be owned. Avoid overfitting lifetimes into generated types unless the active generator already supports that style.
- `ServiceContext` should own long-lived clients, pools, registries, caches, and config snapshots.
- Do not store request-scoped borrows in application-wide state.

## Error Handling

First ask whether the failure is expected, absence, invariant violation, or unrecoverable.

Use:

- `Result<T, E>` for recoverable failures.
- `Option<T>` when absence is normal.
- `?` to propagate errors.
- `expect("specific invariant")` only when the invariant is truly guaranteed.
- `panic!`, `assert!`, and `unreachable!` for programmer bugs or impossible states, not operational failures.

Roze-specific rules:

- Prefer Roze error/result helpers at protocol boundaries: `RozeError`, `roze-result::ApiResponse`, and `roze_rpc::rpc::status_from_error`.
- Preserve request id, trace id, locale, tenant, and error code metadata.
- Categorize errors by recovery: validation/user errors, not found, auth/permission, transient upstream, timeout, rate limited, config invalid, and internal.
- Retry only transient errors. Use Roze governance/retry/breaker surfaces before local retry loops.
- Add context to internal errors before converting them to user-facing HTTP/gRPC shapes.

Avoid:

- `unwrap()` in production logic.
- String-only errors where callers need structured handling.
- Exposing database, parser, or upstream internals directly to clients.
- Swallowing errors without logging, metrics, or documented fallback behavior.

## Async And Concurrency

Ask whether the work is I/O-bound, CPU-bound, or shared-state coordination.

Use:

- `async`/`await` and Tokio tasks for I/O-bound service work.
- `spawn_blocking` or dedicated workers for CPU-heavy or blocking operations.
- `Arc<T>` for immutable shared state.
- `Arc<Mutex<T>>` or `Arc<RwLock<T>>` only when shared mutation is required.
- atomics for simple counters and flags.
- channels for ownership transfer and background worker coordination.

Roze-specific rules:

- Generated REST handlers and RPC servers run in async contexts; do not block the executor with filesystem, CPU-heavy, sleep, or synchronous network calls.
- Do not hold `MutexGuard`, `RwLockGuard`, transaction handles, or other scoped guards across `.await` unless the type and design explicitly support it.
- Use Roze cache, MQ, singleflight, registry, and gateway primitives before inventing local concurrency systems.
- Shared state inside `ServiceContext` must be `Send + Sync` when used by Axum/Tower/tonic services.

Common pitfalls:

- `Rc` or `RefCell` in handler/server state: use `Arc` and thread-safe primitives.
- `thread::sleep` in async code: use `tokio::time::sleep`.
- `future is not Send`: ensure non-`Send` values are dropped before `.await`, use `Arc`, or keep the work local only when the runtime design allows it.
- deadlocks: keep lock ordering explicit and scopes short.

## Resource Lifecycle

Ask when a resource is created, who owns cleanup, and what happens on error.

Use:

- RAII and `Drop` for automatic cleanup.
- `OnceLock` or `LazyLock` for process-wide lazy initialization when a Roze config/context-owned resource is not better.
- pools for expensive reusable resources such as database or network clients.
- scoped guards for locks, transactions, temp files, and leases.

Roze-specific rules:

- Prefer creating clients, pools, registries, health checks, caches, and publishers at service startup and storing them in `ServiceContext`.
- Register dependency readiness with `roze_health::HealthRegistry` when applicable.
- Let generated startup/config code define lifecycle boundaries where possible.
- Do not create expensive clients per request unless the active Roze pattern requires it.

## Dependencies And Cargo

Before adding a dependency, check whether Roze or the standard library already provides the capability.

Selection criteria:

- actively maintained
- clear license
- good docs and examples
- feature flags that avoid unused bloat
- compatible with the workspace MSRV and async runtime
- no duplicate major versions of core dependencies

Roze-specific rules:

- Prefer existing workspace crates and `roze-*` abstractions over adding external crates.
- Keep dependency additions local to the crate that needs them.
- Use workspace dependency versions when the Roze checkout defines them.
- Avoid wildcard versions.
- Be careful with default features on network, TLS, database, metrics, and async crates.

Useful checks:

```bash
cargo tree -p <crate>
cargo tree -d
cargo check -p <crate>
cargo test -p <crate>
cargo clippy -p <crate> --all-targets -- -D warnings
```

## Performance

Measure before optimizing. Prefer clear Roze-compatible code until profiling shows a bottleneck.

Priority order:

1. Algorithm and query shape.
2. Data structure choice.
3. Allocation reduction.
4. Cache/locality.
5. Parallelism or SIMD.

Useful patterns:

- Use `Vec::with_capacity` or `String::with_capacity` when size is known.
- Avoid cloning large DTOs, config, request bodies, and search/model results on hot paths.
- Batch I/O and database/search operations where Roze repositories or clients support it.
- Prefer sequential `Vec` access over hash maps for small collections.
- Use Roze cache and singleflight helpers for hot-key paths.

Avoid:

- optimizing without a benchmark or profile
- benchmarking debug builds
- using `HashMap` or heap allocation for tiny fixed-size data by habit
- adding `unsafe` for performance before measuring safe alternatives

## Unsafe And FFI

Default to safe Rust. Treat `unsafe` as a reviewed boundary, not an escape hatch for borrow or lifetime issues.

Only consider `unsafe` for:

- FFI
- low-level abstractions with clear invariants
- measured hot paths where safe alternatives are insufficient

Required for unsafe code:

- Every `unsafe` block has a `// SAFETY:` comment explaining the invariants.
- Every public `unsafe fn` has a `# Safety` docs section.
- Raw pointers are valid, aligned, initialized, and obey aliasing rules before dereference.
- FFI signatures, ABI, ownership, string encoding, allocation/free ownership, panic boundaries, and thread-safety are documented.
- `impl Send` or `impl Sync` is justified by actual synchronization and invariants.

Prefer:

- `MaybeUninit` over uninitialized or invalid zeroed values.
- `NonNull<T>` for non-null raw pointer wrappers.
- `Atomic*`, `Mutex`, or `RwLock` over `static mut`.
- `bindgen`/`cbindgen` for FFI bindings when appropriate.

## Refactoring And Review

For Rust refactors, analyze impact before editing:

- Find definitions and references with `rg` and LSP when available.
- Check public re-exports and generated ownership boundaries.
- Check macro-generated code and docs references.
- Preserve generated Roze files by editing generators/templates instead of generated glue.
- Keep changes small enough for targeted tests.

Review checklist:

- No production `unwrap()` without a strong invariant.
- No `.clone()` without data-flow justification.
- No `String` where `&str` or `Cow<'_, str>` is the correct API.
- No `Rc`/`RefCell` in async service state.
- No blocking calls in handlers or RPC methods.
- No locks held across `.await`.
- No ignored `Result` or `#[must_use]` values.
- No public fields that allow invalid state when constructors or methods should enforce invariants.
- No `unsafe` without safety docs/comments.
- No custom context/error/metrics/config patterns that bypass Roze conventions.

## Anti-Patterns To Watch

- Fighting the borrow checker with clones, leaks, `'static`, or global state.
- Using `anyhow` at public library boundaries where callers need typed errors.
- Retrying all failures instead of only transient ones.
- Creating per-request clients, pools, or registries.
- Local middleware that duplicates Roze built-ins.
- Handwritten response envelopes outside Roze result helpers.
- Handwritten service-discovery, rate-limit, breaker, cache, or MQ logic when Roze provides it.
- Premature generic abstractions in generated services.
- Over-broad `pub` visibility in app modules.
