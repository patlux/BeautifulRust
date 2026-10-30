# Publishing a library

Rules that only matter when a crate has external consumers: a crate on crates.io, a crate
other teams depend on by git URL, or a `-sys` crate. Internal workspace members that only
the same repository uses may skip this file; `SKILL.md` already covers them. The text here
was part of the upstream `SKILL.md` and was moved out in the `patlux/BeautifulRust` fork
so that application work does not load it on every task.

Sources: the Rust API Guidelines, Effective Rust, Microsoft's Pragmatic Rust Guidelines
and the Google Rust Style Guide. The per-source digests in this directory have the full
reasoning.

## Naming extras

- `iter` / `iter_mut` / `into_iter` return types named `Iter` / `IterMut` / `IntoIter`;
  `keys()` returns `Keys`.
- Checked accessors get unsafe `_unchecked` siblings; `take_foo()` moves a field out,
  `with_foo(|f| ..)` gives closure access.
- Put `mut` in type order: `as_mut_slice`, not `as_slice_mut`.
- Feature names are bare: `std`, `serde`, `alloc`, never `use-std` or `no-std`.

## Derive the standard traits eagerly

Downstream crates cannot add std traits to your types (orphan rule), so you must. Every
public type derives or implements `Debug`, and where the semantics allow it `Clone`,
`Copy`, `PartialEq` / `Eq`, `PartialOrd` / `Ord`, `Hash`, `Default`, `Display`,
`FromStr`, `From` / `TryFrom`, `FromIterator` / `Extend`, `IntoIterator`. Serialization
impls (`serde`) go behind a feature named after the crate. A missing derive is a breaking
change to add later only in the sense that callers had to work around it; a present one is
a promise you cannot remove.

## Accept the most general input

Library code that only iterates takes `impl IntoIterator<Item = T>`, not `&[T]` or
`Vec<T>`. Read-only paths, strings and byte buffers take `impl AsRef<Path>` /
`impl AsRef<str>` / `impl AsRef<[u8]>`. Store `PathBuf` and `String` in the struct.

## Visibility and evolution

- Private by default. Fields stay private unless the struct is passive C-style data.
  Widen to `pub(crate)` or `pub(super)` before `pub`. Making an item more public later is
  non-breaking; the reverse is breaking, and every `pub` item is a promise.
- Mark public enums and public-field structs that cross crate boundaries
  `#[non_exhaustive]` unless the set is a stable ontology. When matching one, list the
  variants you handle and add a wildcard arm.
- Seal traits you don't want implemented downstream, and say so in the docs:

  ```rust
  pub trait TheTrait: private::Sealed { /* ... */ }
  mod private {
      pub trait Sealed {}
      impl Sealed for usize {}
  }
  ```

- Library code returns iterators, not a `Vec` collected only to return, behind `impl
  Iterator<Item = T>` (internal APIs) or a newtype (public APIs you must keep stable).
- Expose each public item at exactly one path. Don't `pub use` a foreign crate's items
  unless you are an umbrella crate. Avoid leaking third-party types into your public API;
  feature-gate `serde` impls behind a feature named `serde`. If a dependency's type must
  appear, re-export it so callers can name the matching version.
- Keep `Rc`, `Arc`, `Box`, and `RefCell` out of public signatures. Take `&T`, `&mut T`,
  or `T` and let callers decide how to own things.
- Mark a function `const` only if it will stay `const`. Don't expose `ErrorKind` enums
  or other internals that lock in failure modes you consider private.
- Closure parameters go last; don't take more than one.

## Library error types

In addition to the rules in `SKILL.md` section 6:

- If a fallible function consumes an argument, return it inside the error so the caller
  can retry without cloning (`FromUtf8Error::into_bytes`).
- Never expose `anyhow::Error`, `eyre::Report` or `Box<dyn Error>` in a library's public
  API, not even behind a type alias.
- Hide `From<PrivateError> for PublicError` impls that users never trigger with
  `#[doc(hidden)]`.
- A library never catches a panic and continues, and never panics on bad input. Pair any
  panicking convenience with a fallible twin and document it under `# Panics`.

## Runtime neutrality

Library code doesn't spawn raw threads or assume an async runtime; take an executor, a
spawn function, or `impl AsyncRead` / `impl Read` from the caller. Public types and the
futures they return are `Send` (and usually `Sync`); pin that with a compile-time
assertion as shown in `SKILL.md` section 8.

## Features

- Features are additive and bare-named: `std`, `serde`, `alloc`, `test-util`. Never
  `no-std`, `use-serde`, or a feature that removes or changes a public item.
- Don't feature-gate struct fields or trait methods. Put mocks and fakes behind
  `test-util`.
- Every feature combination is tested in CI (`cargo hack --feature-powerset` or an
  explicit matrix).

## Semver and toolchain

- Breaking changes include removing or renaming public items, adding a variant to an enum
  that isn't `#[non_exhaustive]`, adding a field to a publicly constructible struct,
  adding a trait bound, making a trait non-object-safe, and raising the MSRV when the
  crate documents one.
- Deprecate in a minor release (`#[deprecated(since = "..", note = "..")]`) before
  removing in a major one.
- Use the latest stable edition for new crates and set an MSRV deliberately
  (`rust-version` in `Cargo.toml`), tested in CI.
- Commit `Cargo.lock` for binaries; for libraries, either policy is acceptable as long as
  CI also tests with `-Z minimal-versions` or the lock file is refreshed regularly.
- Use ranges in dependency requirements (`"1"`, `"1.4"`), never `"*"` or a needless
  `"="`. A `=` pin in a library forces it on every downstream user.
- `cargo semver-checks` before every release.

## Rustdoc

- `///` on every public item; `//!` at the top of every public module and the crate root.
  The first sentence is a single line of about fifteen words, because it becomes the
  summary in listings.
- Sections, in this order, as applicable: `# Examples`, `# Errors`, `# Panics`,
  `# Safety`. Examples use `?` with the `fn main` hidden behind `# ` lines, never
  `unwrap()`, because users copy them verbatim. Describe parameters in prose, not a table.
- Document what the signature can't say: preconditions, side effects, ownership
  transfer, complexity, and why. Don't restate the type or the name.
- Link items with `` [`Type`] ``. Hide impls users never trigger with `#[doc(hidden)]`.
- `cargo doc --no-deps` (here: `mbx doc --no-deps`) must be clean; enable
  `rustdoc::broken_intra_doc_links` as an error and `missing_docs` as a warning in
  `[lints.rust]`.
- Crate-level docs show the primary use case end to end, name the feature flags, and
  state the MSRV.

## Release checklist

- `cargo fmt`, `mbx clippy --all-targets --all-features -- -D warnings`, `mbx test`,
  `mbx doc --no-deps` are clean on the MSRV and on stable.
- `cargo semver-checks` reports no unintended breaking change; the version bump matches.
- `CHANGELOG` names every public change, with migration notes for breaking ones.
- `cargo publish --dry-run` succeeds; `cargo package --list` contains no stray files.
- Tag the release in Git and create the tag before publishing, so the published source is
  reproducible.
