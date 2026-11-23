# Rust Review Workflow

Use this reference when reviewing Rust code, a crate, a diff, or a Roze generator change. Apply the shared rules in [rust-engineering.md](rust-engineering.md) and the machine-checkable policy in [rust-lints.md](rust-lints.md); this file defines review procedure and reporting, not a second copy of those rules.

This workflow follows [`morganlinton/BeautifulRust`](https://github.com/morganlinton/BeautifulRust) baseline `b9718436`, with Roze-specific ownership and framework checks added.

## Establish Scope

Before judging code:

1. Read repository instructions, `Cargo.toml`, relevant docs, public callers, tests, and feature flags.
2. Decide whether the target is a public library, application crate, generated service, generator/template, runtime crate, CLI, or FFI boundary. Public libraries require stricter API evolution; application code should remain concrete and proportionate.
3. Identify generated files and their owner. Review the generator or template when generated output is wrong; do not recommend hand-editing generated glue.
4. For Roze services, read the relevant `.api`, `.proto`, `.ent`, `roze-service.yaml`, config, and `ServiceContext` wiring before reviewing an adapter in isolation.
5. Preserve deliberate project conventions and documented quirks. Flag a trade-off rather than treating every difference from a general style guide as a defect.
6. When a finding depends on language, layout, unsafe, Cargo, edition, rustdoc, lint, MSRV, or SemVer semantics, verify it against [rust-official-standards.md](rust-official-standards.md) and the official document matching the pinned toolchain.

## Run Evidence First

When the toolchain and dependencies are available, run the narrowest relevant commands and broaden for shared/public changes:

```bash
cargo fmt --check
cargo clippy -p <crate> --all-targets --all-features -- -D warnings
cargo test -p <crate>
cargo doc -p <crate> --no-deps
```

Use the repository's documented feature matrix, target-specific checks, generator tests, and Roze smoke/gate scripts when they apply. Do not paste raw compiler or clippy output as the review. Group related diagnostics, explain the consequence, and say which commands could not run.

A review is read-only unless the user also asks for fixes. Do not format or rewrite files merely to produce findings.

## Review Order

Review in this order so style cannot hide real defects.

### 1. Correctness And Soundness

- Safe-call unsoundness, unjustified `unsafe`, invalid raw-pointer/FFI assumptions, or `unsafe impl Send/Sync` without proven invariants.
- Reachable panics, unchecked indexing, overflow, lossy casts, impossible-state assumptions reachable from external input, and destructors that panic or block.
- `Ord`, `PartialOrd`, `Eq`, and `Hash` implementations that disagree about semantic fields.
- Locks or scoped guards held across `.await`, user callbacks invoked under locks, leaked guards, deadlock-prone ordering, and broken cancellation/shutdown paths.
- Nondeterministic map iteration where output, generated files, signatures, snapshots, or tests require stable order.
- Roze readiness remaining green while dependencies or the service are draining.

### 2. Error And Boundary Semantics

- Expected failures represented as panics, unusable public error types, discarded sources/context, or errors exposing sensitive/internal details.
- HTTP/RPC/MQ boundaries that bypass Roze context, numeric error contracts, request/trace metadata, deadlines, structured logs, or protocol conversion helpers.
- Retries applied to permanent failures, unbounded retries, fallback that hides corruption, or missing idempotency around replayable work.
- Failure paths that replace the target during transactional generation or leave partially applied runtime state.

### 3. Public API And Evolution

- Borrowed parameter shapes, ownership wrappers, public fields, boolean switches, sentinel values, third-party types, missing common traits, or naming that obscures cost and ownership.
- Breaking SemVer changes: removed or renamed public items, new variants on exhaustive public enums, fields added to publicly constructible structs, stronger bounds, changed object safety, or changed error/ordering behavior.
- APIs that permit invalid states, collect unnecessarily, expose implementation details, or add speculative genericity.
- Public fallible, panicking, or unsafe APIs missing `# Errors`, `# Panics`, or `# Safety` documentation.

### 4. Async, Lifecycle, And Performance

- Blocking I/O or CPU-heavy work on async executors, non-`Send` service futures, unbounded task/concurrency growth, per-request client creation, or resources outside `ServiceContext` without a reason.
- Missing health registration, cancellation propagation, graceful shutdown, stop timeouts, or exactly-once lifecycle outcomes.
- Algorithm/query shape, unnecessary allocation or cloning, per-item network/database work, lock contention, and caching without measured need or invalidation rules.

### 5. Roze Architecture And Ownership

- Handwritten code duplicating a `roze-*` capability, generated RPC client, model/search repository, middleware, health, config, cache, MQ, transaction/outbox, registry, gateway, or governance surface.
- Business logic in generated route, handler, DTO, RPC adapter, protobuf include, OpenAPI, or service-context files.
- Contract behavior changed without updating IDL, generator tests, docs, SDK/OpenAPI artifacts, compatibility checks, and production evidence where required.
- Custom code that loses Roze request context, typed config/redaction, structured errors, tracing, metrics labels, or readiness/draining behavior.

### 6. Clarity, Documentation, And Tests

- Clone-to-appease-borrow-checker, check-then-unwrap, unreadable iterator chains, string `+`, broad imports, stale suppressions, debug output, and comments that narrate syntax.
- Missing regression/behavior tests, tests coupled to implementation, nondiagnostic assertions, doctests users cannot copy, and untested feature/platform combinations.
- Missing explanation for invariants, side effects, ordering, cancellation, complexity, workarounds, or unsafe boundaries.

## Severity

- **Must fix:** correctness bug, soundness hole, security or data-integrity risk, reachable operational panic, broken protocol/lifecycle contract, or public API mistake that becomes expensive to correct after release.
- **Should fix:** real maintainability, diagnosability, testability, performance, or compatibility cost without immediate incorrect behavior.
- **Consider:** naming, local simplification, polish, or an intentional trade-off worth documenting.

Do not inflate severity because a lint fires. Base severity on user-visible and maintenance consequences.

## Report Format

Lead with findings ordered by severity. Every finding should include:

- a tight `file:line` location, or `whole file` when no narrower location exists
- the violated rule or Roze contract
- the concrete consequence
- the smallest viable fix, using a short code snippet only when it clarifies the change

Group repeated instances into one finding and list the relevant locations. Keep the report proportionate; do not pad a short review with style nits. End with open questions or assumptions, then a brief note on what is already sound and which verification commands ran. If there are no findings, say so explicitly and name residual test or environment gaps.

When lint policy is missing or materially weaker, recommend the applicable entries from [rust-lints.md](rust-lints.md) and identify which findings those entries would catch. Do not silently install a new lint policy during a review.
