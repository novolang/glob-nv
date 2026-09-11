# Changelog

All notable changes to glob-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-09

- **Declares its layer**: `layer = "core"` with `host_modules = ["walk"]` in the manifest — the public API measures to that budget and `novo pkg publish` checks the code against it.  No code changed.  The layers are described under Design in the [publishing guide](https://novo-lang.org/docs/publishing.html#design).

## [0.1.0] — 2026-09-07

The first release: the matcher, the walker, and the two decisions a
glob library has to make in public.

### Added

- `glob` — matching, which allocates nothing. `matches` and
  `matches_with` read a pattern against a path, where `*`, `?` and a
  character class each stop at `/` and a whole-segment `**` crosses
  them; `matches_name` and `matches_name_with` are `fnmatch`'s reading,
  where the candidate is one name and `*` reaches every byte of it.
  `check`, `reason` and `is_valid` refuse a pattern with a reason and a
  byte offset before a caller matches a thousand paths against it.
  `may_descend` is the walker's prefix question, asked with the same
  machine rather than a second one. `has_magic` and
  `literal_prefix_len` are what a walk is planned with. `Options`,
  `options`, `shell_options` and `ignore_case` carry ASCII case folding
  and the shell's leading-dot rule.
- `walk` — expansion, which allocates and carries `[fs]`. `expand`,
  `expand_with` and `expand_depth` expand a pattern against a directory
  tree, depth-first with every level sorted, skipping what they cannot
  read; `walk` and `walk_depth` list a tree; `filter` and `filter_with`
  are `fnmatch.filter` over a list of names.

### Decided

- **`*` does not cross a path separator** in `matches`, following
  `glob` rather than `fnmatch`. `matches_name` is the other reading.
  The README leads with this and `tests/path_tests.nv` asserts it.
- **`[^abc]` negates**, where CPython's `fnmatch` reads `^` as an
  ordinary member. The shell's and the C library's reading.
- **`\` escapes the next byte**, where `fnmatch` has no escape operator
  at all. `[*]` still works.
- **One trailing separator is dropped** from both pattern and
  candidate, so `a/b` and `a/b/` are the same path.
- A `**` that is not a whole path segment is READ as a single `*` and
  REFUSED by `check`, where Rust's `glob` refuses to build the pattern
  at all.

### Known

- `src/glob.nv` does not cross-compile for `--target=nrf52-qemu`: no
  embedded-admitted call reads one byte of a `Str`. Filed against
  novo-lang; nothing else stands between the matcher and a device, and
  the umbrella section pins that exact shape.
- `tests/walk_tests.nv` passes every assertion and exits 23 on a
  reference-count residual in the toolchain, not the package: two
  nested `fs.read_dir` loops leak one list per nested level. Filed
  against novo-lang, with a reproducer naming no glob-nv code.

[0.1.0]: https://github.com/novolang/glob-nv/releases/tag/v0.1.0
