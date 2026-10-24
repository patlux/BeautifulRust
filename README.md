<p align="center">
  <img src="assets/banner.png" alt="Pixel art banner: a crab on a rocky shore at sunset, under the title Beautiful Rust and the subtitle Idiomatic Rust, distilled">
</p>

# Beautiful Rust

> **This is the `patlux` fork** of [morganlinton/BeautifulRust](https://github.com/morganlinton/BeautifulRust),
> trimmed for application and workspace code and adapted to a Pi / `skills`-CLI setup. See
> [What this fork changes](#what-this-fork-changes). Everything below that section is
> upstream's README.

## What this fork changes

- **Narrower trigger.** `SKILL.md` loads when you write, refactor, review or design Rust,
  not on every mention of `cargo` or `.rs`. One-line fixes, dependency bumps and build
  problems do not need 400 lines of style guidance.
- **Application and workspace first.** Rules that only matter for a crate with external
  consumers (visibility and evolution, `#[non_exhaustive]`, sealed traits, re-exports,
  feature naming, semver, MSRV, rustdoc conventions, library error-type details) moved
  unchanged into `skills/rust-best-practices/references/publishing-a-library.md`.
  `SKILL.md` points there when a published crate is involved.
- **`mbx` instead of bare `cargo`.** Compiler-invoking Cargo subcommands run through
  [Mr. Boxington](https://github.com/jdx/mr-boxington) (`mbx clippy`, `mbx test`, `mbx doc`),
  which manages per-worktree target directories. `cargo fmt`, `cargo metadata`, `cargo
  tree` stay plain. Without `mbx`, the same `cargo` subcommand is the equivalent command.
- **Project guidance wins.** A repository's `AGENTS.md`, `Makefile` or `justfile` lint and
  test entry points override the skill's defaults.

The Claude Code plugin manifest, the evals and the five source digests are untouched.

### Install with the `skills` CLI

In a repository that keeps its skills under `skills/` and tracks them in
`skills-lock.json`:

```bash
npx --yes skills@1.5.22 add patlux/BeautifulRust --skill rust-best-practices --agent openclaw --copy --yes
```

Later updates come from this fork with `npx --yes skills@1.5.22 update --project --yes`.

### Sync this fork with upstream

```bash
git fetch upstream
git merge upstream/main
# resolve conflicts in skills/rust-best-practices/SKILL.md and README.md, re-check the
# publishing-a-library.md split, then:
git push origin main
```

---

A Claude Code plugin with one skill, `rust-best-practices`, that teaches Claude to write
idiomatic, robust, maintainable Rust. Once installed, it loads automatically whenever a
task involves Rust, cargo, crates, the borrow checker, clippy, or `.rs` files.

The guidance is distilled from five sources, with a condensed digest of each bundled in
the skill so Claude can consult the original reasoning when it needs to:

- [Google Rust Style Guide](https://google.github.io/styleguide/rustguide.html)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) (official rust-lang checklist)
- [Effective Rust](https://www.lurklurk.org/effective-rust/) by David Drysdale
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) (idioms, patterns, anti-patterns)
- [Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/)

Where the sources disagree, the skill picks a side and says why.

## Install in Claude Code

One step, from a terminal:

```bash
claude plugin install beautiful-rust --marketplace morganlinton/BeautifulRust
```

Or inside a Claude Code session:

```
/plugin marketplace add morganlinton/BeautifulRust
/plugin install beautiful-rust@beautiful-rust
```

Add `--scope project` to the terminal command to enable it for everyone working in the
current repository (each collaborator still runs the install once).

To try it for a single session without installing:

```bash
git clone https://github.com/morganlinton/BeautifulRust
claude --plugin-dir ./BeautifulRust
```

The skill is namespaced under the plugin, so you can invoke it explicitly with
`/beautiful-rust:rust-best-practices`. You rarely need to: it triggers on its own.

## Install in claude.ai

claude.ai takes a zip of the skill folder, with the folder as the zip root:

```bash
cd skills && zip -r ../rust-best-practices.zip rust-best-practices -x '*.DS_Store'
```

Upload it under **Customize > Skills** (on some plans this lives under Settings).
Code execution must be enabled first. Skills uploaded to claude.ai also sync down into
Claude Code sessions signed in with the same account.

## What the skill covers

`skills/rust-best-practices/SKILL.md` is about 500 lines, organized so Claude can scan
it while coding:

1. Mindset: Rust-shaped code, invalid states unrepresentable, application vs library
2. Pre-handback checklist: fmt, clippy with `-D warnings`, test, doc; lint policy
3. Ownership and borrowing: what to do instead of cloning for the borrow checker
4. Types and API design: naming, newtypes, common traits, builders, visibility, generics vs `dyn`
5. Errors and panics: `Option` vs `Result` vs panic, library vs application error types
6. Everyday idioms: iterators, combinators, `format!`, imports, drop guards
7. Concurrency and async: `Send` assertions, lock discipline, `async fn`, yield points
8. Unsafe and FFI: when it's allowed, `SAFETY:` comments, soundness, bindgen
9. Crates, modules, dependencies: workspace layout, features, semver
10. Documentation and comments
11. Tests
12. Performance, in order
13. Final checklist

`skills/rust-best-practices/references/` holds one digest per source for deeper lookups.

## Layout

```
BeautifulRust/
├── .claude-plugin/
│   ├── plugin.json         plugin manifest
│   └── marketplace.json    lets this repo act as its own marketplace
├── skills/
│   └── rust-best-practices/
│       ├── SKILL.md
│       └── references/
│           ├── google-rust-style-guide.md
│           ├── rust-api-guidelines.md
│           ├── effective-rust.md
│           ├── rust-design-patterns.md
│           └── microsoft-rust-guidelines.md
└── README.md
```

## Testing the skill

`skills/rust-best-practices/evals/` holds three test cases in the layout `claude plugin eval`
reads: a library API design task, an application CLI task, and a refactor-this-module task
with an input file. Each case has the prompt, a description of a good result, and one
assertion per file under `graders/`. Run them with and without the skill from the skill
directory:

```bash
claude plugin eval . --allow-tools Write,Edit,Bash --scaffold
```

These are real model runs and are billed. In the last comparison the skill passed 29 of
30 assertions against 24 of 30 without it, at roughly a third more tokens per task.

## Updating

Edit the skill, bump `version` in `.claude-plugin/plugin.json`, commit, and push.
Installed users pick up the new version with:

```bash
claude plugin update beautiful-rust
```

Validate before pushing:

```bash
claude plugin validate .
```

## Sources and attribution

The guidance in this skill is a synthesis. The five digests in
`skills/rust-best-practices/references/` are paraphrased summaries written for this
project; they restate each source's advice in condensed form and link back to the
original. Read the originals for the full reasoning.

| Source | Author | License |
|---|---|---|
| [Google Rust Style Guide](https://google.github.io/styleguide/rustguide.html) | Google | CC BY 3.0 |
| [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) | The Rust Project | MIT or Apache-2.0 |
| [Effective Rust](https://www.lurklurk.org/effective-rust/) | David Drysdale, published by O'Reilly | CC BY-NC-ND 4.0 (online edition) |
| [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) | rust-unofficial contributors | MPL-2.0 |
| [Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/) | Microsoft | MIT |

The skill text, the digests, the banner, and the generator script in this repository
are MIT licensed (see [LICENSE](LICENSE)). The crab is a crab, not the Rust mascot.

## License

MIT. See [LICENSE](LICENSE).
