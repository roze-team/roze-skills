# Official Rust Standards And Authority

Use this reference whenever a task depends on Rust language semantics, standard-library guarantees, editions, Cargo behavior, SemVer compatibility, rustdoc, compiler/clippy lints, unsafe soundness, platform support, or MSRV.

Rust does not publish one formal specification covering the language, standard library, compiler, Cargo, documentation, and unsafe programming. The [Rust Reference](https://doc.rust-lang.org/stable/reference/) is the primary language reference, but the official documentation explicitly says it is not a formal specification and may contain omissions. Treat the official documents below as a versioned authority set rather than copying them into this skill.

## Match The Project Version

Before consulting official documentation:

1. Read `rust-toolchain.toml`/`rust-toolchain`, workspace `Cargo.toml`, package `edition`, `rust-version`, and CI matrix.
2. Use documentation matching the pinned toolchain when behavior changed across releases. Versioned docs use URLs such as `https://doc.rust-lang.org/1.98.0/reference/`.
3. Use stable/latest documentation only when the project intentionally tracks that channel or the rule is confirmed unchanged for the pinned version.
4. Treat edition and compiler version separately. Crates from different editions interoperate, and each crate's manifest controls its edition.
5. Do not use nightly or unstable behavior unless the active repository explicitly opts in and its toolchain/feature gate pins that choice.

## Authority By Question

| Question | Primary official source |
|---|---|
| Syntax, expressions, types, traits, attributes, macros, ownership semantics, ABI, layout, UB | [Rust Reference](https://doc.rust-lang.org/stable/reference/) |
| `std`, `core`, `alloc`, primitive and standard API contracts | [Standard Library documentation](https://doc.rust-lang.org/stable/std/) for the active toolchain |
| Edition differences and migration | [Rust Edition Guide](https://doc.rust-lang.org/edition-guide/) |
| Manifests, dependencies, features, resolver, workspaces, build scripts, profiles, publishing | [Cargo Book and Cargo Reference](https://doc.rust-lang.org/stable/cargo/reference/) |
| Package compatibility and breaking changes | [Cargo SemVer Compatibility](https://doc.rust-lang.org/stable/cargo/reference/semver.html) |
| `rust-version`/MSRV declaration and resolver behavior | [Cargo Rust Version](https://doc.rust-lang.org/stable/cargo/reference/rust-version.html) |
| Documentation syntax, doctests, links, attributes, output | [rustdoc book](https://doc.rust-lang.org/stable/rustdoc/) |
| Standard formatting | [Rust Style Guide](https://doc.rust-lang.org/stable/style-guide/) and rustfmt from the pinned toolchain |
| Compiler flags, targets, codegen, lints, diagnostics | [`rustc` book](https://doc.rust-lang.org/stable/rustc/) and [`rustc` lint listing](https://doc.rust-lang.org/rustc/lints/listing/) |
| Clippy categories, configuration, and lint behavior | [Clippy documentation](https://doc.rust-lang.org/stable/clippy/) and the lint page for the pinned toolchain |
| Unsafe abstractions, variance, unwinding, FFI, uninitialized memory | [Rust Reference](https://doc.rust-lang.org/stable/reference/) first, then [Rustonomicon](https://doc.rust-lang.org/stable/nomicon/) as an incomplete companion |
| Stable-language learning and examples | [The Rust Programming Language](https://doc.rust-lang.org/stable/book/) and [Rust By Example](https://doc.rust-lang.org/stable/rust-by-example/) |

Use `rustc --explain <code>`, `rustc -W help`, `cargo help <command>`, local `rustup doc`, and dependency rustdoc for the exact installed toolchain when available.

## Precedence In Roze Work

Apply these layers without conflating them:

1. Version-matched official Rust documentation governs language, standard-library, Cargo, rustdoc, lint, edition, and unsafe semantics.
2. Active Roze contracts and generated ownership govern framework behavior, wire compatibility, lifecycle, config, errors, observability, and extension points.
3. [rust-engineering.md](rust-engineering.md) and BeautifulRust govern idiomatic design and engineering choices where official semantics permit several valid approaches.
4. Repository instructions, tests, and docs define intended project behavior but cannot make undefined behavior sound or override compiler/Cargo semantics.

When official sources disagree, prefer the version-matched Reference or tool-specific reference over tutorials and companion books. The Rustonomicon itself directs readers to prefer the Reference when they conflict. If compiler behavior and the Reference disagree, do not invent a rule: minimize the example, check the pinned compiler/release notes and known issues, and report the uncertainty.

## Language And Standard Library Contracts

- Do not infer layout, ABI, niche, drop order, aliasing, lifetime extension, auto-trait, or optimization guarantees from observed code generation. Require an explicit Reference or standard-library guarantee.
- Default Rust representation does not promise C-compatible field layout. Use `repr(C)`, `repr(transparent)`, integer enum representation, or alignment/packing attributes only with their documented guarantees and SemVer consequences.
- A safe abstraction must prevent safe callers from causing undefined behavior. Privacy and module boundaries are part of the soundness proof when unsafe code relies on invariants maintained by safe code.
- Validate behavior of standard APIs against docs for the active toolchain, including panic conditions, platform differences, feature stabilization, and const availability.
- Treat implementation details and current compiler optimizations as non-contractual unless an official source explicitly guarantees them.

## Editions And Toolchain

- Preserve each crate's edition unless migration is requested. Do not introduce newer-edition syntax or capture assumptions into a crate whose manifest/toolchain does not support them.
- Use the Edition Guide and `cargo fix --edition` workflow for migration, then review semantic/capture/macro changes and run the full supported test matrix.
- Keep `rust-version` honest: setting it declares package support and influences dependency resolution. Verify the claimed version in CI before presenting it as an MSRV.
- Treat a higher MSRV, removed target, newer OS/service requirement, or new nightly dependency as a compatibility decision requiring documentation and release review.

## Cargo And Features

- Read Cargo's resolver and feature documentation before changing workspace resolver versions, optional dependencies, default features, target dependencies, build dependencies, or feature forwarding.
- Prefer ordinary caret-compatible requirements for registry dependencies unless the repository has a stronger reproducibility policy. Avoid `*` and unbounded ranges; preserve Roze's exact Git revision contracts.
- Cargo features should be additive in engineering design. Account for feature unification and test relevant combinations; do not assume a feature is isolated because one crate did not enable it directly.
- Keep `Cargo.lock` according to package role and repository policy, and do not hand-edit it.
- Build scripts and proc macros execute code during builds; include them in supply-chain, reproducibility, and target-support review.

## SemVer Review

For every public crate/API change, consult Cargo's full SemVer guide rather than relying on a short remembered checklist. Common breaking or possibly breaking areas include:

- removing, renaming, moving, hiding, or conditionally compiling public items
- changing function parameters/returns, strengthening bounds, capture behavior, object safety, or required trait items
- adding fields to publicly constructible structs or variants to exhaustive enums
- changing represented layout/alignment/ABI, `no_std` support, MSRV, platform/environment requirements, or public dependency types
- removing or changing features, optional dependencies, defaults, or behavior enabled through feature unification
- adding inherent methods/trait implementations that create name-resolution or inference ambiguity
- changing runtime behavior, ordering, error classification, panic behavior, or safety contracts even when source compatibility remains

Use deprecation and compatibility shims before removal where project policy allows. Roze 1.x public CLI, generator output, config, metrics, and runtime ordering also follow Roze's stricter documented compatibility contracts.

## Documentation, Formatting, And Lints

- Use pinned rustfmt output as the formatting authority; do not maintain a competing manual layout style.
- Validate reusable crates with rustdoc, doctests, and broken-link checks. Use `//!` for crate/module documentation and `///` for item documentation according to rustdoc behavior.
- Use rustc's lint listing or `rustc -W help` for compiler lint names/default levels in the pinned toolchain.
- Use Clippy categories deliberately. Never enable `clippy::restriction` as a group; the official Clippy documentation requires choosing restriction lints case by case because they may conflict or reject reasonable code.
- Treat new compiler/clippy warnings as toolchain changes to reconcile. Keep warnings-as-errors in controlled CI rather than assuming every dependency/toolchain update is warning-stable.

The project's BeautifulRust-aligned baseline remains in [rust-lints.md](rust-lints.md); official lint documentation decides whether a lint exists and what it means for the active toolchain.

## Unsafe And FFI

- Use the Reference for what operations are unsafe and what constitutes undefined behavior. Use the Rustonomicon for design reasoning, not as a complete or higher-priority specification.
- Document every unsafe contract: valid values, initialization, alignment, provenance/aliasing, lifetime, thread-safety, unwind, ABI, and allocation ownership as applicable.
- Keep unsafe state private behind the smallest module that can uphold its invariants. Audit safe mutations that can invalidate those invariants, not only lines inside `unsafe` blocks.
- Do not assume the absence of memory unsafety means correctness: deadlocks, leaks, races at the application level, overflow policy, aborts, and destructive operations can remain safe Rust while still being defects.
- Verify FFI against platform ABI and target documentation, prevent unwinding across incompatible boundaries, and allocate/free through compatible ownership domains.

## Non-Authoritative Sources

Blog posts, Stack Overflow, generated explanations, old RFC discussions, compiler internals, Unsafe Code Guidelines drafts, and observed assembly can provide context but do not replace stable version-matched documentation. RFCs explain design intent; the stabilized Reference/tool docs define the shipped contract. Use the Unstable Book only for an explicitly pinned nightly feature.

## Maintenance Trigger

Recheck this authority map and affected guidance when Roze changes its pinned toolchain, edition, workspace resolver, MSRV policy, public crate SemVer policy, lint baseline, unsafe boundary, or supported targets. Update rules only when the corresponding official contract or active Roze behavior changes; do not vendor entire official books into this repository.
