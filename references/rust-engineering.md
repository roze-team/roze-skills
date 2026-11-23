# Rust Engineering Patterns

Use this reference for Rust language and ecosystem decisions inside Roze crates, `rozectl` generators, generated REST/RPC services, and application-owned Roze code.

This is the BeautifulRust-aligned source of truth for general Rust decisions in Roze, synchronized from [`morganlinton/BeautifulRust`](https://github.com/morganlinton/BeautifulRust) baseline `b9718436` and merged with Roze-specific guidance previously adapted from `actionbook/rust-skills`. BeautifulRust governs general Rust style and API quality; active Roze contracts govern generated ownership, framework APIs, wire compatibility, lifecycle, and built-in infrastructure.

## Mindset And Scope

- Write Rust-shaped code: ownership and `Drop` for resources, `Option` for absence, `Result` for recoverable failure, traits/enums for polymorphism, and types for invariants. Do not transliterate inheritance, exception, manager/factory, manual-memory, or global-state patterns.
- Make invalid states unrepresentable where practical. Validate once at construction and expose operations that preserve the invariant.
- Distinguish application code from reusable library/framework code. Application code should be concrete and direct; public libraries require stronger type, documentation, error, SemVer, and future-evolution discipline.
- Let rustfmt own formatting, clippy catch mechanical issues, and the compiler enforce memory safety. Spend review attention on behavior, contracts, soundness, cancellation, and architecture.
- Prefer the standard library and active Roze built-ins before dependencies or local infrastructure. Do not use a general Rust style preference to override `.api`/`.proto`/`.ent` contracts or documented generated extension points.

## Ownership And Lifetimes

Before fixing borrow-checker errors, ask who should own the data and how long it should live.

Common decision points:

- Use owned values for domain entities, configuration snapshots, generated DTOs crossing async boundaries, and values stored in `ServiceContext`.
- Use `&T` or `&str` for read-only views inside one call.
- Use `&mut T` for local, exclusive mutation.
- Use `Arc<T>` for shared application state across async tasks, Roze native HTTP handlers, tonic services, registry clients, caches, pools, and background workers.
- Use `Cow<'_, str>` only when borrowed data may need mutation or normalization.
- Clone value objects when duplication is domain-correct. Do not add `.clone()` only to silence ownership errors.

When a borrow fails, try these in order before shared ownership or cloning:

1. Bind a temporary to extend its lifetime or narrow an existing borrow with a block.
2. Split a long expression into typed intermediate values.
3. Move a field with `mem::take`, `mem::replace`, or `Option::take`.
4. Split a large state object so disjoint data can be borrowed independently.
5. Clone small immutable data deliberately when duplication is semantically correct.
6. Use indices, arenas, or `Rc`/`Arc` plus interior mutability only when sharing is part of the design. Avoid self-referential structs.

Prefer owned data in structs unless borrowing is essential to the abstraction. Descriptive lifetimes such as `'src` or `'conn` help on long-lived types; simple functions can use short lifetimes or elision. Build with local mutation, then expose immutable state after setup.

Error-oriented checks:

- `E0382` moved value: decide whether ownership should transfer, be borrowed, or be shared.
- `E0597` borrowed value does not live long enough: check whether a reference is crossing request/task boundaries where owned or `Arc` data is needed.
- `E0506` or `E0507`: check whether mutation should happen in a narrower scope or via a different data shape.
- `E0716`: bind temporaries when references need a named owner.

Roze-specific notes:

- Generated DTOs can usually be owned. Avoid overfitting lifetimes into generated types unless the active generator already supports that style.
- `ServiceContext` should own long-lived clients, pools, registries, caches, and config snapshots.
- Do not store request-scoped borrows in application-wide state.
- A cheaply cloned client may hide `Arc<Inner>` internally; callers should not need to manage the ownership wrapper unless it is part of the Roze API contract.

## Rust-Shaped Types And APIs

### Naming And Conversions

- Use `UpperCamelCase` for types, traits, and variants; `snake_case` for functions, methods, modules, and locals; `SCREAMING_SNAKE_CASE` for constants/statics. Treat acronyms as words: `HttpClient`, `Uuid`.
- Getters omit `get_`: use `name()`, `name_mut()`, `set_name(...)`, and `take_name()`. Use `get` only when there is one obvious value, as with `Cell::get`.
- Conversion names communicate cost and ownership: `as_` is a cheap borrowed view, `to_` creates an owned value or otherwise performs work, and `into_` consumes `self`. Use `_unchecked` only for unsafe/unchecked siblings with a checked alternative.
- Use `new` for the primary constructor, `try_new` when validation can fail, `with_capacity` or `new_with_*` for clear variants, and domain verbs such as `open` or `connect` for I/O resources.
- Name iterators `Iter`, `IterMut`, and `IntoIter`. If a collection exposes `len()`, also expose `is_empty()`.
- Avoid vague suffixes such as `Manager`, `Factory`, `Helper`, or `Util` when a domain name describes the responsibility. Do not repeat the module name in the type.

Model domain invariants with Rust types instead of comments and conventions:

- Use enums for mutually exclusive modes and public boolean choices, newtypes for IDs or units that must not mix, `Option<T>` for absence, and data-carrying enum variants for state-specific data. Avoid sentinel values such as `-1` or empty strings.
- Use the strongest standard type that fits the domain, such as `Path`/`PathBuf` for paths, `Duration` for elapsed time, and `NonZero*` for internal non-zero invariants. Keep wire DTO types dictated by `.api`/`.proto` contracts and convert at the application boundary.
- Prefer named fields over positional tuples except for single-field newtypes and very local code. Use fallible constructors, `TryFrom`, or `FromStr` when construction validates an invariant.
- Add `#[repr(transparent)]` to a single-field newtype only when layout compatibility is part of the contract.
- Use independent bit flags only when choices can combine; use an enum when exactly one choice is valid.
- Application-owned Roze code should stay concrete and direct. Framework/library APIs need stricter evolution discipline: private fields by default, narrow visibility, stable error types, and `#[non_exhaustive]` for extensible public enums or public-field structs when exhaustive matching is not part of the contract.

Design public APIs around ownership and cost:

- Accept `&str`, `&[T]`, `&Path`, `&T`, or `Option<&T>` rather than `&String`, `&Vec<T>`, `&PathBuf`, `&Box<T>`, or `&Option<T>`. `AsRef`/`IntoIterator` can broaden frequently used library inputs; application code should prefer direct concrete inputs. Take `T` by value when the callee stores or consumes it instead of borrowing and cloning internally.
- Keep `Arc`, `Rc`, `RefCell`, locks, and third-party implementation types out of public signatures unless sharing or interoperability is itself the contract. Roze-generated context and client surfaces are explicit exceptions when the active framework API owns that shape.
- Use `as_` for cheap borrowed views, `to_` for owned conversion, and `into_` for consuming conversion. Implement `From`/`TryFrom` and `FromStr` instead of ad hoc conversion names.
- Derive applicable standard traits deliberately. Public types normally need `Debug`, with secrets redacted; derive `Clone` only when duplication is valid and `Copy` only for small plain values. Do not ship `PartialOrd` without `Ord` when a total order exists. Error types need `Display` and `std::error::Error`; zero-argument `new()` and `Default` should agree. If `Ord`, `Eq`, or `Hash` is handwritten, they must compare the same semantic fields. Add `#[must_use]` when discarding a returned value/type is almost certainly a bug.
- Avoid public boolean parameters. Prefer a named enum or separate methods so call sites remain readable and a third mode can be added safely.
- Use a builder when several optional values or cross-field validation make constructors unclear. Keep simple constructors simple; do not add genericity, traits, or builders speculatively.
- Up to a couple of optional values can use named constructors. For larger or cross-validated configuration, use a builder whose setters name fields and whose `build() -> Result<_, _>` performs all validation. Group four or more loose primitive arguments into semantic types.

### Traits, Generics, And Evolution

- Implement `From`/`TryFrom`, not `Into`/`TryInto` directly. Implement `FromIterator`, `Extend`, and owned/borrowed `IntoIterator` only when the type is genuinely collection-like.
- Implement `Deref` only for smart-pointer or owning-collection behavior, never to simulate inheritance. Overload operators only for their conventional mathematical/domain meaning.
- Put trait bounds on the `impl` or function that needs them rather than on the type definition when derivation and storage do not require the bound.
- Define a trait for generic/dynamic dispatch or an extension trait, not merely to group methods. Keep essential operations as inherent methods and required trait methods minimal; derive default behavior from the minimal core.
- If an operation has a natural receiver, make it a method; use a free function for computations without one. Keep associated functions primarily for construction and type-level operations.
- Prefer concrete types, then generics/`impl Trait`, then `dyn Trait`. Use trait objects for runtime heterogeneity or binary-size concerns, ideally behind a domain wrapper. Avoid deeply nested generic service types.
- Decide object safety intentionally. Put methods unavailable to trait objects behind `where Self: Sized`.
- Seal traits when downstream implementations would prevent safe evolution, and document that decision.
- Use the least restrictive closure bound that matches use: `FnOnce`, then `FnMut`, then `Fn`. Put a closure parameter last and avoid APIs with several unrelated callbacks.
- Keep parameter ordering consistent across related APIs: operation-specific inputs first and ubiquitous context/configuration last, unless an established Roze signature dictates otherwise.
- Reach for a macro only after functions, generics, and traits cannot express the operation. Prefer `macro_rules!` to a proc macro when it is sufficient, keep input recognizable, and do not hide changes to a type's kind or function signature.
- Expose each public item at one canonical path. Avoid leaking dependency types; feature-gate optional integrations and re-export a dependency type only when callers must name the exact compatible version.
- Return iterators from reusable library APIs when eager collection is unnecessary. Use an opaque iterator internally or a named wrapper when a public concrete iterator type would freeze implementation details.
- Features are additive and bare-named (`std`, `serde`, `test-util`). A feature must not remove or change an existing public item, field, or trait method.
- Keep reusable mocks/fakes behind a `test-util` feature when downstream integration tests need them; do not make production `Default` constructors perform I/O.
- Adding enum variants, public struct fields, trait bounds, or required trait methods can be breaking. Use `#[non_exhaustive]`, sealing, defaults, deprecation, and major releases deliberately rather than after the compatibility problem appears.
- Mark an API `const` only when maintaining const evaluation is a compatibility commitment. Do not expose implementation-only error kinds or fields that callers cannot act on.

Roze uses generated `src/logic/prelude.rs` and REST group preludes as intentional extension points. Do not apply generic "no prelude" advice to those files, and do not replace generated DTOs or public protocol enums merely to satisfy a style preference.

## Error Handling

First ask whether the failure is expected, absence, invariant violation, or unrecoverable.

Use:

- `Result<T, E>` for recoverable failures.
- `Option<T>` when absence is normal.
- `?` to propagate errors.
- `expect("specific invariant")` only when the invariant is truly guaranteed.
- `panic!`, `assert!`, and `unreachable!` for programmer bugs or impossible states, not operational failures.

Panics are not exceptions. Do not use `catch_unwind` for ordinary control flow or continue a library after an arbitrary panic. Pair a justified panicking convenience with a fallible API when callers can reasonably recover, and document reachable panic conditions.

- Every intentional `expect`, `assert!`, or `panic!` message should explain the invariant and include relevant values when useful. Prefer `expect("config is loaded before workers start")` over `unwrap()`.
- Pattern-match instead of check-then-unwrap. Use `if let`, `match`, or `let Some(value) = value else { ... };`.
- Treat a poisoned lock as a violated invariant unless the application has a proven recovery protocol; do not casually continue with `into_inner()`.
- Destructors must not block, panic, or report recoverable failure. Provide an explicit consuming `close`/`shutdown` method when teardown can fail, and let `Drop` perform bounded best-effort cleanup.

For public library errors:

- Use a focused enum or struct implementing `Debug`, `Display`, `std::error::Error`, and normally `Send + Sync + 'static`. Preserve wrapped causes through `source()`/`#[source]` and `From` conversions.
- Do not expose `()`, `String`, `Box<dyn Error>`, or `anyhow::Error` when callers need structured recovery. Application internals may use contextual `anyhow`/`eyre` style errors before mapping at the boundary.
- Keep display text concise, lowercase, and without trailing punctuation. Include paths, values, and operation context needed to diagnose the failure without exposing secrets.
- Do not create one crate-wide enum containing unrelated failure domains. Expose only distinctions callers can act on; keep implementation-only kinds private.
- If a fallible consuming operation can reasonably be retried, consider returning the consumed value through the error instead of forcing a clone.

Roze-specific rules:

- Prefer Roze error/result helpers at protocol boundaries: `RozeError`, `roze-result::ApiResponse`, and `roze_rpc::rpc::status_from_error`.
- Preserve request id, trace id, locale, tenant, and error code metadata.
- Categorize errors by recovery: validation/user errors, not found, auth/permission, transient upstream, timeout, rate limited, config invalid, and internal.
- Retry only transient errors. Use Roze governance/retry/breaker surfaces before local retry loops.
- Add context to internal errors before converting them to user-facing HTTP/gRPC shapes.
- Keep response messages stable and safe while retaining detailed causes for structured internal logs and metrics.

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

Prefer message passing when ownership transfer matches the workflow. When state must be shared, keep linked invariants under one lock, keep lock scopes short, never invoke user-provided callbacks while holding a lock, and do not return lock guards from public APIs.

Public clients and service futures should be `Send` and normally `Sync` where their documented use requires it. Pin important assumptions with compile-time assertions so a refactor cannot silently make a generated/native HTTP or tonic future non-`Send`.

Use `async fn` for ordinary asynchronous APIs. Return `impl Future` only when trait constraints or deliberate pre-poll setup require it. Runtime-neutral reusable libraries should accept I/O traits, executors, or spawn functions rather than secretly creating threads or assuming a particular runtime; Roze runtime crates and generated services may intentionally use the workspace Tokio lifecycle.

Roze-specific rules:

- Generated REST handlers and RPC servers run in async contexts; do not block the executor with filesystem, CPU-heavy, sleep, or synchronous network calls.
- Do not hold `MutexGuard`, `RwLockGuard`, transaction handles, or other scoped guards across `.await` unless the type and design explicitly support it.
- Use Roze cache, MQ, singleflight, registry, and gateway primitives before inventing local concurrency systems.
- Shared state inside `ServiceContext` must be `Send + Sync` when used by Roze native HTTP/Tower/tonic services.

Common pitfalls:

- `Rc` or `RefCell` in handler/server state: use `Arc` and thread-safe primitives.
- `thread::sleep` in async code: use `tokio::time::sleep`.
- `future is not Send`: ensure non-`Send` values are dropped before `.await`, use `Arc`, or keep the work local only when the runtime design allows it.
- deadlocks: keep lock ordering explicit and scopes short.
- long CPU loops in async tasks: move them to `spawn_blocking`/workers or yield cooperatively when chunking is the intended design.

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

## Idioms And Clarity

- Prefer short iterator pipelines when they read top-to-bottom; use an explicit `for` loop when the body has multiple responsibilities, complex early exits, or would become a long combinator chain.
- Use `Option`/`Result` combinators when they clarify one transformation, and `match` or `let ... else` when branches carry distinct behavior. Never use check-then-unwrap.
- Build strings with `format!`, `write!`, `push_str`, or `push` rather than chained `+`. For hot buffers, reuse capacity only after measurement shows it matters.
- Bind cleanup guards to a name such as `_guard`; `let _ = guard` drops immediately. Keep lock and transaction guards visibly scoped.
- Make types explicit when `.into()`, `.parse()`, `.collect()`, or a complex iterator result would otherwise hide intent.
- For closures that need a mix of borrowed and owned captures, clone/rebind the owned values in a small block and then use `move`; do not clone unrelated state into the closure.
- Import types and traits for unqualified use, but usually qualify free functions by module (`mem::take`, `fs::read`). Avoid glob imports except documented framework preludes, enum variants inside a local match-heavy scope, and `use super::*` in tests.
- Replace unexplained magic values with named constants whose docs state their source and operational effect.
- Keep externally visible output deterministic. Sort map-derived output or choose an ordered representation when order is part of logs, generated files, CLI output, snapshots, or tests.
- Use structured `tracing` events in services and libraries. Reserve stdout/stderr for a CLI's intentional user-facing output; do not leave `dbg!` or incidental prints in production paths.
- Comments explain invariants, trade-offs, and why a workaround exists. Do not narrate obvious statements or preserve an implementation diary in code.

## Dependencies And Cargo

Before adding a dependency, check whether Roze or the standard library already provides the capability.

Roze 1.0 development, generated production verification, and release CI are pinned to Rust 1.98.0 through the workspace toolchain. Use that pin for reproducible framework work and treat the scheduled latest-stable lane as forward-compatibility evidence. The pin is not an MSRV declaration; do not claim a fixed MSRV until a dedicated CI lane continuously verifies one.

Selection criteria:

- actively maintained
- clear license
- good docs and examples
- feature flags that avoid unused bloat
- compatible with the workspace MSRV and async runtime
- no duplicate major versions of core dependencies

Crate and module structure:

- Keep workspace crates single-purpose and flat under the repository's established layout. Split on a real ownership/build boundary, not merely because a module is long.
- Name a leaf module after its primary concept. Group modules by domain/use case rather than dumping items into `traits`, `helpers`, or `utils` buckets.
- Keep non-leaf module files focused on declarations and deliberate re-exports. Do not invent a general prelude; preserve Roze's generated application preludes as explicit framework extension points.
- Share versions and lint policy through workspace tables when the active checkout does so. Commit `Cargo.lock` for binaries/applications and follow the repository policy for libraries.

Roze-specific rules:

- Prefer existing workspace crates and `roze-*` abstractions over adding external crates.
- Keep dependency additions local to the crate that needs them.
- Use workspace dependency versions when the Roze checkout defines them.
- Avoid wildcard versions.
- Be careful with default features on network, TLS, database, metrics, and async crates.
- Prefer normal compatible version ranges over wildcard or needless exact pins. Preserve exact Git revisions where Roze reproducibility contracts require them.
- Audit dependency duplication, licenses, advisories, unused dependencies, build scripts, and proc macros proportionate to release risk.
- Set an MSRV only when CI continuously verifies it. The Roze toolchain pin is a reproducibility contract, not automatically an MSRV promise.
- Do not run hidden initialization before `main`, and do not add unstable features to a stable-toolchain workspace.

Useful checks:

```bash
cargo tree -p <crate>
cargo tree -d
cargo audit
cargo deny check
cargo check -p <crate>
cargo test -p <crate>
cargo clippy -p <crate> --all-targets -- -D warnings
```

## Tooling And Lint Policy

Scale verification to the changed surface, then broaden it for shared or public behavior:

```bash
cargo fmt --check
cargo clippy -p <crate> --all-targets --all-features -- -D warnings
cargo test -p <crate>
cargo doc -p <crate> --no-deps
```

Use [rust-lints.md](rust-lints.md) as the complete BeautifulRust machine-checkable baseline. Reconcile it with the active Roze workspace rather than creating duplicate tables or breaking generated/protocol-required shapes.

- Enforce warnings as errors in CI commands, not with `#![deny(warnings)]` in source; a new toolchain warning should not make an otherwise unchanged crate unbuildable outside the controlled CI lane.
- Fix a lint when practical. For an intentional local exception, prefer the narrowest `#[expect(lint, reason = "...")]` supported by the pinned toolchain so stale suppressions are detected. Use `#[allow]` primarily for generated or macro-expanded code that cannot use `expect` reliably.
- Keep lint configuration at the workspace or crate manifest boundary instead of scattering crate-level attributes.
- Treat clippy as mechanical evidence, not design proof. Review public contracts, generated ownership, error semantics, cancellation, and deterministic behavior separately.
- Run `cargo doc --no-deps` for reusable/public API changes so broken links and rustdoc examples are part of handback evidence.

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
- Use `Box<[T]>`, `Box<str>`, or `Arc<str>` for long-lived immutable data when their ownership model and measured allocation profile fit better than growable collections.
- Reserve capacity when a reliable size is known and consider `shrink_to_fit` only for long-lived collections whose excess capacity is materially measured.
- Reuse caller-owned buffers in proven hot APIs, batch work instead of locking or performing I/O per item, and choose a faster hasher only for trusted keys after profiling.
- Derive `Copy` only for small plain values where implicit copying is unsurprising. Do not make large buffers or secret/key material silently copyable.
- Change the global allocator only at the application boundary and only with benchmark evidence across representative workloads.

Avoid:

- optimizing without a benchmark or profile
- benchmarking debug builds
- using `HashMap` or heap allocation for tiny fixed-size data by habit
- adding `unsafe` for performance before measuring safe alternatives
- optimizing a micro-allocation while leaving algorithm, query shape, network round trips, or contention unmeasured

## Documentation And Tests

Public framework APIs and reusable crates should document what signatures cannot express:

- Give public items a concise rustdoc summary. Add `# Examples`, `# Errors`, `# Panics`, and `# Safety` sections when applicable; examples should use `?` rather than copyable `unwrap()` patterns.
- Use `//!` for crate/module-level public guidance and `///` for items. Keep the first sentence useful in generated listings and order special sections consistently: examples, errors, panics, safety.
- Document preconditions, side effects, ownership transfer, complexity, cancellation, and compatibility constraints instead of restating the item name.
- Link Rust items with rustdoc links and keep comments focused on why an invariant is sound or a workaround is required.
- Hide implementation-only conversion glue from public docs only when users cannot trigger or name it; do not hide relevant compatibility behavior.

Test observable contracts rather than implementation shape:

- Put private-unit tests beside the module and public API behavior in integration tests when that boundary matters. Treat doctests as executable public examples.
- Add a failing regression test before fixing a bug when feasible. Test properties such as round trips, ordering, idempotency, cancellation, and retry boundaries instead of mirroring branches or constants.
- Cover supported feature combinations and platform-specific paths proportionate to risk. A feature that changes compilation needs compile evidence in its supported matrix.
- Make time, randomness, I/O, and network behavior controllable through existing Roze test helpers or narrow injected interfaces. Do not make `Default` perform I/O.
- Fuzz parsers and untrusted contract inputs when risk justifies it; run Miri for unsafe boundaries that support it. Keep assertions diagnostic with relevant input values.

## Unsafe And FFI

Default to safe Rust. Treat `unsafe` as a reviewed boundary, not an escape hatch for borrow or lifetime issues.

Only consider `unsafe` for:

- FFI
- low-level abstractions with clear invariants
- measured hot paths where safe alternatives are insufficient

Mark an API `unsafe` only when violating caller obligations can cause undefined behavior, not merely because the operation is destructive or operationally dangerous. Do not use `transmute` to cast enums or extend lifetimes, and do not add `unsafe impl Send/Sync` to silence the compiler.

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

### Unsafe Review Checklist

Use this checklist before accepting unsafe code in Roze runtime crates, generated helpers, or application extension points:

- Confirm the unsafe block is necessary. If the goal is only to avoid ownership, lifetime, or borrow-checker friction, redesign with safe Rust.
- Keep unsafe blocks small and wrapped by a safe API whenever possible.
- Require a `// SAFETY:` comment for every unsafe block that states the invariant, who upholds it, and why the operation is sound.
- Require `# Safety` docs for every public `unsafe fn`; document caller obligations, valid input ranges, aliasing rules, lifetime requirements, and thread-safety assumptions.
- Validate raw pointers before dereference: non-null, aligned, initialized, within allocation bounds, and valid for the required lifetime.
- Avoid creating `&T` or `&mut T` from raw pointers unless Rust aliasing rules are actually satisfied.
- Prefer `NonNull<T>` for stored non-null pointers and `PhantomData` when a wrapper owns or borrows data not visible in its fields.
- Prefer `MaybeUninit<T>` for initialization protocols; avoid invalid zeroed values, especially references, enums, bools, and non-zero types.
- Check `#[repr(C)]`, field layout, padding, alignment, and endian assumptions for FFI or binary layout code.
- Do not let Rust panics cross FFI boundaries; catch or prevent panics around `extern "C"` callbacks.
- Define allocation ownership across FFI: which side allocates, which side frees, and which function performs release.
- Keep `CString`/buffer owners alive while passing raw pointers to C; never pass a pointer from a temporary.
- Do not manually `Drop` memory owned by foreign code, and do not let Rust automatically drop foreign-owned memory.
- For `impl Send` or `impl Sync`, verify synchronization, aliasing, and lifetime invariants under concurrent access.
- Add tests around boundary conditions and run `cargo miri` when the target crate and dependencies support it.
- A safe wrapper must remain sound for every sequence of safe calls; if callers can trigger undefined behavior without using `unsafe`, the abstraction is unsound regardless of internal comments.

Roze-specific unsafe guidance:

- Generated service glue should almost never need unsafe. If unsafe appears in generator output, treat it as a design smell and require a generator test plus a safety rationale.
- Runtime crates should expose safe abstractions to generated services and app code; unsafe should stay behind narrow crate-local boundaries.
- FFI or low-level integrations must preserve Roze context, error, tracing, and shutdown behavior at the safe wrapper boundary.

## Web And Cloud-Native Roze Services

Use this guidance when working on generated REST/RPC/stream services, gateway-facing code, MQ consumers, service discovery, config, or deployment behavior.

Core constraints:

- Handlers and RPC methods are async service entry points. Do not block them with synchronous I/O, CPU-heavy loops, `thread::sleep`, or long-held locks.
- Shared service state must be thread-safe. Use `ServiceContext` with `Arc`-backed clients, pools, registries, caches, and publishers.
- Request-scoped data belongs in extractors, Roze context, function arguments, or local variables; do not place it in application-wide state.
- Cross-cutting behavior belongs in Roze middleware, gateway governance, RPC metadata, MQ envelope handling, or generated adapters before custom local code.
- Every protocol boundary should preserve request id, trace id, tenant, locale, auth metadata, deadline/cancellation intent, structured errors, logs, and metrics labels.
- Readiness should report real dependency state through Roze health checks rather than ad hoc endpoints.

REST/API guidance:

- Prefer generated route/handler/logic boundaries. Keep handlers thin and move business behavior into `src/logic/**`.
- Use generated DTO validation and Roze response/error helpers before writing custom extractors or response envelopes.
- Prefer built-in middleware for trace, recover, metrics, CORS, timeout, rate limit, breaker, max connections, shedding, gunzip, body limit, auth/JWT, and idempotency.
- Add custom Tower layers only when the built-in config/middleware surface cannot express the behavior, and preserve Roze context and metrics.
- Do not expose Axum request, response, extractor, or router types from generated services; use Roze native HTTP APIs.
- Avoid per-request client construction. Put HTTP clients, DB handles, registry clients, cache clients, MQ publishers, and search clients in `ServiceContext`.

RPC and service-discovery guidance:

- Prefer generated tonic server/client adapters and `roze_rpc` context/error metadata helpers.
- Pass `&roze_context::Context` through app-owned logic when downstream calls need request metadata.
- Use Roze registry/cached resolver/gateway discovery surfaces before adding service-discovery code.
- Treat timeout, retry, breaker, rate-limit, fallback, and outlier behavior as governance configuration. Do not hide it inside business logic.

Stream/MQ guidance:

- Use `roze_mq`, `roze_kafka`, or `roze_nats` abstractions before broker-specific app code.
- Make ack, nack, retry, DLQ, replay, idempotency, and envelope version behavior explicit.
- Keep handlers idempotent where retries are possible.
- Do not block consumer tasks; offload CPU-heavy work and bound concurrency where needed.
- Emit topic, group, partition, offset, attempt, outcome, request/trace id, and error kind where available.

Cloud-native review checklist:

- Does startup fail fast for invalid required config, and preserve last-good config during hot reload failures?
- Are `/healthz`, `/readyz`, `/startupz`, `/metrics`, or protocol equivalents backed by Roze health/metrics primitives?
- Are shutdown and cancellation paths clear for HTTP, RPC, stream workers, background tasks, and pools?
- Are retries bounded and limited to transient errors?
- Are generated files left generator-owned and business changes kept in application-owned extension points?
- Are deployment artifacts, smoke scripts, and docs updated when public behavior changes?

## Refactoring Existing Rust

For Rust refactors, analyze impact before editing:

- Find definitions and references with `rg` and LSP when available.
- Check public re-exports and generated ownership boundaries.
- Check macro-generated code and docs references.
- Preserve generated Roze files by editing generators/templates instead of generated glue.
- Keep changes small enough for targeted tests.
- Characterize undocumented behavior before changing it, especially byte output, ordering, error classification, and public signatures.
- Preserve quirks that are outside the requested change and call them out instead of silently "cleaning them up."
- Fix the full class of issue requested, not only the cheapest instance. Keep public behavior stable unless changing it is the task; call out necessary signature or compatibility changes explicitly.
- When tests are absent, add characterization coverage before structural changes. Keep changes small enough for reviewers to connect each edit to behavior.

For review procedure, severity, and report structure, use [rust-review.md](rust-review.md). For cross-language migrations, use [rust-porting.md](rust-porting.md). Those references depend on this file rather than repeating its general Rust rules.

## Handback Checklist

- Relevant `cargo fmt --check`, clippy with `-D warnings`, tests, and public-library docs pass, or unavailable checks are named.
- No reachable production `unwrap`; intentional panic/expect/assert paths explain the invariant and relevant values.
- Public inputs use borrowed/general standard forms where appropriate, without clone-to-own or leaked ownership wrappers.
- Domain invariants use types rather than sentinel values or public boolean switches.
- Public types implement the applicable common traits; error types are structured and retain sources.
- `Ord`, equality, and hashing agree; externally visible output is deterministic.
- Async code does not block, hold guards across `.await`, lose cancellation/context, or create unbounded work.
- Unsafe is necessary, minimal, documented, wrapped safely, and tested at its boundary.
- Public docs cover errors, panics, safety, side effects, and examples as applicable.
- Application code remains direct; public library code preserves SemVer; generated files remain generator-owned; Roze built-ins are used before local infrastructure.

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
- Sentinel values, public boolean switches, or loosely related primitive parameters where a domain type should enforce the invariant.
- `#![deny(warnings)]` in source or broad lint suppressions without a reason.
- Line-by-line ports that preserve manual memory, exception, global-state, or class patterns instead of behavior.

## Standards Provenance

BeautifulRust synthesizes the Google Rust Style Guide, official Rust API Guidelines, Effective Rust, Rust Design Patterns, and Microsoft Pragmatic Rust Guidelines. This reference merges their operational rules with Roze rather than copying five overlapping digests. When a rule needs deeper rationale or a stable checklist code, consult BeautifulRust's corresponding `skills/rust-best-practices/references/` file at the pinned baseline; when sources disagree, this merged reference and the active Roze contract are authoritative for Roze work.
