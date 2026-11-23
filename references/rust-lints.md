# Rust Lint Policy

Use this reference when adding or reviewing Cargo/clippy policy. It synchronizes the machine-checkable BeautifulRust standard from baseline `b9718436` and adapts adoption to Roze workspaces.

The active repository remains authoritative. Reuse an equivalent existing workspace policy instead of creating a second table. Apply new lints deliberately to handwritten crates; generated code, proc-macro output, FFI bindings, binaries, and framework-required signatures may need narrow documented exceptions.

## Enforcement Model

- Configure lints in `[workspace.lints]` or crate `[lints]`, not scattered crate attributes.
- Keep lint levels at `warn` in manifests and enforce them with `cargo clippy ... -- -D warnings` in controlled CI/release commands.
- Do not put `#![deny(warnings)]` in source.
- Fix findings where practical. For intentional exceptions, prefer the narrowest `#[expect(lint, reason = "...")]`; use `#[allow]` for generated/macro code or when the pinned toolchain cannot use `expect` reliably.
- Do not enable library-only policy blindly for binaries or generated services.

## Workspace Lints

For a new handwritten library/workspace without an equivalent policy, use this as the BeautifulRust baseline and reconcile it with the active Roze checkout:

```toml
[workspace.lints.rust]
unsafe_op_in_unsafe_fn = "warn"
missing_debug_implementations = "warn" # library/public types
missing_docs = "warn"                  # reusable library APIs
unreachable_pub = "warn"
rust_2018_idioms = { level = "warn", priority = -1 }

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }

# Panics and fallible APIs.
unwrap_used = "warn"
expect_used = "allow"
panic_in_result_fn = "warn"
missing_panics_doc = "warn" # public libraries
missing_errors_doc = "warn" # public libraries
missing_assert_message = "warn"

# Suppressions.
allow_attributes = "warn"
allow_attributes_without_reason = "warn"

# Unsafe.
undocumented_unsafe_blocks = "warn"
multiple_unsafe_ops_per_block = "warn"

# API shape and ownership.
fn_params_excessive_bools = "warn"
struct_excessive_bools = "warn"
clone_on_ref_ptr = "warn"
redundant_clone = "warn"
rc_buffer = "warn"

# Public evolution; normally library-only.
exhaustive_enums = "warn"

# Idioms.
string_add = "warn"
wildcard_imports = "warn"

# Debug/output leftovers; print lints are normally library-only.
dbg_macro = "warn"
todo = "warn"
print_stdout = "warn"
print_stderr = "warn"

# Lossy conversions.
as_conversions = "warn"
cast_possible_truncation = "warn"
cast_sign_loss = "warn"

# Pedantic rules intentionally not enforced by this standard.
module_name_repetitions = "allow"
must_use_candidate = "allow"
module_inception = "allow"
enum_glob_use = "allow"
```

Member crates inherit workspace policy with:

```toml
[lints]
workspace = true
```

The suppression lints require a toolchain that supports reasoned attributes; Roze's active pinned toolchain decides availability.

## Clippy Configuration

Place this in workspace `clippy.toml` when the active checkout has no conflicting setting:

```toml
max-fn-params-bools = 0
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-print-in-tests = true
```

`max-fn-params-bools = 0` makes every boolean function parameter visible for review. Private local helpers may retain a boolean when the call sites are adjacent and the meaning is obvious; use a narrow reasoned exception rather than weakening the workspace threshold.

## Roze Reconciliation

- Generated contracts and framework-owned templates must be fixed at the generator when a lint indicates a real issue. Use scoped generated-code exceptions only when the generated shape is required by another tool or protocol.
- Roze's generated `src/logic/prelude.rs` and documented framework preludes are intentional exceptions to generic wildcard/prelude advice.
- CLI binaries may print intentional user-facing output; libraries and services should use structured tracing.
- Public wire enums generated from `.api`/`.proto` contracts may need exhaustive semantics. Do not add `#[non_exhaustive]` if it breaks the contract or generated compatibility model.
- `missing_docs`, `missing_*_doc`, `print_*`, and `exhaustive_enums` are usually library policy. Evaluate them separately for application binaries and generated services.
- `as_conversions` can be noisy around protocol/FFI boundaries; preserve checked conversion and document a narrow exception when the wire type guarantees the range.

## Verification

Run the affected crate first, then workspace-wide CI when changing shared policy:

```bash
cargo clippy -p <crate> --all-targets --all-features -- -D warnings
cargo test -p <crate>
```

Lint policy is not proof of correct API, protocol, lifecycle, or generated ownership. Use [rust-review.md](rust-review.md) for those checks.
