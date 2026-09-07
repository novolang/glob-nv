# bugs

Known defects in this package, one Markdown file each.

```bash
novo bugs new a-defect-you-found
novo bugs list --status open
novo bugs close a-defect-you-found --as fixed --fixed-in 0.2.2
novo bugs audit          # what CI runs on every push
```

`novo bugs new` writes the file from the toolchain's own template, with
the package name and version filled in from `novo.toml`, so the shape is
the same here as in every other novo-lang repository — and
`novo bugs audit` is what keeps it that way.

A filing is a file so that it clones with the code: whoever has the
package has the known defects, offline, in `grep`, and the commit that
fixes one can close it in the same commit.

## What strangers filed, and what you decided

```bash
novo bugs pull                     # -> bugs/inbox/, quarantine
novo bugs list --inbox             # what is waiting to be read
novo bugs accept <slug>            # read it, then promote it into bugs/
novo bugs sync                     # tell the reporters what you decided
```

Somebody who does not have this repository files at
`https://novo-lang.org/v1/bugs` naming this package. `novo bugs pull`
fetches those into **`bugs/inbox/`** — nothing there counts in
`novo bugs list` or in the audit's totals, because it is text you have
not read yet. `novo bugs accept` is what moves one out, and that is the
record that you read it.

`novo bugs sync` goes the other way: it writes the status of every filing
carrying a `tracker:` slug back, so the person who reported the defect
learns it is fixed and which release to upgrade to. The credential is
this package's **publish token** — the one `novo pkg publish` uses; there
is no second one to ask for. When both sides changed, your repository
wins for the status and the release, and the tracker keeps the
reporter's own words.

[Bugs in a repository](https://novo-lang.org/docs/bugs.html) is the
whole schema — the fields, the lifecycle, and what is deliberately not a
field. [Report a bug](https://novo-lang.org/docs/report.html) is how
somebody without this repository tells you about one.
