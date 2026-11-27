# JSGo Security Notes

Status: a formal security review has not yet been performed. This
records
the security-relevant properties of the current implementation and the
threat model for what is planned. Nothing here is a claim of a formal
audit.

## Threat model (current)

The JSGo toolchain runs locally, on the developer's machine, during
builds. The trust boundaries are:

1. **Source code → compiler.** A `.jsgo` file is untrusted input to
   `jsgo check/build`. Expected outcome: a diagnostic, never a crash,
   hang, or arbitrary file write.
2. **Compiler → host C toolchain.** Generated C and the embedded
   runtime are produced by the compiler itself (not by the source
   program), so source-level injection into the C compilation step is
   not reachable from `.jsgo` text.
3. **Compiled program → runtime environment.** The produced executable
   runs with the user's privileges; JSGo has no sandbox.

## Properties of the current implementation

- **No format-string surface.** `println` compiles to one of 12
  type-specific C functions (`jsgo_print_int32(int32_t)`, …). There is
  no user-controlled format string anywhere in the runtime, and no
  varargs printing.
- **No dynamic evaluation.** JSGo has no `eval`, no runtime code
  loading, and no plugin mechanism. Compiled programs cannot generate
  or execute code.
- **Builds are hermetic with respect to the network.** `jsgo check`,
  `build`, `run`, `test`, `fmt`, `lint`, and `doc` never fetch:
  package sources come from the local content-addressed module cache,
  verified against the SHA-256 digests `jsgo.lock` pins. Only
  the explicit dependency commands `jsgo get` and `jsgo update`
  perform network I/O — HTTPS-only, plain `http://` refused except
  loopback, redirects re-checked — and they extract archives without
  ever executing dependency-supplied code.
- **No third-party dependencies.** The Go module has no external
  dependencies; compiled programs link only against libc. This removes
  the dependency-supply-chain attack surface for the toolchain itself.
- **Strict typing as an injection barrier.** No implicit conversions
  mean a `string` cannot silently become a number (or vice versa) at
  any boundary — a class of bugs that becomes command/format injection
  in weaker languages.
- **Definite assignment and no uninitialized reads** remove
  uninitialized-memory disclosure from the language level (strings and
  primitives are initialized before use; `E0210`).
- **Diagnostic rendering escapes source text.** Rendered diagnostics
  print source excerpts as data, not as terminal control sequences
  beyond the excerpt itself.
- **Package fetching is verified and constrained.** `jsgo get`
  and `jsgo update` fetch over HTTPS only (plain `http://` refused
  except loopback, redirects re-checked), verify every archive
  against the SHA-256 pinned in `jsgo.lock`, refuse zip-slip paths,
  symlinks, and oversized entries during extraction, extract only
  `.jsgo` and `jsgo.mod`, and never run dependency code — there are
  no dependency-supplied build scripts (see `PACKAGE_MANAGER.md`).

## Known gaps (tracked, not ignored)

| Gap | Tracking |
| --- | --- |
| Conservative collection retains garbage until the next pass (a stale pointer-like word keeps its block alive); stop-the-world pauses grow with heap size | `MEMORY_MODEL.md`; live task stacks are scanned, precise/scheduler-driven collection stays future work for v0.1 |
| Integer overflow wraps silently | planned review |
| Division by zero follows C semantics (SIGFPE) at runtime | planned review (runtime hardening) |
| No sandboxing of compiled programs | out of scope (native executables) |
| Installers are not code-signed, so SmartScreen and Gatekeeper warn on first run | Preparing signing behind an optional CI secret; user-facing steps are in the shipped README |

### Installer signing

Every installer this project ships — the Windows MSI, both macOS
`.pkg`s and both Linux `.deb`s — is currently **unsigned**. That is a
distribution gap, not a code gap: nothing in the toolchain is
affected, but Microsoft Defender SmartScreen and macOS Gatekeeper
warn on first run, and SmartScreen has no publisher identity to
carry trust forward, so **every new release starts its reputation
from zero**.

Signing is being prepared behind an optional CI secret so the MSI
signs the moment credentials exist; until then the build is
unchanged. Two things worth knowing before spending money on it:

- **Signing does not remove the warning outright.** Microsoft's own
  guidance is that a newly signed file still shows an
  "unrecognized app" prompt until the publisher identity accumulates
  download history — typically several weeks and hundreds of clean
  installs. What signing buys is reputation that accumulates *across*
  versions instead of resetting each release.
- **EV certificates no longer bypass SmartScreen.** That behaviour
  was removed in 2024; EV and OV now go through the same reputation
  process. Paying the EV premium purely to silence the prompt is no
  longer justified.

On Windows 11 with **Smart App Control** enabled, unsigned files are
blocked outright rather than warned about, and Smart App Control
cannot be re-enabled once turned off.

## Planned

- **Crypto stdlib (`std/crypto`):** vetted primitives only; no
  hand-rolled ciphers in application code.
- **Fuzz testing of the lexer/parser/checker** (planned) — the front
  end is the component most exposed to hostile input.
- **Security review and memory-safety review** as release exit
  criteria, including the C runtime once channels and the scheduler
  land (the conservative collector is in scope from now on).

## Reporting

Security issues in the toolchain should be reported privately to the
maintainers before public disclosure.
