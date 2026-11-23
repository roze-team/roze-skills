# Porting Existing Systems To Rust And Roze

Use this reference when translating C, C++, Python, Go, JavaScript, or TypeScript into Rust. Apply [rust-engineering.md](rust-engineering.md) to the resulting code. This workflow follows [`morganlinton/BeautifulRust`](https://github.com/morganlinton/BeautifulRust) baseline `b9718436` and adds Roze contract/generator boundaries.

Port observable behavior, not the source language's structure. The target should look like Rust a Rust developer would write for the same contract, not C/Python/Go expressed with Rust syntax.

## Understand The Original

Before writing Rust:

1. Read implementation, tests, callers, data formats, deployment/config behavior, and failure handling.
2. Inventory every public input, output, side effect, and failure. Include ordering, tie-breaking, empty input, whitespace/case rules, numeric limits, encoding, cancellation, retries, and error-versus-valid-edge decisions.
3. Separate observable behavior from language accidents such as manual allocation, out-parameters, exception hierarchies, global mutable state, fixed buffers, classes that only group functions, and runtime type checks.
4. Preserve unknown quirks and record ambiguity. Do not silently fix source bugs while claiming compatibility.
5. Identify the Roze boundary: REST/RPC/stream/model/search contract, background service, CLI, runtime crate, or internal library.

## Port Tests First

Translate original tests before implementation and keep a recognizable mapping between them. If tests are missing, add characterization tests from the behavior inventory. Test output bytes/order and failure classes when observable, not only successful values.

Use differential tests on representative and edge-case inputs when both implementations can run. A bounded set that covers actual input classes is usually more useful than a huge compatibility harness for exotic inputs the product never receives.

## Design The Rust API

Design signatures and types from behavior, then implement against the ported tests.

### C And C++

| Source pattern | Rust target |
|---|---|
| `malloc/free`, `new/delete`, create/destroy pairs | Owned `String`, `Vec`, `Box`, or a narrow RAII wrapper with `Drop` |
| `NULL` meaning absence | `Option<T>` |
| Error codes or `errno` | `Result<T, E>` with a typed error implementing `Display` and `Error` |
| Out-parameters | Values in `Ok` or fields on the error variant |
| Flag integers/bitmasks | Named options struct, enum, or `bitflags` when flags are independent |
| Pointer plus count | Slice or iterator |
| `char *` plus length | `&str`/`String` for valid text, `&[u8]`/`Vec<u8>` for bytes |
| Fixed buffers | Growable buffers unless truncation/limit is observable behavior |
| Preprocessor constants/macros | `const`, enum, function, generic, or trait before a Rust macro |
| Mutable global state | Caller-owned state or a lifecycle-owned Roze context resource |

Do not introduce raw pointers or `unsafe` unless a real FFI boundary remains. Keep FFI in a narrow `-sys`/adapter layer, document ownership and ABI, and expose a safe Rust API.

### Python

| Source pattern | Rust target |
|---|---|
| Exception hierarchy | Focused error enum/structs with source data preserved |
| `None` | `Option<T>` |
| Magic strings representing a closed set | Enum with `FromStr`/`Display` when text conversion is public |
| Dict/class used as a record | Struct with named fields |
| Observable dict insertion order | `Vec`, index-preserving map, or explicit sorting; not a plain `HashMap` |
| Duck typing or `isinstance` branches | Trait for open behavior or enum for a closed set |
| Unbounded integer | Deliberately sized integer plus checked conversion/arithmetic where input can exceed it |
| Stable keyed sorting | `sort_by_key`/`sort_by` with tie behavior verified |
| Parse wrapped in `try/except` | `Result` propagation or explicit match with typed mapping |

Use Rust standard string/iterator behavior unless exact source-runtime behavior is required for real inputs. Document differences in Unicode, whitespace, integer range, and formatting instead of recreating the entire Python runtime.

### Go, JavaScript, And TypeScript

- Go `(value, error)` becomes `Result`; `nil` becomes `Option`; interfaces become traits or enums; goroutines/channels become bounded tasks/channels chosen for actual concurrency requirements.
- JavaScript/TypeScript `null` and `undefined` become `Option`; thrown errors become `Result`; plain records become structs; union-like strings become enums; `number` becomes a deliberately chosen integer or `f64` with conversion checks.
- Preserve source ordering guarantees and asynchronous cancellation/error behavior. Do not assume Rust `HashMap` order or silently turn rejected promises into logged-and-ignored failures.

## Integrate With Roze

- Express external REST/RPC/stream contracts in `.api` or `.proto` and regenerate framework-owned boundaries.
- Express data models in `.ent`/supported model inputs and use generated repositories rather than porting an ORM layer literally.
- Put long-lived clients, pools, registries, caches, publishers, and ported stateful adapters in `ServiceContext` or documented application hooks.
- Map source errors into domain errors internally, then use Roze HTTP/RPC/MQ boundary helpers so request/trace/tenant/locale/deadline metadata survives.
- Replace source retry, breaker, rate-limit, discovery, health, logging, cache, MQ, transaction/outbox, and lifecycle infrastructure with Roze built-ins when they provide the required contract.
- Keep business behavior in application-owned logic and extensions; do not write it into generated handlers/adapters.

## Keep The Port Proportionate

- Do not recreate the source runtime, grow the public API, or add builders/custom iterators/generic abstraction unless callers need them.
- Keep the source algorithm unless it is incorrect or materially unsuitable at realistic scale. Record intentional algorithm changes and their behavioral consequences.
- Use standard traits Rust callers expect when useful, but do not implement every possible trait speculatively.
- Match realistic inputs and explicit compatibility requirements. Record exotic differences instead of building compatibility shims without evidence.
- Avoid new dependencies when the standard library or Roze already supplies the capability.

## Verify

Run the original suite when possible, then the target's relevant checks:

```bash
cargo build -p <crate>
cargo test -p <crate>
cargo clippy -p <crate> --all-targets --all-features -- -D warnings
```

Also run contract generation/validation, compile-smoke, integration, or production-reference checks required by the affected Roze boundary. Compare both implementations on the same inputs when practical.

## Porting Notes

For a substantial port, keep concise reviewer-facing notes containing:

- API mapping from each original public function/type to its Rust/Roze equivalent
- test mapping from each original test to its Rust test
- deliberate differences and why they are acceptable
- preserved quirks that must not be "cleaned up" accidentally
- algorithm or dependency changes
- verification commands and results, including checks that could not run

Do not duplicate the shared Rust rules in the notes; link to the code and explain only decisions specific to the port.
