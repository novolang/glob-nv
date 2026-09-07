# glob-nv

Glob patterns: `?`, `*`, `**`, character classes and ranges, matched
against a path without allocating — and a walker that expands one
against a real directory tree.

```novo
use glob
use walk

fn main() [fs, io]
    println(str.from_bool(glob.matches("src/**/*.nv", "src/deep/main.nv")))  // true
    println(str.from_bool(glob.matches("src/*.nv", "src/deep/main.nv")))     // false

    for path in walk.expand("orbit/glob-nv", "src/*.nv")
        println(path)                       // src/glob.nv, src/walk.nv
```

```
novo pkg add glob-nv
```

No dependencies. The matcher is a hand-written scan over bytes; there
is no regular expression behind it and there could not be — a pattern
compiled into a regex is a second language to get the corners of `[]a]`
and `[a-]` wrong in, and the translation costs an allocation that this
package's whole point is not to make.

## `*` does not cross a path separator

This is the one place `fnmatch` and `glob` disagree with each other, and
every library has to pick a side. In Python:

```python
fnmatch.fnmatch("a/b", "*")   # True
glob.glob("*")                # never returns "a/b"
```

**glob-nv follows `glob`.** In `glob.matches`, a `*`, a `?` and a
character class each stop at `/`, and `**` — written as a whole path
segment — is the operator that crosses them. That is the reading a
build tool needs: `src/*.nv` is meant to name the sources in one
directory, and a `*` that swallowed `/` would make it match
`src/a/b/c.nv` as well.

The other reading is one call away, because a caller filtering bare
file names wants it:

```novo
glob.matches("*", "a/b")        // false — the path reading
glob.matches_name("*", "a/b")   // true  — fnmatch's reading
```

`matches_name` treats the whole candidate as ONE name, so there `*`,
`?` and a class match every byte including `/`, and `**` is just two
`*`s. Rust's `glob` crate spells the same choice
`MatchOptions::require_literal_separator`, and defaults it the other
way; here it is not an option but two named calls, so a reader of the
call site can see which question was asked.

## Matching costs nothing

`glob.matches` walks the caller's two strings with four `Int` cursors
in a `@value` frame. No pattern is compiled, nothing is copied, and the
call allocates zero heap cells:

```novo
use glob

let o = glob.shell_options()
glob.matches("src/**/*.nv", "src/deep/main.nv")     // true
glob.matches_with("*.nv", ".hidden.nv", o)          // false — the shell's dot rule
glob.may_descend("src/**/*.nv", "docs")             // false — do not open it
```

That is a claim a test keeps rather than a paragraph: the suite builds
a program that matches, extracts every `glob` function from the emitted
LLVM at `--opt=0`, and fails if any of them contains a call to
`novo_alloc`. Thirty-eight functions, zero allocations, on every run.

The other half of the claim is the module split. `glob` never
allocates. `walk` is where lists and paths come from — `expand`,
`walk`, `filter` — and every call in it allocates; the ones that read
directories carry `[fs]` and every function in it carries
`@tier(app)`. A program with no heap compiles `glob` and stops there.

Novo has no `[alloc]` effect to carry that line with: SPEC § 5.1's
effect vocabulary is exhaustive and `alloc` is `E3005`, because
`@tier(embedded)` rejects the allocating *constructs* directly and a
declaration could not be enforced. A module boundary is the version of
the same claim a build either crosses or does not.

## It does not backtrack exponentially

`a*a*a*a*a*a*a*a*b` against a candidate of four thousand `a`s is the
pattern that kills a matcher which recurses on every `*`: each star
tries every split, the failure is only found after the last byte, and
the cost is exponential in the number of stars. It is also the pattern
somebody eventually types.

The matcher here remembers the last `*` it walked past and RESUMES from
there rather than returning to a caller who would try the next split.
`**` gets the same treatment one level up, with a path segment playing
the part of a byte. The cost is O(pattern × candidate), and for a fixed
pattern linear in the candidate — measured, in `tests/orbit/test_glob_nv.sh`:

| candidate | 500 | 1000 | 2000 | 4000 |
|---|---|---|---|---|
| µs per match | 11 | 23 | 45 | 90 |

| path depth | 100 | 200 | 400 | 800 |
|---|---|---|---|---|
| µs per match | 6 | 12 | 24 | 49 |

The suite asserts the ratio between consecutive doublings stays under
6, and that every timing loop got the right answer — a matcher that got
fast by answering the wrong question is still a red run.

## Two modules

| Module | What it is for | Allocates |
|---|---|---|
| `glob` | match a pattern against a path or a name; validate one and hear why; ask whether a directory is worth opening; the two helpers a walk is planned with | **no** |
| `walk` | expand a pattern against a directory tree in a stable order, walk a tree, filter a list of names | yes; `[fs]` where it reads directories |

Every function, its signature, its effect row and a worked example of
each entry point are on
[the package's page](https://novo-lang.org/packages/glob-nv), generated
from these sources at every publish. A table of names here would be a
second original, and the second original is the one that goes stale.

## Checking a pattern is a separate call

`matches` returns a `Bool`, which has no room for "your pattern does not
mean what you think". So `check` is where that lives, and the point of
splitting them is the shape of the work: ask once, then match ten
thousand paths knowing what the pattern means.

```novo
use glob

let c = glob.check("src/**c/x")
c.is_valid()          // false
c.offset()            // 4 — where the finding is
glob.reason(c)        // RecursiveNotWhole
```

Every finding names a pattern `matches` still answers for, by a
degraded reading that is written down and tested:

| Finding | The pattern | What `matches` does with it |
|---|---|---|
| `UnterminatedClass` | `[abc`, `[]`, `[!]` | a literal `[` — `fnmatch`'s answer |
| `BadRange` | `[z-a]` | matches nothing |
| `DanglingEscape` | `a\` | a literal backslash |
| `RecursiveNotWhole` | `a**b`, `**c`, `***` | one `*`, which stays inside its segment |

Rust's `glob` refuses to BUILD a pattern with a `**` that is not a whole
segment (`Pattern::new` answers `InvalidRecursive`); `check` is how you
get that behaviour here, and `matches` still answers so that a caller
who never asked is not left with an exception in a loop.

## Where it deliberately differs from `fnmatch`

Fifty-four of CPython's fifty-eight `fnmatch` cases in
`tests/fnmatch_tests.nv` agree byte for byte. The four that do not are
two decisions, each of them a named test with its reason:

- **`[^abc]` negates.** CPython translates a class straight into a
  regular expression's own, where `^` is only special at the front — so
  `[^a]` there is the two members `^` and `a`, and `[!a]` is the only
  way to write a negated class. Here `^` negates exactly as `!` does,
  which is what the shell and the C library do and what somebody
  writing `[^a]` almost always meant. The cost: a caret can no longer
  be the FIRST member of a class by writing itself. `[a^]` and `[!^]`
  are unambiguous and both readings agree on them.
- **`\` escapes.** `fnmatch` has no escape operator at all — its
  `translate` escapes the backslash itself, so `fnmatchcase("*", "\\*")`
  is `False` in Python and a caller with a real `*` in a file name has
  to write `[*]`. glob-nv takes the shell's reading: `\` makes the next
  byte literal. `[*]` still works, so Python's spelling is not broken.

Everything else `fnmatch` decided, this package decided the same way,
including the parts that read like accidents: a `]` written first in a
class is a member, `-` first or last is a member, an unterminated `[`
matches itself, and `\` inside a class is a member rather than an
escape.

## Hidden files, and the other options

`Options` has two fields and there are two named constructors:

```novo
glob.options()          // case-sensitive; a `*` reaches `.hidden`
glob.shell_options()    // case-sensitive; a leading `.` must be written
glob.ignore_case(o)     // the same, with ASCII case folding on
```

`options()` is Rust `glob`'s default (`require_literal_leading_dot:
false`); `shell_options()` is Python `glob`'s and every POSIX shell's.
With the shell rule on, `**` also refuses to descend into a segment
beginning with `.`, which is what keeps a walk out of `.git`.

Case folding is ASCII and nothing above 127 is touched — that needs a
Unicode table, and this package carries none. A negated class asks the
membership question for both cases of the byte and negates the answer
ONCE, so `[!a-c]` does not match `B`; getting that backwards is the
easy bug here and it has its own test.

## The walker

```novo
use walk

walk.expand(root, "src/**/*.nv")        // paths relative to `root`
walk.expand_with(root, pattern, o)      // with the shell's dot rule, say
walk.expand_depth(root, pattern, o, 3)  // and a depth bound
walk.walk(root)                         // every path under `root`
walk.filter(names, "*.nv")              // fnmatch.filter, over a list
```

Three things it promises.

**Relative paths in and out.** The pattern is matched against the path
relative to `root` — `src/main.nv`, not `/home/you/proj/src/main.nv` —
and the answers come back the same way, so two checkouts at different
paths produce the same list.

**A stable order.** Entries are sorted at every directory level and
visited depth-first, parent before child. `fs.read_dir` promises no
order of its own, so this one is the walker's and is asserted against a
built tree.

**It does not die on a tree it cannot fully read.** A directory whose
read bit is off contributes no entries and stops nothing — it is still
listed, because it is there, and the walk carries on past it.

And one bound worth knowing about: `fs.is_dir` follows symbolic links,
so a link pointing at its own ancestor is a cycle. The walkers descend
at most 64 levels by default, and `walk_depth` / `expand_depth` take
the bound as a parameter.

A pattern segment with no operator in it names one directory entry, so
`walk` never opens a directory that no path matching the pattern could
pass through — that is `glob.may_descend`, which shares the matching
machine rather than reimplementing it, and it is why expanding
`src/**/*.nv` over a repository does not read `target/`.

## What it does not do

- **No brace expansion.** `{a,b}.nv` is two patterns, not one, and
  expanding it means allocating the alternatives. A caller who wants it
  writes the loop; a future package can own it.
- **No `[[:alpha:]]` classes.** POSIX character classes need a locale
  or a Unicode table. The byte ranges are here and they are ASCII.
- **No `~` expansion and no path normalisation.** `~`, `.` and `..` are
  ordinary bytes in a segment. Expanding a home directory reads the
  environment, which the matcher does not do and the walker does not
  guess at.
- **Bytes, not characters.** `?` matches one BYTE, so it matches one
  ASCII character and one third of a three-byte UTF-8 one. Making it a
  character means decoding, and decoding allocates — which the embedded
  tier does not admit.
- **It is not `fs.glob`.** The standard library's `fs.glob` is the C
  library's `glob(3)`: it expands against the real filesystem, supports
  `~`, has no `**`, and gives you no way to ask about a path you are
  holding. This package answers the matching question separately from
  the filesystem one, which is what a build tool, a `.gitignore` reader
  and a command line's file arguments all actually need.

Each of these is a test rather than a sentence, in
`tests/path_tests.nv` and `tests/check_tests.nv`, with the answer the
package actually gives.

## The device build

`glob` is written to the embedded tier's rules: nothing in it allocates,
every loop is a counted `for` over a bound computed before it starts
(SPEC § 12.4), and the state of a match is four `Int`s in a frame.

It cross-compiles for `--target=nrf52-qemu` and the umbrella boots the
image: `tests/embedded_probe.nv` matches `sensors/**/temp` against
`sensors/floor2/room7/temp` on a Cortex-M and writes its verdict to a
UART.

It did not, until novo-lang 0.8.6, and the reason was not this package:
there was no embedded-admitted call that read one byte of a `Str`.
`str.byte_at_or(s, i, default)` is that call now, and `byte_of` is
written with it. The `walk` module is still host-only and still not
part of the device build — it knows about lists and the filesystem —
which is what the module split is for.

## Tests

```bash
novo test tests                     # every module under tests/ — 41 tests
novo test tests/fnmatch_tests.nv    # 14 — CPython's cases, and the two differences
novo test tests/path_tests.nv       # 13 — the separator decision, `**`, hidden files
novo test tests/check_tests.nv      #  7 — every finding, and what it degrades to
novo test tests/walk_tests.nv       #  7 — the walker, over a tree it builds
novo test src/glob.nv               # the documented examples — 8
novo test src/walk.nv               # and 3 more
```

The `fnmatch` cases are CPython's, from `Lib/test/test_fnmatch.py`, and
the `**` cases are the ones Rust's `glob` crate carries in its own
tests. Both were **retyped** into this repository's own table shape —
one call per case — rather than the upstream files being vendored, so
nothing here carries anybody else's licence text. The expected answers
were then checked against what `fnmatch.fnmatchcase` on a real CPython
and `glob.glob` over a real directory tree actually return, rather than
against what the upstream files claim.

`tests/walk_tests.nv` passes every assertion and exits 23 on a
reference-count residual that is not this package's: two nested
`fs.read_dir` loops leak one list per nested level, which is a
toolchain defect with a thirteen-line reproducer naming no glob-nv code
and is filed against novo-lang. The package does not restructure around
it — that would cost the depth-first walk order — so the umbrella
section pins the residual at its exact count instead.

## Licence

Apache-2.0. See `LICENSE`.
