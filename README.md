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
| Windows (x86-64) | [⬇ **MSI installer**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo-0.2.0-windows-x64.msi) |
| macOS (Apple silicon) | [⬇ **.pkg installer**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo-0.2.0-darwin-arm64.pkg) |
| macOS (Intel) | [⬇ **.pkg installer**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo-0.2.0-darwin-amd64.pkg) |
| Linux (x86-64) | [⬇ **.deb package**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo_0.2.0_amd64.deb) |
| Linux (aarch64) | [⬇ **.deb package**](https://github.com/jsgo-prog-lang/jsgo/raw/main/installers/jsgo_0.2.0_arm64.deb) |

## Contents

```
docs/        the official language and tooling documentation
examples/    runnable example programs (.jsgo), grouped by theme
installers/  native installers: Windows MSI, macOS .pkg, Linux .deb
LICENSE      MIT
```

## Install

### Windows

Open `installers/jsgo-0.2.0-windows-x64.msi`. It installs
`jsgo.exe` under `Program Files\JSGo` and adds that directory to your
`PATH`, so a new shell picks it up immediately.

**SmartScreen.** The installer is not code-signed yet, so Microsoft
Defender SmartScreen shows *"prevented an unrecognized app from
starting"* the first time you open it. That is a reputation check —
no signature and no download history — not a malware finding about
this file. Choose **More info → Run anyway**. To clear the mark from
PowerShell first:

```powershell
Unblock-File .\jsgo-0.2.0-windows-x64.msi
```

On Windows 11 with **Smart App Control** enabled, unsigned files are
blocked outright rather than warned about; Smart App Control cannot
be re-enabled once turned off, so disable it before installing.

### macOS

Open `installers/jsgo-0.2.0-darwin-arm64.pkg` on Apple silicon or
`jsgo-0.2.0-darwin-amd64.pkg` on Intel. It installs `jsgo` to
`/usr/local/bin` and the documentation to
`/usr/local/share/doc/jsgo/`.

The package is not signed, so macOS asks you to confirm it the first
time: right-click the file and choose **Open**. From a terminal,
`sudo installer -pkg <file> -target /` installs it directly.

### Linux

Install the package for your architecture with your package manager:

```sh
sudo dpkg -i installers/jsgo_0.2.0_amd64.deb   # x86-64
sudo dpkg -i installers/jsgo_0.2.0_arm64.deb   # aarch64
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

Every example in this directory is run against the compiler it ships
with as part of the release gate, so the version you downloaded is
the version they were checked against.

## Editor support

JSGo ships a language server (`jsgo lsp`) that provides diagnostics,
completion, hover, go-to-definition, and formatting over the standard
Language Server Protocol. See
[`docs/EDITOR_SETUP.md`](docs/EDITOR_SETUP.md) to connect your editor.

## License

MIT — see [`LICENSE`](LICENSE).
