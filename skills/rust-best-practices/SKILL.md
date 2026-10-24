---
name: rust-best-practices
description: >-
  Write idiomatic, robust, maintainable Rust. Use this skill when you write new Rust code
  (crates, modules, functions, types, traits, error types, async, unsafe or FFI code),
  refactor or review existing Rust, or design an API or an error-handling strategy. Not
  needed for a one-line fix, a dependency bump, a build or toolchain problem, or reading
  Rust only to answer a question. Synthesized from the Google Rust Style Guide, the Rust
  API Guidelines, Effective Rust, Rust Design Patterns, and Microsoft's Pragmatic Rust
  Guidelines. This fork keeps the rules for application and workspace code in SKILL.md,
  moves crate-publishing rules to a reference, and runs Cargo through mbx.
---

# Rust best practices

Distilled from the Google Rust Style Guide, the Rust API Guidelines, *Effective Rust*,
*Rust Design Patterns* and Microsoft's Pragmatic Rust Guidelines; one digest per source in
`references/`. Where they disagree, this file picks.

Scope: application code and internal workspace crates. Rules that only matter for a crate
with external consumers (semver, features, sealed traits, rustdoc, re-exports) live in
`references/publishing-a-library.md`; read it before touching a published crate's API.

Project guidance wins: a repository's `AGENTS.md`, `Makefile`, `justfile` or CI config
overrides the entry points and conventions below.

## 1. Mindset

- **Write Rust-shaped code.** Errors are `Result`, resources are ownership plus `Drop`,
  polymorphism is traits and enums, nullability is `Option`. `Deref` to a "base" struct,
  `Manager`/`Factory` types and `throw_if_null` helpers are ported habits, not Rust.
- **Make invalid states unrepresentable.** Enums instead of bool flags and "must be X if
  Y" comments, newtypes for units and IDs, fallible constructors that validate once.
- **Application code is not library code.** Library code follows the API rules strictly.
  Application code favors simplicity: concrete types, `anyhow` errors, `Vec` returns, no
  speculative generality.
- **Let the tools do the work.** rustfmt formats, clippy checks idioms, the compiler
  checks memory. Spend attention on design.

## 2. Before you hand back code

```bash
cargo fmt
mbx clippy --all-targets --all-features -- -D warnings
mbx test
mbx doc --no-deps   # crates other crates depend on: catches broken doc links
```

`mbx` (Mr. Boxington) wraps Cargo with managed per-worktree target directories and passes
flags through unchanged: `mbx build|check|test|clippy|run|bench|doc`. Plain `cargo` is
right for commands without build output (`fmt`, `metadata`, `tree`, `add`). Never set
`CARGO_TARGET_DIR`, `--target-dir`, `RUSTC_WRAPPER` or sccache alongside it. Without
`mbx`, the same `cargo` subcommand is the equivalent command.

- Fix lints, don't silence them. For a genuinely wrong lint use
  `#[expect(clippy::name, reason = "...")]` on the narrowest item (Rust 1.81+; older
  toolchains: `#[allow]` plus a comment). `#[allow]` only in generated or macro code.
- Never `#![deny(warnings)]` in source; new toolchains add warnings. Enforce
  `-D warnings` in CI.
- Configure lints once in `Cargo.toml` (`[lints.rust]`, `[lints.clippy]`,
  `[workspace.lints]`). Baseline: `unsafe_op_in_unsafe_fn`,
  `missing_debug_implementations`, `clippy::all`; where tolerated
  `clippy::undocumented_unsafe_blocks`, `clippy::allow_attributes_without_reason`.

## 3. Ownership and borrowing

**Prefer unique ownership.** `Rc`/`Arc` only when shared ownership is part of the design,
`RefCell`/`Mutex` only when interior mutability is. They move checks to run time; use them
on purpose.

**Don't clone to satisfy the borrow checker.** Such a clone silently forks data. Try, in
order:

1. Extend a temporary with a named `let`, or shorten a borrow with a block.
2. Split a long expression into typed intermediates so the failing step is visible.
3. Move out of `&mut self` with `std::mem::take`, `std::mem::replace` or `Option::take`.
4. Split a large struct so disjoint fields can be borrowed separately (the compiler
   tracks disjoint field borrows within one function, not across method calls).
5. Clone small immutable data deliberately, with a comment if it's not obvious.
6. Indices into an owning `Vec`, or `Rc<RefCell<_>>` with `Weak` back-pointers, as a
   last resort. No self-referential structs.

```rust
// Move out of a &mut enum without cloning.
fn a_to_b(e: &mut MyEnum) {
    if let MyEnum::A { name, x: 0 } = e {
        *e = MyEnum::B { name: std::mem::take(name) };
    }
}
```

**Accept the most general borrowed type:** `&str` not `&String`, `&[T]` not `&Vec<T>`,
`&T` not `&Box<T>`, `Option<&T>` not `&Option<T>`. Read-only paths, strings and bytes may
take `impl AsRef<Path | str | [u8]>` on parameters, never on fields (store `PathBuf`,
`String`).

**Take ownership only when you need it, by value.** Store or consume → `T`; read → `&T`;
mutate in place → `&mut T`. Never `&T` plus an internal `.clone()`. Return several values
as a tuple or struct, not `&mut` out-parameters (reusing a caller buffer like `Read::read`
is the exception).

**Prefer owned data in structs.** `&'a T` in a struct forces the lifetime onto every
container; `String`/`Vec<u8>` is rarely the bottleneck. Name lifetimes on types
descriptively (`'src`, `'conn`).

**Freeze after setup:** build with `let mut`, then `let data = data;`.

**Keep `Rc`, `Arc`, `Box`, `RefCell` out of shared signatures.** Take `&T`, `&mut T` or
`T`. A shared service implements cheap `Clone` by wrapping its state in `Arc<Inner>`
internally.

## 4. Types and API design

### Naming

- Standard casing; acronyms are one word (`Uuid`, `HttpClient`). Type parameters short,
  from the main bound (`R: Read`, `W: Write`).
- No `get_` prefix: `len()`, `name()`, `name_mut()`, `set_name(T)`. Bare `get` only when
  there is one obvious thing to get (`Cell::get`).
- Conversion prefixes signal cost: `as_` (free borrow→borrow), `to_` (expensive),
  `into_` (consumes `self`).
- Constructors: `new`, `with_capacity`/`new_with_foo`, `try_new` (fallible), `from_foo`
  (pair with a `From` impl). I/O resources use domain verbs (`File::open`,
  `TcpStream::connect`).
- Error types read verb-object-error: `ParseBoolError`.
- No `Service`, `Manager`, `Factory`, `Helper`, `Util` suffixes; name the thing
  (`Bookings`, `Dispatcher`). Don't repeat the module name (`foo::Id`, not `foo::FooId`).
- `len()` comes with `is_empty()`.

### Model the domain with types

- **Newtypes** for semantic distinctions (`Miles(f64)` vs `Kilometers(f64)`); a type alias
  does not prevent mixing. `#[repr(transparent)]` when layout matters; `derive_more` for
  forwarding. A newtype with an invariant (`Month`) validates in `TryFrom`/`FromStr`/
  `try_new` and offers no infallible `From` from the raw type.
- **No `bool` parameters in shared signatures.** `report(true)` is unreadable and cannot
  grow a third mode. Use a two-variant enum or two methods: `Widget::new(Size::Small,
  Shape::Round)`, not `Widget::new(true, false)`. Exception: a private helper whose call
  sites sit a few lines away. In a refactor an existing `bool` parameter is a smell to
  remove, not a detail to preserve.
- **Named fields over tuples**, except single-field newtypes and very local code.
- **`Option` for absence, never sentinels** (`-1`, `""`). **Data-carrying enums** instead
  of "field X only matters when Y is set".
- **Strongest std type:** `PathBuf`/`&Path`, `Duration`, `NonZeroU32` internally (plain
  integers at API boundaries).
- **Independent flags** → `bitflags`; an enum models exactly one choice.

### Standard traits

- Derive when every field supports it: `Debug` on every type (hand-write to redact
  secrets), `Clone` (not for unique resources or key material), `Copy` only for small
  bitwise types, `PartialEq`/`Eq`, `PartialOrd`/`Ord` (never `PartialOrd` alone when a
  total order exists), `Hash` consistent with `Eq`, `Default`. `Display` for anything
  humans read. `#[must_use]` on pure functions and on types whose discarding is a bug.
- Implement `From`/`TryFrom`, never `Into`/`TryInto` (blanket impls). Prefer
  `from`/`into` over `as`, which allows lossy casts. `FromStr` instead of an ad-hoc
  `from_str`; `FromIterator`/`Extend` on collections; `IntoIterator` for owned and
  borrowed forms. `Deref` only on smart pointers and owning collections, never to fake
  inheritance. Operators only for their mathematical meaning.
- No derivable bounds on struct definitions (`#[derive(Clone)] struct Good<T>`, not
  `struct Bad<T: Clone>`); bounds go on the `impl` blocks that need them.
- Hand-written `Ord` means hand-written `PartialEq`, `PartialOrd` and `Hash` that compare
  the same fields; a derived `PartialEq` next to a custom `Ord` breaks `BTreeMap`,
  `sort` and `dedup`. Document which fields equality ignores.

### Constructors, builders, defaults

- Provide `new()` even with `Default`, and make them agree. Derive `Default` so callers
  can write `..Default::default()`.
- Up to two optional parameters: `new`, `with_a`, `with_b`. Beyond that, or when
  construction validates or has side effects: `Foo::builder()` → `FooBuilder`, setters
  named after fields (`timeout(d)`) and infallible, `build() -> Result<Foo, Error>` does
  all validation including cross-field checks.
- Four or more loose arguments get grouped into semantic types: `Deposit::new(account,
  amount)`, not four `&str`/`u64`.

### Visibility

Private by default; fields private unless the struct is passive data. Widen `pub(crate)`
→ `pub(super)` → `pub`. Widening later is non-breaking; narrowing is not, and every `pub`
item of a published crate is a promise (`references/publishing-a-library.md`).

### Traits and generics

- A trait exists for generic programming (`T: Trait`, `impl Trait`, `dyn Trait`) or as an
  extension trait (`FooExt`), not to organize code. Put essentials in inherent methods;
  trait impls forward to them.
- Prefer in order: concrete types; generics; `dyn Trait`, wrapped in your own struct
  (`struct DynStore(Arc<dyn Store>)`). Don't nest generics on service-level types
  (`Service<Backend>` yes, `Service<Backend<Store>>` no).
- Decide object safety early; exclude generic or `Self`-returning methods with
  `where Self: Sized`. Keep required methods minimal and default the rest, as `Iterator`
  does from `next`.
- Clear receiver → method. No receiver → free function, not `Database::check(params)`.
  Associated functions are for construction.
- Closures take the most general bound (`FnOnce` > `FnMut` > `Fn`), not bare `fn`
  pointers, and go last. Parameter order is consistent across a crate: call-specific
  first, ubiquitous context (loggers, allocators) last.
- Macros only when functions, generics and traits cannot do the job; `macro_rules!` over
  proc macros; macro input looks like its output; a macro never changes a type's kind or
  a function's signature.

## 5. Refactoring someone else's code

- Fix every smell the request is about, not just the cheap ones: borrow-checker clones,
  `&String`/`&Vec<T>` parameters, `unwrap` on fallible paths, `Rc<RefCell<_>>` in
  signatures, `get_` getters, `bool` parameters, `+` string concatenation,
  `#![deny(warnings)]`, error types without `Error`, nondeterministic `HashMap` order in
  output.
- Keep public behavior identical unless changing it is the point: same operations, same
  success and failure semantics, same output bytes. Panic → `Err` is the change callers
  want. Dropping an error variant, removing a derive or changing a constructor's inputs is
  not; a necessary signature change is listed with before, after and reason.
- Preserve quirks you don't understand and flag them instead of silently "fixing" them.
- No tests? Pin current behavior with a few before touching the module.

## 6. Errors and panics

| Situation | Use |
|---|---|
| Absence is normal (`first()`, `checked_add`) | `Option<T>` |
| The caller may need to know what went wrong | `Result<T, E>` |
| A bug or violated invariant inside your code | `panic!` / `assert!` / `expect("why")` |

- **Panics are not exceptions.** Never report bad input with one, never use
  `catch_unwind` for control flow (no-op under `panic = "abort"`, may leave state
  inconsistent), never catch and continue in reusable code. A justified panicking
  convenience has a fallible twin (`get`/`get_unchecked`, `new`/`try_new`) and a
  `# Panics` section.
- **Every intentional panic names the relevant values:** `assert!(buf.len() >= HEADER,
  "buffer too small: {} bytes", buf.len())`, `expect("config is loaded before workers
  start")`. `expect("reason")` over `unwrap()`; don't restate the condition.
- **Pattern-match, don't check-then-unwrap:** `if let Some(v) = opt` or
  `let Some(v) = opt else { return .. };`, never `if opt.is_some() { opt.unwrap() }`.
- **Lock poisoning is fatal:** `.lock().expect("lock poisoned")`; no `into_inner`
  recovery.
- **Applications:** `anyhow` or `eyre` with `.context("opening config")`, one app-level
  error type, `main` may return `Result<(), anyhow::Error>`. Error messages name the
  input they failed on (file, key, line).
- **Crates other crates depend on** (workspace members included) return a dedicated
  error type from every fallible shared function: implements `std::error::Error`
  (`Debug` + `Display`), is `Send + Sync + 'static`, implements `source()`; `Display` is
  lowercase, concise, no trailing punctuation (`"invalid header length"`); `From<Upstream>`
  so `?` converts, `map_err` only for foreign errors or context. Shape: `thiserror` enum
  with one variant per cause, or a struct with private `kind` and `is_xxx()` accessors.
  Not one crate-wide enum of every failure; never `()`, `String`, `Box<dyn Error>` or
  `anyhow::Error` in a shared signature.
- **Destructors never fail, panic or block.** `close(self) -> Result<()>` for fallible
  teardown; `Drop` does best effort.
- **Document failure modes:** `# Errors` for any `Result`, `# Panics` for any reachable
  panic, `# Safety` for every `unsafe fn`.

## 7. Everyday idioms

- **Iterators over hand loops** when the pipeline is short and reads top-down; collect
  fallible steps into `Result<Vec<_>, E>` and `?`. Keep `for` when the body is large,
  does several jobs or needs awkward early returns. Bind a long chain to a name instead of
  putting it in a `for` header; `for x in it` over `.for_each`.
- **`Option`/`Result` combinators** (`map`, `and_then`, `ok_or_else`, `unwrap_or_else`,
  `as_ref`, `as_deref`) when clearer than `match`, `match` when not.
- **`format!`/`write!` with inline args** (`format!("{name}: {count}")`), never chained
  `+` or `str::replace`; `write!` into an existing buffer in hot paths.
- **Explicit types where inference obscures intent:** `.into()`, `.parse()`,
  `.collect()`, complex iterator results. `let x: Vec<u32> = ..` over a turbofish.
- **Drop guards** bound to a name (`let _guard = ..`); `let _ = ..` drops immediately.
- **Closures capturing mixed ownership:** rebind in a block, then `move`:

  ```rust
  let job = {
      let cfg = cfg.clone();
      move || run(cfg, data)
  };
  ```

- **Imports:** one `use` per module path; types and traits unqualified (`Regex`), free
  functions module-qualified (`mem::take`, `fs::read`). A trait used in one function is
  imported there. No glob imports except `use super::*` in tests and `use Enum::*` inside
  a function that matches on it.
- **Magic numbers** become documented `const`s: where the value comes from, what changing
  it affects.
- **Deterministic output:** sort or use `BTreeMap` before printing or serializing
  `HashMap` contents.
- **Logging:** `tracing`/`log`, not `println!`/`dbg!`, except a CLI's own stdout.

## 8. Concurrency and async

- Shared types and the futures they return are `Send` (usually `Sync`). Pin it so a
  refactor cannot regress it silently:

  ```rust
  const _: () = {
      const fn assert_send<T: Send>() {}
      assert_send::<MyClient>();
  };
  ```

  An `Rc` or `RefCell` borrow held across `.await` makes a future `!Send`.
- Message passing over shared state. When sharing: linked invariants under one lock, lock
  scopes of a few lines, no user closures while holding a lock, no `MutexGuard` returned
  from a public method.
- `async fn` over `fn -> impl Future`, except in traits or when setup must run outside
  the async block.
- CPU-bound work in async yields (`tokio::task::yield_now().await` every tens of
  microseconds) or moves to `spawn_blocking`.
- Reusable code neither spawns raw threads nor assumes a runtime; it takes an executor, a
  spawn function or `impl AsyncRead`/`impl Read` from the caller. The binary decides.

## 9. Unsafe and FFI

- `unsafe` for exactly three reasons: an abstraction safe code can't express, a
  benchmarked performance need, FFI or platform calls. std and reviewed crates already
  contain the audited `unsafe`; use them first.
- Every `unsafe` block or impl has a `// SAFETY:` comment; every `unsafe fn` a `# Safety`
  section. Enable `unsafe_op_in_unsafe_fn`.
- The module boundary is the soundness boundary: smallest module that upholds the
  invariant, wrapped in a safe API. A safe function that can reach UB through any
  sequence of safe calls is unsound, and unsound code is never acceptable.
- `unsafe` marks UB risk, not danger (`delete_database()` is safe). No `transmute` to cast
  enums or dodge lifetimes; no `unsafe impl Send` to silence the compiler.
- Miri and adversarial inputs for unsafe code.
- FFI: `#[repr(C)]`, sized integers, `CString`/`CStr`, allocate and free on the same side,
  RAII wrappers around C-owned pointers, null checks returning `Result`, `catch_unwind`
  at the boundary. `bindgen` in a `foo-sys` crate; logic in a safe core crate.

## 10. Crates, modules, dependencies

- Small single-purpose crates in a flat `crates/` workspace; never a crate inside another
  crate's `src/`. Share versions and lints via `[workspace.dependencies]` and
  `[workspace.lints]`.
- A leaf module is named after its primary item (`Foo` in `foo.rs`); non-leaf modules hold
  only `mod` declarations and re-exports. Essentials at the crate root, the rest grouped
  by use case (`account`, `network`), not by kind (`traits`, `errors`). Re-export
  individually, never by glob; no prelude.
- Every dependency costs compile time, binary size and supply-chain exposure (`build.rs`
  and proc macros included). Check what the workspace already pulls in first. Version
  ranges (`"1"`, `"1.4"`), never `"*"` or a needless `"="`. Commit `Cargo.lock` for
  binaries. Audit with `cargo tree --duplicates`, `cargo deny`, `cargo udeps`,
  `cargo audit`.
- No code before `main()` (no `ctor` crates); no unstable features on stable; latest
  stable edition for new crates.
- Features, semver, MSRV, public re-exports: `references/publishing-a-library.md`.

## 11. Documentation and comments

- Document what the signature can't say: preconditions, side effects, ownership transfer,
  complexity, why. Don't restate the name or type (`/// Removes an element.` on
  `remove_element` is noise).
- `///` on items other modules or crates call: one-line summary first, then `# Errors`,
  `# Panics`, `# Safety` as applicable. Full rustdoc conventions for published crates:
  `references/publishing-a-library.md`.
- `//` explains why, not what: `// SAFETY:`, `// TODO(issue-123):`, non-obvious values,
  workarounds with a link. No design journals or rule-compliance tables in code; document
  the end state.

## 12. Tests

- Private internals: `#[cfg(test)] mod tests { use super::*; .. }`. Public behavior:
  `tests/`, preferred when either would do. Doc examples are tests too.
- Test observable behavior and properties (monotonic, round-trip, idempotent), not
  restated constants or mirrored branches.
- Failing test before the bug fix. Cover each feature combination and platform path.
- I/O, time, randomness and network are mockable: `new_mocked() -> (Self, MockCtrl)`
  behind `test-util`, not a `Default` that performs I/O.
- `cargo fuzz` for parsers of untrusted input; `criterion`/`divan` with
  `std::hint::black_box` for benchmarks; Miri over `unsafe`.
- Assertions carry context: `assert_eq!(got, want, "input {input:?}")`.

## 13. Performance, in order

Usability first, optimize after measuring. Then: `Vec::with_capacity` or `collect()` over
push loops; caller-owned buffers in hot APIs (`fn read_into(&mut self, buf: &mut Vec<u8>)`);
`Box<[T]>`, `Box<str>`, `Arc<str>` for long-lived immutable sequences; `shrink_to_fit` on
long-lived collections built without a reservation; a faster hasher (`foldhash`, `FxHash`)
only for trusted keys; batch work instead of per-item locking; applications may set a
`#[global_allocator]` such as `mimalloc`. `Copy` for small plain types, never large ones.

## 14. Final checklist

- `cargo fmt`, `mbx clippy -- -D warnings`, `mbx test` clean, or the project's own
  entry points are.
- No `unwrap()` outside tests and `main`; every `expect`, `assert`, `panic` names the
  relevant values.
- Shared functions take `&str`, `&[T]`, `impl AsRef<_>` or `impl IntoIterator`, not
  `&String`/`&Vec<T>`.
- Every type derives `Debug` and the applicable common traits; shared error types
  implement `Error + Display + Send + Sync + 'static`.
- No `bool` parameters in shared signatures; no hand-written `Ord` without matching
  `PartialEq` and `Hash`.
- No `.clone()` whose only job is to satisfy the borrow checker.
- No glob imports, preludes or glob re-exports.
- Every `unsafe` block has `// SAFETY:`; every `unsafe fn` has `# Safety`.
- Application code stayed simple; a published crate also passed
  `references/publishing-a-library.md`.

## References

- `references/publishing-a-library.md`: crates with external consumers: visibility and
  evolution, `#[non_exhaustive]`, sealed traits, re-exports, features, semver, MSRV,
  rustdoc, library error-type details, release checklist.
- `references/rust-api-guidelines.md` (https://rust-lang.github.io/api-guidelines/): all
  54 checklist items for a public API.
- `references/effective-rust.md` (https://www.lurklurk.org/effective-rust/): generics vs
  `dyn`, `Result` vs panic, iterators vs loops, borrow-checker tactics, dependencies, CI.
- `references/rust-design-patterns.md` (https://rust-unofficial.github.io/patterns/):
  idioms, patterns, anti-patterns with code.
- `references/microsoft-rust-guidelines.md` (https://microsoft.github.io/rust-guidelines/):
  error structs, panic policy, unsafe policy, async, workspace layout, performance.
- `references/google-rust-style-guide.md` (https://google.github.io/styleguide/rustguide.html):
  formatting, imports, module layout, accessor naming, application vs library.
