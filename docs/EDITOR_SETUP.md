# Editor setup — type errors as soon as you save

JSGo's type checker runs in batch (`jsgo check`/`build`/`run`), so
without one of the integrations below a type error is only reported
when you invoke a command yourself. There are four
ways to get automatic feedback, in increasing order of formality:

| Mode | What you get | Cost |
| --- | --- | --- |
| `jsgo watch` | Re-checks in your terminal whenever a `.jsgo` file changes | Nothing to install |
| Editor on-save task | Errors appear in the **Problems panel** and jump you to the line, on every save | One extension |
| Language server (`jsgo lsp`) | Diagnostics whenever the buffer changes — no save required — plus hover with signatures/doc comments and go-to-definition across the project | Built into the toolchain — see below |
| VS Code extension | Everything the language server gives, plus highlighting, snippets, run/build/check/lint/format commands — one install, no configuration | Marketplace, or `code --install-extension` — see below |

## Prerequisite: `jsgo` on your PATH

`jsgo watch`, the on-save task and a hand-rolled `jsgo lsp` all
invoke `jsgo` as a command, so the toolchain has to be on the PATH the
editor inherits. Verify with `jsgo version`. If it is missing:

Install JSGo with the installer for your platform (see the README);
the `jsgo` it places on your `$PATH` is all these modes need.

Any directory on your `$PATH` works: the scaffolded task appends
`~/.local/bin` to its own `PATH` (an `options.env` entry in
`.vscode/tasks.json`), so it finds `jsgo` even when the editor was
launched from the Dock.

**The VS Code extension is the exception** — it bundles the toolchain,
so installing it needs no prerequisite at all. The setting below lets
you point it at a different executable if you have one.

## Editor on-save (Cursor / VS Code)

`jsgo init` (and `jsgo new`) scaffold a `.vscode/` directory into every
new project with three files:

| File | Role |
| --- | --- |
| `tasks.json` | The `JSGo: check` task: runs `jsgo check` over the project and carries a problem matcher that parses `severity[code]: message` + `--> file:line:col` lines into Problems-panel entries. `revealProblems: onProblem` opens the panel only when something is wrong. Its `options.env.PATH` appends `~/.local/bin`, so the task works even when the editor was launched from the Dock. |
| `settings.json` | Wires the **Trigger Task on Save** extension to run that task whenever a `**/*.jsgo` file is saved (`restart` handles rapid consecutive saves). |
| `extensions.json` | Recommends `gruntfuggly.triggertaskonsave`, so the editor offers to install it when you open the project. |

Flow: **save** → extension fires → task runs `jsgo check` → problem
matcher parses the diagnostics → Problems panel + inline navigation.
No manual command.

For an **existing project**, copy the directory from a fresh scaffold:

```sh
jsgo init /tmp/jsgo-scaffold
cp -r /tmp/jsgo-scaffold/.vscode /path/to/your/project/
```

Open the folder that *contains* `.vscode` as your editor workspace —
folder-level settings and tasks are read from the opened workspace
root.

### Several programs in one workspace

A whole-project `jsgo check` walks every `.jsgo` file as **one**
program, so a folder holding several programs — each with its own
`main` — reports duplicate-`main` errors. This repository is exactly
that shape (`examples/*.jsgo` and `bin/*/src/*.jsgo` are separate
programs), so its root `.vscode/` task checks each program in turn
instead:

```sh
failed=0; for f in examples/*.jsgo bin/*/src/*.jsgo; do
  [ -e "$f" ] || continue
  jsgo check "$f" || failed=1
done; exit $failed
```

Diagnostics keep workspace-root-relative paths
(`bin/myapp/src/main.jsgo:3:1`), so problem-matcher entries resolve
to the right file. Scaffolded single-program projects do not need
this: their `jsgo check` walk stops at the `jsgo.mod` root.

## Terminal workflow: `jsgo watch`

```sh
jsgo watch              # whole project (same inputs as `jsgo check`)
jsgo watch src/main.jsgo  # one file plus its imports
```

An initial check runs immediately, then every 300 ms the watcher
fingerprints the inputs (path, mtime, size) and re-checks only when
something changed, printing a `--- hh:mm:ss re-checking ---` header
before each round. Stop it with Ctrl-C; it writes nothing to disk, so
it is always safe to kill.

## Language server: `jsgo lsp` (feedback as you edit)

Editors that speak the Language Server Protocol can run JSGo's own
server instead of a save task. Your editor starts it —
it speaks framed JSON-RPC over stdin/stdout and lives until the
editor sends `exit`:

```sh
jsgo lsp    # no arguments; stdout carries protocol frames only
```

| Capability | What you get |
| --- | --- |
| Diagnostics | Every buffer change re-checks the whole project with your open buffers overlaid — all files' problems are published, stale ones cleared, with the same codes, messages, and spans `jsgo check` prints |
| Hover | The declaration under the cursor as a fenced `jsgo` signature plus its doc comment; values show `name: type` from the checker |
| Go-to-definition | Jumps to the declaration — across files — filtered to the workspace |
| Completion | A bare word completes the enclosing scope: the enclosing function's locals with their **checked** types, the declarations the file can reach (its own plus its file imports — never a sibling's), the standard-module and import aliases, the builtin types, and the keywords the language actually writes. After a `.` you get the receiver's own and inherited fields and methods — interface members through an interface receiver, `static` members through the class name, never the constructor. Inside an `import "…"` you get the seventeen standard modules, or the workspace's own files when the spec starts `./`, inserted right after the quote |
| Find references | Every use of the declaration under the cursor, across files, with the declaration itself included unless the editor asks for uses only. A member resolves through the receiver's type, so a field or method has one answer rather than one per class that spells the name; dispatch is symmetric — a call written through an interface and a call written on a class name the same declarations |
| Formatting | The whole buffer formatted by the same normalizer `jsgo fmt` uses, as one edit — run from your editor's Format Document. It formats the **live buffer**, so unsaved work is never reverted, and a buffer that does not parse is left untouched |
| Positions | UTF-16 columns, what LSP clients expect |

Wire it up with your editor's generic LSP client, pointing it at the
`jsgo lsp` command with the project root as the workspace folder (the
server roots itself from the `rootUri` the editor sends). Neovim, for
example:

```lua
vim.lsp.start({
  name = 'jsgo',
  cmd = { 'jsgo', 'lsp' },
  root_dir = vim.fs.dirname(vim.fs.find({ 'jsgo.mod', '.git' }, { upward = true })[1]),
})
```

The server never writes to disk: diagnostics, completion, references,
and formatting all come from the checked project plus your open
buffers. If your editor sends no `rootUri`, start it
from the project root — gathering works from the working directory.

A half-typed member access (`let v = box.`) is the case most editors
get wrong; JSGo's server repairs it by checking a throwaway copy of
the buffer with a placeholder in it, so the members you are about to
pick are still offered while the line does not yet parse.

## The VS Code extension (shipped)

The Marketplace carries the extension as
[`jsgo.jsgo-lang`](https://marketplace.visualstudio.com/items?itemName=jsgo.jsgo-lang).
Install it from there, or from the command line:

```sh
code --install-extension jsgo.jsgo-lang
```

The extension's project home is
[github.com/jsgo-prog-lang/jsgo](https://github.com/jsgo-prog-lang/jsgo),
which also carries the installers and this documentation.

It is a workspace extension with **no dependencies of its own** — no
Node.js, no Go, no Python and nothing to `npm install` — and it
**bundles the JSGo toolchain**, so installing it needs no `PATH` edit
and no other prerequisite. Language features come from
`jsgo lsp` and run, build, check and lint from the `jsgo` executable;
both are the copies inside the extension, so a
`.jsgo` program runs on nothing but the extension itself.

| Feature | Behaviour |
| --- | --- |
| Highlighting, bracket matching, auto-closing | TextMate grammar plus language configuration for `.jsgo` |
| Snippets | `main`, `fun`, `class`, `interface`, `result`, `chan`, … |
| Diagnostics | The language server, on every edit, in the Problems panel |
| Hover | Signature and doc comment under the cursor |
| Go to Definition / Find References | Across the workspace, through the language server |
| Completion | The enclosing scope, members after `.`, module and workspace paths after `import "` |
| Format Document / Format On Save | The same normalizer `jsgo fmt` uses, always 4-space indent |
| Problem matcher | `$jsgo`, for tasks you write yourself |

### Commands

Open the Command Palette (`Cmd/Ctrl+Shift+P`) and type `JSGo:`:

| Command | What it does |
| --- | --- |
| `JSGo: Run File` | Saves the file, builds it into `.jsgo-build/`, runs it in a terminal |
| `JSGo: Build File` | Saves the file and compiles it into `.jsgo-build/` |
| `JSGo: Check Project` | Runs `jsgo check` over the project and fills the Problems panel |
| `JSGo: Lint Project` | Runs `jsgo lint` over the project |
| `JSGo: Format File` | Formats the current buffer |
| `JSGo: Restart Language Server` | Restarts `jsgo lsp` after a toolchain change |

`jsgo run` and `jsgo build` write their executable into `.jsgo-build/`,
which `jsgo init` already adds to `.gitignore`, so running from the
editor never drops a binary next to your sources. Check and lint
report through the same `severity[code]: message` +
`--> file:line:col` shape `jsgo check` prints on a terminal, so the
extension's problem matcher and a hand-written task agree.

### Settings

| Setting | Default | Purpose |
| --- | --- | --- |
| `jsgo.path` | *(bundled toolchain)* | Use a different `jsgo` executable instead of the copy inside the extension |
| `jsgo.lsp.enabled` | `true` | Start the language server |

Resolution order: `jsgo.path` when set, otherwise the bundled copy,
otherwise `jsgo` on the `PATH`, otherwise the installer's default
locations. Only a platform the extension has no binary for, or a
deliberately emptied archive, can end up with no executable — and
then the extension says so and offers to open the setting.
Highlighting, snippets and bracket matching keep working regardless,
because they do not need the toolchain.

## Troubleshooting

- **Task never runs** — the Trigger Task on Save extension is
  installed? (Accept the recommendation from `extensions.json`, or
  install `gruntfuggly.triggertaskonsave` manually.) Workspace folder
  = the project root containing `.vscode/`?
- **`jsgo: command not found` in the task** — the scaffolded task
  already appends `~/.local/bin` to its `PATH` (`options.env` in
  `tasks.json`). If your `jsgo` lives somewhere else, add that
  directory yourself:

  ```json
  "options": { "env": { "PATH": "${env:PATH}:/opt/jsgo/bin" } }
  ```

- **Errors from a different file** — the task runs a whole-project
  check on purpose: it catches breakage anywhere, not only in the
  file you just saved.

## Known limits

Save-time checking remains the zero-setup path; the language server
(extended) adds per-edit diagnostics, hover, go-to-
definition, completion, find-references, and formatting where your
editor speaks LSP, and the shipped VS Code extension
packages all of that together with run/build/check/lint. Every one of
them runs the same checker, the same diagnostics, and the same source
positions as `jsgo check`; none of them changes how errors are
produced.

There is deliberately **no debugger**: the runtime
emits no debug information yet, so no breakpoint or stepping
protocol exists to implement. Use the `JSGo: Run File` /
`JSGo: Build File` commands (or `jsgo run` / `jsgo build` in a
terminal), and `jsgo watch` or save-time checking for feedback.
Debugging is planned once the runtime can carry debug information.
