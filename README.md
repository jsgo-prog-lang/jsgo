# JSGo

**JavaScript syntax. Go power. Strict types. Native performance.**

JSGo is a statically typed programming language. You write `.jsgo`
files, the `jsgo` compiler type-checks them strictly and compiles them
to a native executable. Compiled programs depend only on the small
embedded JSGo runtime — never on Node.js, Python, or the Go runtime.

This distribution contains everything you need to install JSGo and
start writing programs.

**Project home:** [github.com/jsgo-prog-lang/jsgo](https://github.com/jsgo-prog-lang/jsgo)
— source, issues, release binaries and the official documentation.

## Download

Pick your platform. Each installer puts `jsgo` on your `PATH` and
carries the nine official documents with it.

| Platform | Download |
| --- | --- |
| Windows (x86-64) | [⬇ **MSI installer**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo-0.2.1-windows-x64.msi) |
| macOS (Apple silicon) | [⬇ **.pkg installer**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo-0.2.1-darwin-arm64.pkg) |
| macOS (Intel) | [⬇ **.pkg installer**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo-0.2.1-darwin-amd64.pkg) |
| Linux (x86-64) | [⬇ **.deb package**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo_0.2.1_amd64.deb) |
| Linux (aarch64) | [⬇ **.deb package**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo_0.2.1_arm64.deb) |

## Contents

```
docs/        the official language and tooling documentation
examples/    runnable example programs (.jsgo), grouped by theme
installers/  native installers: Windows MSI, macOS .pkg, Linux .deb
LICENSE      MIT
```

## Install

### Windows

Open `installers/jsgo-0.2.1-windows-x64.msi`. It installs
`jsgo.exe` under `Program Files\JSGo` and adds that directory to your
`PATH`, so a new shell picks it up immediately.

**SmartScreen.** The installer is not code-signed yet, so Microsoft
Defender SmartScreen shows *"prevented an unrecognized app from
starting"* the first time you open it. That is a reputation check —
no signature and no download history — not a malware finding about
this file. Choose **More info → Run anyway**. To clear the mark from
PowerShell first:

```powershell
Unblock-File .\jsgo-0.2.1-windows-x64.msi
```

On Windows 11 with **Smart App Control** enabled, unsigned files are
blocked outright rather than warned about; Smart App Control cannot
be re-enabled once turned off, so disable it before installing.

### macOS

Open `installers/jsgo-0.2.1-darwin-arm64.pkg` on Apple silicon or
`jsgo-0.2.1-darwin-amd64.pkg` on Intel. It installs `jsgo` to
`/usr/local/bin` and the documentation to
`/usr/local/share/doc/jsgo/`.

The package is **not signed or notarised** (the project does not yet
hold an Apple Developer ID), so macOS Gatekeeper cannot verify it and
shows *"Apple could not verify … is free of malware"* the first time
you open it. That is a reputation check on an unsigned download, not a
malware finding about this file. Three ways to install anyway:

1. **Right-click → Open.** Right-click (or Control-click) the `.pkg`
   in Finder and choose **Open**, then **Open** again in the dialog.
   This works for the plain Gatekeeper warning. If macOS shows the
   stricter *"could not verify"* dialog instead, use (2) or (3).

2. **Remove the quarantine flag, then double-click.** In a terminal:
   ```sh
   xattr -d com.apple.quarantine jsgo-0.2.1-darwin-arm64.pkg
   ```
   (use the `darwin-amd64` name on Intel). This clears the download
   mark; Gatekeeper then treats the file like a locally built one.
   macOS 15+ also stamps a SIP-protected `com.apple.provenance`
   attribute that cannot be removed, but clearing quarantine is
   enough for the installer to run.

3. **Install from the terminal (bypasses Gatekeeper entirely).**
   ```sh
   sudo installer -pkg jsgo-0.2.1-darwin-arm64.pkg -target /
   ```
   This is the most reliable route and is the one to use in scripts
   and CI. It does not prompt for confirmation.

After any of the three, `jsgo` is on your `PATH` in a new shell.

### Linux

Install the package for your architecture with your package manager:

```sh
sudo dpkg -i installers/jsgo_0.2.1_amd64.deb   # x86-64
sudo dpkg -i installers/jsgo_0.2.1_arm64.deb   # aarch64
```

It installs `jsgo` to `/usr/bin/jsgo` and the documentation to
`/usr/share/doc/jsgo/`.

## Quick start

```sh
jsgo init my-app     # scaffold a new project
cd my-app
jsgo run             # compile and run
jsgo build           # produce a native executable
```

Or work with a single file:

```sh
jsgo run hello.jsgo
```

```
Hello, JSGo!
```

### Hello, JSGo

```jsgo
function main() {
    const message: string = "Hello, JSGo!";
    println(message);   // builtin: no import needed
}
```

`print(value)` writes without a trailing newline and `println(value)`
adds one. Both are built into the language — no import required.

### Commands

```
jsgo init [dir]        create a new project
jsgo check [file]      parse and type-check without producing an executable
jsgo watch [file]      re-check automatically whenever inputs change
jsgo build [file]      compile to a native executable
jsgo run [file] [args] compile and run
jsgo test [file]       compile and run the project's tests
jsgo fmt [files...]    rewrite files in the house style
jsgo lint [file]       run the soundness lints
jsgo doc [files...]    write a Markdown API reference
jsgo lsp               serve the Language Server Protocol over stdio
jsgo get [pkg...]      fetch packages
jsgo update [pkg...]   re-resolve dependencies
```

Run `jsgo <command> -h` for the options of any command.

## Why JSGo?

### Why should I learn it?

If you already write JavaScript or TypeScript, you already know most
of JSGo's syntax. Functions, `let`/`const`, `class`, `extends`,
`async`/`await`, arrow functions and `for` loops all mean what they
mean there. What JSGo adds back is a type checker that refuses to let
a whole class of bugs reach runtime.

```js
// JavaScript: no complaint now, "23" at runtime
function add(a, b) { return a + b; }
console.log(add("2", 3));   // "23"
```

```jsgo
// JSGo: caught before the program runs
function add(a: int32, b: int32): int32 {
    return a + b;
}

function main() {
    println(add("2", 3));
}
```

```
error[E0201]: argument 1 of 'add': cannot use string as int32
```

There is a second reason to look: JSGo is a small, readable compiler.
It is written in Go, parses your file into a syntax tree, type-checks
it, lowers it to a typed IR, and emits C — the whole pipeline is
documented in [`docs/LANGUAGE_SPEC.md`](docs/LANGUAGE_SPEC.md) if you
have ever wanted to see how a language really works.

### How powerful is it?

JSGo compiles to C and links against a small embedded runtime, so a
JSGo program is a **native executable**. No Node.js, no JVM, no Go
runtime and no interpreter on the target machine — just a binary.

What that gets you:

- **Real concurrency.** `async`/`await` tasks, `Channel<T>`,
  `select`, and `go worker(arg)` run on one cooperative scheduler
  where `time.sleep` and socket waits park tasks instead of blocking
  the program.
- **Generics, closures, classes and interfaces** — unconstrained
  generics (`Box<T>`), arrow functions that capture locals, single
  inheritance with virtual dispatch, and structural interfaces.
- **Pointers with a safety net.** `*T`, `&x` and `*p` exist for when
  you need them; address-taken locals are boxed so an escaping
  pointer never dangles, and a conservative mark–sweep collector
  reclaims memory.
- **Cross-compilation** for Linux and macOS on x86-64 and arm64.
- **One toolchain.** `jsgo build`, `run`, `test`, `fmt`, `lint`,
  `doc`, `watch` and `lsp` all come from a single binary.

```jsgo
async function square(n: int32): int32 {
    return n * n;
}

async function report(): int32 {
    let t: Task<int32> = square(7);   // starts eagerly
    let v: int32 = await t;           // suspends until it finishes
    println(v);                       // 49
    return v;
}

function main() {
    report();
}
```

### What kinds of apps can I build?

| Kind of app | What you would use |
| --- | --- |
| Command-line tools and scripts | `os`, `filesystem`, `path`, `strconv` |
| REST APIs and web servers | `http.serve` + `json` (see [`examples/server/main.jsgo`](examples/server/main.jsgo)) |
| Socket-level network services | `net` (`listen`/`accept`/`read`/`write`) |
| Database-backed services | `database` (five `Result`-returning calls) |
| Concurrent workers and pipelines | `async`/`await`, `Channel<T>`, `select` |
| Tested, documented libraries | `testing`, `jsgo test`, `jsgo doc` |

A whole HTTP server is this short:

```jsgo
import "http";

function greet(req: Request): Response {
    return Response(200, {"Content-Type": "text/plain"},
        "you requested " + req.path);
}

function main() {
    let r = http.serve("127.0.0.1:8080", greet);
    if (!r.isOk()) {
        println("serve failed: " + r.error());
    }
}
```

[`examples/server/main.jsgo`](examples/server/main.jsgo) is the
fuller version — a TODO service with full CRUD, JSON bodies and an
embedded row store.

JSGo targets the command line and the server. Browser apps, mobile,
GUI and games are out of scope for now.

### What are the benefits?

- **Familiar syntax, strict types.** You keep the shape of the code
  you already write; you lose the runtime surprises.
- **Errors that read like feedback, not puzzles.** Every diagnostic
  has a stable code, a source location, and usually a `help:` line
  telling you what to write instead.
- **Nothing to install on the target machine.** Ship one binary; it
  does not need a language runtime on the other side.
- **Reproducible dependencies.** `jsgo.lock` pins the exact version
  *and* the SHA-256 of every package, so a cold install verifies what
  a warm install used.
- **The boring parts are built in.** A formatter (`jsgo fmt`), a
  soundness linter (`jsgo lint`), a documentation generator
  (`jsgo doc`), a language server (`jsgo lsp`) and a package manager
  (`jsgo get`) are part of the toolchain, not a shopping list.
- **A 17-module standard library** (112 checked functions) covering
  strings, math, time, encoding, crypto, JSON, HTTP, databases and
  testing — see [`docs/STANDARD_LIBRARY.md`](docs/STANDARD_LIBRARY.md).

### What are the advantages and disadvantages?

**Advantages**

- JavaScript/TS syntax with compile-time types — a low barrier in,
  a high safety bar once you are there.
- Native output with a garbage collector, so you get performance
  without manual memory management.
- Concurrency primitives (`Channel<T>`, `select`, `go`) that are
  syntax, not a library afterthought.
- One binary for the whole developer experience: build, test, format,
  lint, document, and an editor language server.

**Trade-offs — read these before betting a project on it**

- **v0.2 is experimental.** It works and the test suite is green, but
  it is pre-1.0: expect breaking changes and do not ship it as a
  dependency of anything critical yet.
- **Not everything from JavaScript is here.** Template strings and
  `regex` are reserved with clear `E0020` diagnostics. Object
  literals, destructuring, spread/rest, `for-of`/`for-in` and
  `try/catch` are deliberately deferred — each raises a diagnostic
  rather than silently compiling.
- **No auto-dereference.** Through a pointer you write
  `(*p).field`, not `p.field`.
- **Single inheritance only**, and structs use positional
  constructors (`Point(1.0, 2.0)`), not object-literal syntax.
- **You need a C toolchain to build.** The compiler emits C and
  hands it to `cc`.
- **The ecosystem is young.** `jsgo get` is new and there are few
  third-party packages compared with npm, PyPI or Go modules.
- **The collector is conservative**, so it trades some throughput
  for never reclaiming live memory.

### How does it compare to other languages?

| | JavaScript | TypeScript | Go | Rust | Python | **JSGo** |
| --- | --- | --- | --- | --- | --- | --- |
| Type checking | dynamic | static (erased at runtime) | static | static | dynamic | **static** |
| What you ship | source | source (JS) | native binary | native binary | source | **native binary** |
| Syntax | — | JS + types | C-like | its own | English-like | **JS/TS-like** |
| Concurrency | event loop | event loop | goroutines | async + ownership | threads/async | **tasks + channels** |
| Memory | GC | GC | GC | ownership, no GC | GC | **GC (conservative)** |
| Needs to build | nothing | nothing | Go toolchain | Rust toolchain | nothing | **a C compiler** |
| Maturity | huge | huge | large | growing | huge | **v0.2 experimental** |

The short version: **TypeScript's syntax, Go's compile-to-native
model, and a standard library that assumes you are writing a server
or a tool.** Choose JavaScript when you must run in a browser,
Python when you want the largest ecosystem and the softest ramp,
Go or Rust when you need production-grade tooling today. Choose JSGo
when you want native output with syntax you already know, and you
are happy being an early adopter.

### What real-world problems can it solve today?

Given where the project is, the honest answer is *prototypes,
internal tools and learning projects*:

- **Internal CLI tools** — a script that grew past "just a shell
  script", where a single distributable binary is the whole point.
- **REST microservices and side projects** — `http.serve` plus
  `json` is enough for a real API, as
  [`examples/server/main.jsgo`](examples/server/main.jsgo)
  demonstrates end to end.
- **Batch and data-processing jobs** — parse, transform and write
  files with `strconv`, `encoding` and `filesystem`, with `Result`
  instead of exceptions.
- **Concurrent workers** — a pool of tasks feeding a `Channel<T>`,
  with `select` whichever source is ready first.
- **Teaching and experimentation** — a language small enough to
  read the whole compiler for, which is rare among modern languages.

It is not yet the right choice for a safety-critical system, a
long-lived public API, or anything that must not change under you.

### Career opportunities and future scope?

Be direct about this: **JSGo is not a hiring market today.** Nobody
is listing "3 years of JSGo" in a job ad. What it *is* good for:

- **Transferable skills.** Static typing, generics, interfaces,
  concurrency primitives, memory models and how a compiler pipeline
  works are the same concepts asked about in TypeScript, Go and Rust
  interviews. Learning them in a small language makes them easier to
  see.
- **A credible portfolio piece.** "I read and patched a compiler"
  is a stronger story than another to-do app.
- **Early-contributor leverage.** A project at v0.2 has far more
  room for meaningful contribution than a mature one does — the
  roadmap still has debugger integration and benchmarking to land,
  and a future backend would not touch the front end.

**Where the project is heading:** the developer-ecosystem wave has
already shipped the formatter, linter, documentation generator,
package manager and language server. Debugger integration and
benchmarking remain, and the pipeline keeps the front end, type
checker and backend independent so a future LLVM backend would not
touch the front end.

If you want to watch a language go from "it compiles" to "people
build on it", this is an unusually legible moment to join.

## Documentation

| Document | What it covers |
| --- | --- |
| [`docs/LANGUAGE_SPEC.md`](docs/LANGUAGE_SPEC.md) | The complete language: syntax, semantics, statements, expressions, errors |
| [`docs/SYNTAX.md`](docs/SYNTAX.md) | A syntax reference with grammar productions |
| [`docs/TYPE_SYSTEM.md`](docs/TYPE_SYSTEM.md) | Types, inference, conversions, interfaces, subtyping |
| [`docs/MEMORY_MODEL.md`](docs/MEMORY_MODEL.md) | Values, references, copying, and lifetimes |
| [`docs/CONCURRENCY_MODEL.md`](docs/CONCURRENCY_MODEL.md) | Tasks, channels, `select`, and synchronization |
| [`docs/STANDARD_LIBRARY.md`](docs/STANDARD_LIBRARY.md) | The standard library: every module and function |
| [`docs/PACKAGE_MANAGER.md`](docs/PACKAGE_MANAGER.md) | Creating, installing, and publishing packages |
| [`docs/EDITOR_SETUP.md`](docs/EDITOR_SETUP.md) | Editor integration, diagnostics, and formatting |
| [`docs/SECURITY.md`](docs/SECURITY.md) | The security model and reporting process |

Start with [`docs/LANGUAGE_SPEC.md`](docs/LANGUAGE_SPEC.md) if you are
new to JSGo.

## Examples

The `examples/` directory holds runnable programs grouped by theme,
so you can read one idea at a time:

| Directory | Example | Demonstrates |
| --- | --- | --- |
| `basics/` | `hello.jsgo` | A first program and `println` |
| `basics/` | `greet.jsgo` | String building, branching, comparison |
| `control-flow/` | `fib.jsgo` | Recursion, loops, typed functions |
| `control-flow/` | `grade.jsgo` | `switch`: constant cases, multiple values |
| `functions/` | `pipeline.jsgo` | Arrow functions as values: store, pass, compose |
| `types/` | `pets.jsgo` | Classes, inheritance, and interfaces |
| `types/` | `shapes.jsgo` | Structs, value and pointer receivers |
| `collections/` | `wordcount.jsgo` | Counting frequencies with strings, arrays, maps |
| `errors/` | `temperature.jsgo` | Parsing text into numbers, `Result`-based failure |
| `errors/` | `cat.jsgo` | Reading a file, with `Result` instead of exceptions |
| `concurrency/` | `channels.jsgo` | `go` tasks, `Channel<T>`, and `select` |
| `server/` | `main.jsgo` | A JSON REST API with full CRUD over HTTP |

Run any of them:

```sh
jsgo run examples/basics/hello.jsgo
jsgo run examples/control-flow/fib.jsgo
```

`errors/cat.jsgo` copies a file named on its command line, so it
takes an argument:

```sh
jsgo run examples/errors/cat.jsgo some-file.txt
```

`server/main.jsgo` is a server, so it keeps running instead of
exiting — stop it with Ctrl+C. It listens on port 8080 and prints
the address when it is ready:

```sh
jsgo run examples/server/main.jsgo
```

Every example in this directory is checked against the compiler it
ships with as part of the release gate, so the version you
downloaded is the version they were checked against.

## Editor support

JSGo ships a language server (`jsgo lsp`) that provides diagnostics,
completion, hover, go-to-definition, and formatting over the standard
Language Server Protocol. See
[`docs/EDITOR_SETUP.md`](docs/EDITOR_SETUP.md) to connect your editor.

## License

MIT — see [`LICENSE`](LICENSE).
