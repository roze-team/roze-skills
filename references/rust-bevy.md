# Bevy Rust Exceptions

Read this reference only when the target `Cargo.toml` depends on `bevy` or `bevy_ecs`. It synchronizes the BeautifulRust Bevy exceptions checked against Bevy 0.20.0 at baseline `b9718436`. Confirm the project's locked Bevy version first; release-specific APIs and observer/message behavior may differ.

The shared rules in [rust-engineering.md](rust-engineering.md) still apply except for the explicit Bevy conventions below.

## Preserve Bevy Conventions

- Take system parameters such as `Res<T>`, `ResMut<T>`, `Query<D, F>`, `Single<D, F>`, `Commands`, `Local<T>`, `MessageReader<M>`, and `MessageWriter<M>` by value. Bevy constructs these borrow-like parameters for each run; do not change them to references or flag them as needless pass-by-value.
- Use Bevy's documented `use bevy::prelude::*`. This does not permit an application-defined wildcard prelude.
- Plain-data components/resources may expose public fields when no cross-field invariant needs protection. Keep fields private when methods must preserve an invariant.
- Complex query types and many system parameters are normal. Use an alias or derived `SystemParam` when a reused concept deserves a name, not merely to silence a lint.

For a Bevy crate using the BeautifulRust lint baseline, reconcile these exceptions:

```toml
[lints.rust]
elided_lifetimes_in_paths = "allow"

[lints.clippy]
needless_pass_by_value = "allow"
type_complexity = "allow"
too_many_arguments = "allow"
```

Keep these exceptions scoped to Bevy crates.

## Correctness Checks

- Two system queries that can mutate the same component set panic at runtime even when they compile. Make sets provably disjoint with filters, or use `ParamSet` when overlapping access is intentionally sequential. The same conflict applies to `Res<T>` plus `ResMut<T>` of one resource.
- `Commands` are deferred until a synchronization point. Explicitly order systems when one must observe another system's spawn, insert, or despawn in the same frame.
- Systems in an unordered tuple may run in either order. Use `.chain()`, `.before()`, `.after()`, or system sets for actual data/message dependencies.
- Do not unwrap `Query::single()` or `get(entity)` on normal gameplay state. Use `Single`, an explicit `let Ok(...) = ... else { return };`, or a deliberate error handler.
- Commands targeting an entity already despawned can fail under the default handler. Prevent duplicate despawn/update or use `try_*` commands when disappearance is expected.
- Each `MessageReader` consumes each message once. Order a reader after its writer when same-frame reaction is required.

## Testing Systems

- Drive systems through a headless `App` without `DefaultPlugins` when rendering is unnecessary.
- Insert required resources explicitly and advance time manually for deterministic tests.
- Add focused behavior tests plus a smoke test that installs the complete plugin and calls `app.update()` at least twice. Query conflicts and missing resources often appear only when systems first execute together.
