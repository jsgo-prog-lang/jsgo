# JSGo Package Manager

Status: **implemented**. `jsgo get` and
`jsgo update` resolve GitHub-hosted packages, fetch them into a
content-addressed module cache, and pin exact versions with SHA-256
digests in `jsgo.lock`. Builds never fetch: `check`/`build`/`run`/
`test` only read the cache (`SECURITY.md`).

## What the specification requires (§12)

```bash
jsgo init myapp     # implemented (project scaffold)
jsgo get pkg-name   # implemented — `jsgo get github.com/owner/repo[@constraint]`
jsgo update         # implemented — re-resolve within jsgo.mod constraints
```

- **Common applications must build with zero downloads.** The standard
  library is vendored inside the compiler; only third-party packages
  are fetched.
- Lockfiles (`jsgo.lock`) pin exact resolved versions for
  reproducible builds.
- Version management follows semantic versioning constraints
  (`^1.2`, `>=1.0 <2.0`).

## The manifest: `jsgo.mod`

`jsgo init` writes a project manifest:

```toml
module = "my-app"

language = "0.1"

[build]
entry = "src/main.jsgo"
target = "native"
```

Current semantics:

- `module` names the project; the built executable is named after its
  last path segment.
- `language` records the language version the project targets.
- `[build] entry` is written by `jsgo init` but is **informational in
  v0.1 — nothing reads it**. The actual no-argument input
  selection for `jsgo build`/`run`/`check` is: the file argument, if
  given → every `.jsgo` file under the project root (hidden,
  `node_modules`, `tests`, and `packages` directories and
  `*_test.jsgo` files are skipped) → `src/main.jsgo`, if present →
  the only `*.jsgo` file in the current directory → an error
  ("no .jsgo files found").
- `[build] target` is informational in v0.1 (native only).

Dependencies live in the manifest and are edited **surgically** by
`jsgo get` — comments, unknown sections, and the file's byte layout
survive:

```toml
[dependencies]
github.com/owner/logger = "^1.2"
```

An explicit `@constraint` on the command line is recorded verbatim;
an omitted constraint is recorded as `^<resolved>` (caret), so future
patch/minor updates stay possible. Resolution is **flat and
single-version** across the transitive graph; when two requirers
disagree and their constraints do not intersect, `jsgo get` fails at
exit 1 naming both requirers and the conflicting versions.

## `jsgo.lock`

Written only when its bytes change (a warm `jsgo get` prints
"already up to date"):

```toml
[package "github.com/owner/logger"]
version = "1.2.0"
url = "https://github.com/owner/logger/archive/refs/tags/v1.2.0.tar.gz"
sha256 = "…"
```

- Entries are sorted (stable, diff-friendly lockfiles).
- The SHA-256 pins the archive: a tampered download is refused
  before extraction (integration-tested).
- No-argument `jsgo get` installs exactly what the lock pins and is
  the only fully offline path when the module cache is warm; any
  actual resolution re-reads the tags API even on a warm cache.

## Implemented design

1. `jsgo get github.com/owner/logger@^1.0` resolves the version
   through GitHub's tags API (`Link`-paginated, `http.Client`
   check-and-redirect policy), downloads the source archive into the
   module cache, and records the dependency in `jsgo.mod` plus the
   exact version, URL, and checksum in `jsgo.lock`.
2. Imports (`import "github.com/owner/logger"` — plain namespace,
   `{ a as b } from` named, `* as ns from` star) resolve through the
   module cache via the checker's module-resolution step — the single
   integration point, as planned. A package that is not installed
   fails with `E0106` and the exact `jsgo get` command to run.
3. `jsgo update` re-resolves within `jsgo.mod` constraints: no
   arguments drop every pin and re-resolve the whole graph; named
   module paths drop only those pins, keeping the rest. Every move —
   or "already up to date" — is reported; a named module that is not
   a dependency exits 1.
4. Verification: checksums in `jsgo.lock`, HTTPS-only fetching
   (plain `http://` refused except loopback, redirects re-checked),
   zip-slip/symlink/size guards during extraction, no dependency code
   ever executed — see `SECURITY.md`.

## Non-goals (v1)

- No central registry requirement (modules are fetched from their own
  VCS/HTTP endpoints).
- No build scripts from dependencies (dependencies are JSGo source
  only; C dependencies are a future, explicitly-scoped feature).
