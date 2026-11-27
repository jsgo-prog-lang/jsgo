# JSGo Standard Library

Status: **seventeen modules implemented.** `strings`, `strconv`,
`math`, `errors`, `os`, `filesystem`, `path`, `time`, `encoding`,
`crypto`, `logging`, `sync`, `json`, `net`, `http`, `database`, and
`testing` work end to end — 112 functions in total, on top of the
always-available `println`/`print` builtins. Everything after that is
the design target from the specification; the tables below describe
what ships today.

## Import rules (implemented)

```jsgo
import "strings";      // accepted
import "std/strings";  // also accepted (canonical prefix)
import "widget";       // error E0106: unknown module (note lists the known ones)
```

- Imports must precede all `function` declarations.
- One import per module per file (`E0103` on duplicates).
- An import makes the module name available as a member-access base:
  `strings.contains(s, "x")`.
- Unknown members are `E0105`; module values cannot be used as values
  (`E0209`).
- When a class or struct shares a module's name, the *imported module
  wins for calls*; without the import the class/struct wins.

## Builtins: `println` and `print`

Printing needs no import at all — the two
functions are available in every file:

| Function | Signature | Behaviour |
| --- | --- | --- |
| `println` | `(value) -> void` | prints `value` followed by `\n` |
| `print` | `(value) -> void` | prints `value` with no trailing newline |

Accepted argument types: all integers, floats, `bool`, `string` (the
compiler dispatches to a type-specific runtime function), and structs
whose fields are all printable — a `Result` prints as
`{ok: true, value: 42, error: ""}`. Exactly one
argument (`E0204` otherwise); `void`, arrays, maps, classes,
interfaces, pointers, and functions cannot be printed (`E0213` — they
have no runtime print entry
point;). Number→string conversion ships as the
built-in `string(value)`, so formatted output works today:
`print("count: " + string(n))`. Still deliberately not implemented:
`printf`-style formatting (planned with `strings` work).

A user function, struct, or local named `println`/`print` shadows the
builtin, and neither can be taken as a value (`E0101` with a builtin
help). Because they are builtins rather than a module, they are also
rejected as a `go` spawn target (`E0011`).

## Implemented: `strings`

Search and transform:

| Function | Signature | Behaviour |
| --- | --- | --- |
| `strings.contains` | `(s: string, sub: string) -> bool` | substring present |
| `strings.startsWith` | `(s: string, prefix: string) -> bool` | |
| `strings.endsWith` | `(s: string, suffix: string) -> bool` | |
| `strings.indexOf` | `(s: string, sub: string) -> int32` | first index, `-1` if absent |
| `strings.lastIndexOf` | `(s: string, sub: string) -> int32` | last index, `-1` if absent |
| `strings.replace` | `(s: string, old: string, new: string) -> string` | first occurrence only |
| `strings.replaceAll` | `(s: string, old: string, new: string) -> string` | every occurrence |
| `strings.repeat` | `(s: string, n: int32) -> string` | `n < 0` is a runtime error (exit 1) |

Split, join, trim, case:

| Function | Signature | Behaviour |
| --- | --- | --- |
| `strings.split` | `(s: string, sep: string) -> []string` | keeps empty segments; `sep == ""` splits byte-wise |
| `strings.join` | `(parts: []string, sep: string) -> string` | |
| `strings.trim` | `(s: string) -> string` | ASCII whitespace, both ends |
| `strings.trimStart` | `(s: string) -> string` | ASCII whitespace |
| `strings.trimEnd` | `(s: string) -> string` | ASCII whitespace |
| `strings.toUpper` | `(s: string) -> string` | ASCII letters only |
| `strings.toLower` | `(s: string) -> string` | ASCII letters only |

JS-style extraction and padding:

| Function | Signature | Behaviour |
| --- | --- | --- |
| `strings.substring` | `(s: string, start: int32, end: int32) -> string` | JS semantics: bounds clamp to length, reversed range swaps |
| `strings.charAt` | `(s: string, i: int32) -> string` | `""` when out of range |
| `strings.padStart` | `(s: string, width: int32, pad: string) -> string` | pad repeated; last copy may be cut |
| `strings.padEnd` | `(s: string, width: int32, pad: string) -> string` | pad repeated; last copy may be cut |

Limitations (documented, tested): case conversion and trim operate on
ASCII bytes (non-ASCII bytes pass through unchanged); `split` is
byte-wise, so `split(s, "")` yields one single-byte string per byte and
may split a multi-byte character mid-sequence; an empty pattern in
`replace`/`replaceAll` returns the input unchanged.

## Implemented: `strconv`

| Function | Signature | Behaviour |
| --- | --- | --- |
| `strconv.parseInt` | `(s: string) -> Result<int32, string>` | full string must parse; out of `int32` → error |
| `strconv.parseInt64` | `(s: string) -> Result<int64, string>` | full string must parse |
| `strconv.parseFloat` | `(s: string) -> Result<float64, string>` | full string must parse |
| `strconv.formatInt` | `(n: int32, radix: int32) -> string` | radix `2`–`36`, digits `0-9a-z` |

Error messages are stable and tested:
`strconv: invalid syntax "xyz"` and
`strconv: value out of range "99999999999"`. A radix outside 2–36 is a
hard runtime error (exit 1). `formatInt` is safe for `INT32_MIN`.

```jsgo
let n = strconv.parseInt("42");
if (n.ok) {
    println(n.value + 1);   // 43
} else {
    println(n.error);
}
```

## Implemented: `math`

All functions take and return `float64` (no integer overloads yet).

| Function | Signature | Notes |
| --- | --- | --- |
| `math.pi` | ` -> float64` | call syntax: `math.pi` |
| `math.e` | ` -> float64` | |
| `math.sqrt` | `(x) -> float64` | |
| `math.abs` | `(x) -> float64` | |
| `math.floor` | `(x) -> float64` | |
| `math.ceil` | `(x) -> float64` | |
| `math.round` | `(x) -> float64` | JS semantics: ties toward +∞ (`floor(x+0.5)`) |
| `math.sin` / `math.cos` / `math.tan` | `(x) -> float64` | radians |
| `math.log` | `(x) -> float64` | natural logarithm |
| `math.log10` | `(x) -> float64` | |
| `math.pow` | `(x, y) -> float64` | |
| `math.min` / `math.max` | `(x, y) -> float64` | simple comparison, no NaN handling |

## Implemented: `errors`

`Error` is still the builtin alias of `string`, so these are
string-valued today but the API is future-proof:

| Function | Signature | Behaviour |
| --- | --- | --- |
| `errors.new` | `(message: string) -> Error` | identity (`nil` message → `""`) |
| `errors.message` | `(e: Error) -> string` | the message |
| `errors.is` | `(a: Error, b: Error) -> bool` | compares message content |

## Implemented: `os`

| Function | Signature | Behaviour |
| --- | --- | --- |
| `os.args` | ` -> []string` | arguments after the program name (`jsgo run main.jsgo a b` → `["a", "b"]`) |
| `os.exit` | `(code: int32) -> void` | terminates immediately; deferred calls do **not** run after it (running them would need the runtime defer stack) |
| `os.getenv` | `(name: string) -> string` | `""` when unset |

## Implemented: `filesystem`

| Function | Signature | Behaviour |
| --- | --- | --- |
| `filesystem.readFile` | `(path: string) -> Result<string, string>` | whole file as a string |
| `filesystem.writeFile` | `(path: string, contents: string) -> bool` | truncate + write |
| `filesystem.appendFile` | `(path: string, contents: string) -> bool` | append |
| `filesystem.exists` | `(path: string) -> bool` | |
| `filesystem.makeDir` | `(path: string) -> bool` | single directory level |
| `filesystem.makeDirs` | `(path: string) -> bool` | mkdir -p: creates every missing level; `false` if a component exists as a file or the final path is not a directory |
| `filesystem.remove` | `(path: string) -> bool` | file or empty directory |
| `filesystem.listDir` | `(path: string) -> Result<[]string, string>` | entry names sorted ascending, `.`/`..` skipped; missing directory → OS reason |

`readFile` errors carry the OS reason:
`open /path: No such file or directory`.

## Implemented: `path`

POSIX lexical path manipulation (no filesystem access, `/` separators):

| Function | Signature | Behaviour |
| --- | --- | --- |
| `path.join` | `(parts: []string) -> string` | non-empty parts joined with `/` and cleaned; `[]` → `"."` |
| `path.basename` | `(p: string) -> string` | final element |
| `path.dirname` | `(p: string) -> string` | everything before the final element (cleaned) |
| `path.ext` | `(p: string) -> string` | JS `extname`: `".gz"`; a leading dot is a hidden file, so `".gitignore"` → `""` |
| `path.clean` | `(p: string) -> string` | lexical normalization (`.`/`..` resolution) |
| `path.isAbs` | `(p: string) -> bool` | starts with `/` |

## Implemented: `time`

| Function | Signature | Behaviour |
| --- | --- | --- |
| `time.now` | ` -> int64` | wall-clock Unix milliseconds (the `Date.now` shape) |
| `time.monotonic` | ` -> int64` | nanoseconds since an arbitrary origin — measures elapsed time, never jumps with the wall clock |
| `time.format` | `(ms: int64, layout: string) -> string` | UTC, strftime layout (`%Y`, `%m`, `%d`, `%H`, `%M`, `%S`...) |
| `time.formatLocal` | `(ms: int64, layout: string) -> string` | same, in the host's local zone |
| `time.sleep` | `(ms: int64) -> void` | parks the calling task for at least `ms` milliseconds; other tasks keep running, and `ms <= 0` returns without parking |

Unknown layout directives pass through verbatim; an empty layout
returns `""`.

## Implemented: `encoding`

Bytes are `[]uint8`; hex output is lowercase; base64 is the strict
standard alphabet with `=` padding.

| Function | Signature | Behaviour |
| --- | --- | --- |
| `encoding.hexEncode` | `(b: []uint8) -> string` | lowercase hex, two digits per byte |
| `encoding.hexDecode` | `(s: string) -> Result<[]uint8, string>` | even length and `0-9a-fA-F` only; `""` → empty array |
| `encoding.base64Encode` | `(b: []uint8) -> string` | standard alphabet, padded |
| `encoding.base64Decode` | `(s: string) -> Result<[]uint8, string>` | length must be a multiple of 4; invalid characters or misplaced padding → error |

## Implemented: `crypto`

Digests take text and return lowercase hex — the common
password/token case needs no byte plumbing; encode raw bytes with
`encoding.hexEncode`.

| Function | Signature | Behaviour |
| --- | --- | --- |
| `crypto.sha256` | `(text: string) -> string` | FIPS 180-4, 64 hex digits |
| `crypto.sha512` | `(text: string) -> string` | FIPS 180-4, 128 hex digits |
| `crypto.hmacSha256` | `(key: string, text: string) -> string` | RFC 4210/4231; a key longer than 64 bytes is hashed first |
| `crypto.randomBytes` | `(n: int32) -> []uint8` | `n < 0` is a runtime fault; entropy from `/dev/urandom` |

Vectors in the test suite are cross-checked against `openssl dgst`.

## Implemented: `logging`

One process-wide logger, explicitly configured:

| Function | Signature | Behaviour |
| --- | --- | --- |
| `logging.configure` | `(level: string, path: string) -> bool` | level `debug`/`info`/`warn`/`error` (unknown → runtime fault); `path == ""` → stderr, else append to that file; returns `false` only when the file cannot be opened — state then stays unchanged |
| `logging.debug` | `(msg: string) -> void` | below `info`, filtered by default |
| `logging.info` | `(msg: string) -> void` | |
| `logging.warn` | `(msg: string) -> void` | |
| `logging.error` | `(msg: string) -> void` | |

Defaults: level `info`, destination stderr, so the four level calls
work with zero configuration. Line format, flushed per line:

```text
<ISO-8601 UTC millis> <LEVEL> <msg>
2026-11-13T04:21:09.898Z INFO server started
```

## Implemented: `sync`

Primitives are opaque `int32` handles returned by their constructors —
not language-level types — so `sync` rides the
existing stdlib-table mechanism. Every wait/lock parks the calling
task on the cooperative scheduler (channels already do this). Misuse
is a runtime fault (stderr `jsgo: runtime error: ...`, exit 1): an
invalid handle, a handle of the wrong kind, unlocking a mutex this task
does not own, relocking it, or a negative wait-group counter.

| Function | Signature | Behaviour |
| --- | --- | --- |
| `sync.waitGroup` | ` -> int32` | new wait group |
| `sync.wgAdd` | `(h: int32, delta: int32) -> void` | counter reaching 0 wakes every waiter |
| `sync.wgDone` | `(h: int32) -> void` | `wgAdd(h, -1)` with the negative-counter fault |
| `sync.wgWait` | `(h: int32) -> void` | parks until the counter is 0 (immediate when already 0) |
| `sync.mutex` | ` -> int32` | new mutex |
| `sync.lock` | `(h: int32) -> void` | FIFO hand-off; relocking in the same task faults |
| `sync.unlock` | `(h: int32) -> void` | grants ownership to the oldest waiter before waking it |
| `sync.tryLock` | `(h: int32) -> bool` | `false` when already locked |
| `sync.rwMutex` | ` -> int32` | new read/write mutex |
| `sync.rwLock` / `sync.rwUnlock` | `(h: int32) -> void` | write side; writer priority — waiting writers keep readers out |
| `sync.rwRLock` / `sync.rwRUnlock` | `(h: int32) -> void` | shared read side; woken readers re-check (a writer may have arrived) |
| `sync.atomicLoadI32` / `atomicStoreI32` | `(*int32...) -> ...` | plain operations under the single-threaded v0.1 scheduler |
| `sync.atomicAddI32` | `(*int32, delta: int32) -> int32` | read-modify-write, returns the new value |
| `sync.atomicLoadI64` / `atomicStoreI64` / `atomicAddI64` | `(*int64...) -> ...` | 64-bit variants |

```jsgo
let wg = sync.waitGroup;
let m = sync.mutex;
sync.wgAdd(wg, 2);
go worker(wg, m);
go worker(wg, m);
sync.wgWait(wg);
println("both done");
```

## Implemented: `json`

`json.parse` builds a dynamic `Json` tree — the runtime value behind
the builtin `Json` type — and `json.stringify` serializes any
stringifiable value back to text:

| Function | Signature | Behaviour |
| --- | --- | --- |
| `json.parse` | `(text: string) -> Result<Json, string>` | strict RFC 8259; every error carries its byte offset |
| `json.stringify` | `(value) -> string` | type-dispatched like `println`: integers, floats, `bool`, `string`, structs (recurring through fields), arrays, maps, `Json` trees, and a `Result` carrying one |

The nine accessors are builtin methods on the `Json` value itself
(the import only provides `parse`/`stringify`):

| Method | Signature | Behaviour |
| --- | --- | --- |
| `.get` | `(key: string) -> Json` | object member; missing key → undefined |
| `.at` | `(i: int32) -> Json` | array element; out of range → undefined |
| `.str` | ` -> string` | string payload; wrong kind → `""` |
| `.num` | ` -> float64` | number payload; wrong kind → `0` |
| `.bool` | ` -> bool` | boolean payload; wrong kind → `false` |
| `.isNull` | ` -> bool` | `true` for `null` **and** undefined — "no value here" (JS `== null`) |
| `.has` | `(key: string) -> bool` | object member present (objects only) |
| `.len` | ` -> int32` | string length, array length, or object key count; other kinds → `0` |
| `.keys` | ` -> []string` | object keys; other kinds → `[]` |

Rules and limits (all tested):

- Wrong-kind accessors are **total**: they yield zero values, never a
  fault, so navigating untrusted input is safe.
- `Json` values cannot be constructed (`Json(...)` → `E0221`),
  compared (`==`/`!=` → `E0202`), printed (`E0213`), or given methods
  (`E0104`); trees come only from `json.parse`.
- `stringify` rejects classes, function values, and `void` (`E0223`).
- Numbers print the shortest decimal form that round-trips; `NaN` and
  infinities print `null`, `-0` prints `0` (JavaScript
  `JSON.stringify` behavior). Non-string map keys stringify as text
  (`{1: "one"}` → `{"1":"one"}`). Object key order is unspecified
  (hash-table order).
- Parse errors are stable strings `json parse: <reason> at offset <n>`
  (e.g. `unexpected end of input`, `expected a string key`); duplicate
  keys keep the last value; `\u` escapes decode with surrogate pairs;
  nesting past 128 levels fails the parse with
  `maximum nesting depth exceeded (128)`.
- A cyclic value passed to `stringify` faults the program (exit 1)
  with `json.stringify: maximum nesting depth exceeded (128)` — JSON
  text cannot express a cycle.

```jsgo
let r = json.parse("{\"name\":\"Ada\"}");
if (r.isOk) {
    let j = r.value;
    println(j.get("name").str);  // Ada
    println(j.len);              // 1
    println(json.stringify(j));    // {"name":"Ada"}
}
```

## Implemented: `net`

Sockets are opaque `int32` handles misused the way
the `sync` primitives are: any operation on an unknown,
closed, or wrong-kind handle faults the program (exit 1) with the
operation named. `listen` and `http.serve` take a numeric
`"host:port"` — an empty host (`":0"`) binds all interfaces, a
non-numeric host is rejected because name resolution would block the
scheduler, and port `0` requests an ephemeral port that `localAddr`
reads back as `"ip:port"`. `accept` and `read` park the calling task
until a connection or data arrives, so a blocked socket never
freezes other tasks; `net.close` on a handle wakes anything parked on
it with an error.

| Function | Signature | Behaviour |
| --- | --- | --- |
| `net.listen` | `(addr: string) -> Result<int32, string>` | binds a numeric IPv4 `host:port` (`":0"` = ephemeral); errors: `net: address must be host:port`, `net: only numeric IPv4 hosts are supported`, `net: port must be 0-65535`, or the OS bind failure |
| `net.accept` | `(h: int32) -> Result<int32, string>` | parks until a connection arrives, returns a connection handle; `net.close(h)` while parked aborts with `net: socket closed` |
| `net.read` | `(h: int32) -> Result<string, string>` | parks until data arrives, returns up to 8 KiB; end of stream is `Ok("")`; close-while-parked aborts with `net: socket closed` |
| `net.write` | `(h: int32, data: string) -> Result<int32, string>` | sends every byte (parking for buffer room), returns the byte count; strings are NUL-terminated, so a payload may not contain `\0` |
| `net.close` | `(h: int32) -> void` | closes the socket and aborts its parked waiters; a second close faults (`invalid handle`) |
| `net.localAddr` | `(h: int32) -> string` | bound `"ip:port"` (empty string if the query fails) |

```jsgo
let lr = net.listen("127.0.0.1:0");
if (lr.isOk) {
    let h = lr.value;
    println(net.localAddr(h));  // e.g. 127.0.0.1:53411
    let cr = net.accept(h);         // parks until a client arrives
    ...
}
```

## Implemented: `http`

`http.serve` is a single-handle HTTP/1.1 **server** (there is no HTTP client in v0.1). It binds like `net.listen`,
spawns the accept loop as a scheduler task, and returns the listener
handle immediately — `main` may return while the server keeps
serving, and `net.close(handle)` shuts it down (the parked accept
loop aborts and the process exits when no task remains). Each
connection becomes its own task, so a slow handler never blocks the
next request.

| Function | Signature | Behaviour |
| --- | --- | --- |
| `http.serve` | `(addr: string, handler: function(Request): Response) -> Result<int32, string>` | binds, spawns the accept loop, returns the listener handle; address errors are the same as `net.listen` |

The handler receives a builtin `Request` value and returns a
builtin `Response`; both are ordinary struct values (constructors,
fields, `==`, `json.stringify` all work; declaring methods on them is
`E0104`):

| Type | Fields |
| --- | --- |
| `Request` | `method: string`, `path: string`, `query: string` (raw, no `?`), `headers: map[string]string` (names lowercased), `body: string` |
| `Response` | `status: int32`, `headers: map[string]string`, `body: string` |

Server behaviour (all tested):

- The runtime writes `Content-Length` from the body and
  `Connection: close` (no keep-alive in v0.1); user headers with
  those names are skipped.
- Malformed request heads answer `400 Bad Request`, a
  `Transfer-Encoding` other than none answers `501 Not Implemented`,
  a head larger than 64 KiB answers `400`, and a declared body larger
  than 16 MiB answers `413` — always without waking the handler.
- Headers are read case-insensitively into lowercased names; the
  query string is passed through raw.
- Strings are NUL-terminated internally, so request lines, header
  values, and bodies may not contain `\0` (documented limitation).

```jsgo
function handle(req: Request): Response {
    return Response(200, {"Content-Type": "text/plain"},
        req.method + " " + req.path);
}

function main {
    let r = http.serve("127.0.0.1:8080", handle);
    if (!r.isOk) {
        println("serve failed: " + r.error);
    }
}
```

## Implemented: `database`

`database` is an embedded row store: a directory of
JSON-lines table files, one `{"key":...,"value":...}` row per line,
rewritten and flushed on every mutation. Handles are opaque `int32`
values — process-lifetime (there is no `close`), created by `open`
and misused the way `sync`/`net` handles are: any operation
on an unknown handle faults the program (exit 1).

| Function | Signature | Behaviour |
| --- | --- | --- |
| `database.open` | `(path: string) -> Result<int32, string>` | creates the directory if needed and returns a 1-based store handle; a path that is not a directory is an error |
| `database.put` | `(h: int32, table: string, key: string, value: string) -> Result<int32, string>` | upserts the row in place (an update keeps its position); `Ok(1)` |
| `database.get` | `(h: int32, table: string, key: string) -> Result<string, string>` | the row's value; a missing key is `Err("database: key not found")` |
| `database.delete` | `(h: int32, table: string, key: string) -> Result<int32, string>` | removes the row; a missing key is the same not-found error; `Ok(1)` |
| `database.list` | `(h: int32, table: string) -> Result<string, string>` | a JSON array of the table's keys in file order (e.g. `["1","2"]`) |

Rules and limits (all tested):

- Table names must match `[A-Za-z0-9_-]+` (max 128) — anything else
  is `Err("database: invalid table name")`, so a handle can never
  escape its directory.
- `database: key not found` is a **stable exact string**: compare it
  with `==` to distinguish "missing" from an OS failure.
- Keys and values are arbitrary strings; they are JSON-escaped into
  the row, so quotes, newlines, and Unicode round-trip.
- Rows the parser cannot read (a hand-edited file) pass `put` and
  `delete` untouched and stay invisible to `get` and `list`.
- OS failures (permissions, a full disk) surface as the same
  `read <path>: <reason>` / `write <path>: <reason>` errors the
  `filesystem` package uses.

```jsgo
let h = database.open("data");
if (!h.isOk) { println(h.error); return; }
let s: int32 = h.value;
database.put(s, "todos", "1", "write tests");
let g = database.get(s, "todos", "1");
println(g.value);                 // write tests
println(database.list(s, "todos").value);  // ["1"]
```

## Implemented: `testing`

The Go-style test framework, driven by `jsgo test`:
discovery collects every `Test*` function (parameterless, `void`,
not async), the build renames the program's `main` and synthesizes a
driver that runs them in source order, and the process exit code
reports failures. The pieces are importable directly, so a
hand-written driver works too:

| Function | Signature | Behaviour |
| --- | --- | --- |
| `testing.run` | `(name: string, fn: function: void) -> void` | runs one test: prints `PASS  <name>` or `FAIL  <name>` plus the recorded message; nested runs fault |
| `testing.finish` | ` -> void` | prints `<n> tests, <p> passed, <f> failed` and exits 1 when any test failed |
| `testing.assert` | `(cond: bool, msg: string) -> void` | passes when `cond`, otherwise records `msg` and stops the test |
| `testing.fail` | `(msg: string) -> void` | records `msg` and stops the test unconditionally |
| `testing.equal` | `(got, want, msg: string) -> void` (both sides any stringifiable type) | type-dispatched like `json.stringify`: both sides are serialized and compared; on mismatch the message gains `got:`/`want:` lines |

Rules and limits (all tested):

- `equal` takes any stringifiable value (the `json.stringify` set);
  both operands must have the **same type** (`E0202` otherwise, like
  `==`), with literal adoption in either order (`equal(n, 5...)`
  and `equal(5, n...)`). Non-stringifiable operands are `E0223`.
- A failing assertion **stops that test**, not the process: the
  message is recorded and control returns to `run`, which prints
  `FAIL` and moves on. The summary and exit code come from `finish`.
- An assertion outside `run` (or from a different task) is a program
  fault: `jsgo: runtime error: testing assertion outside a test`.
- Map equality follows `stringify`'s key order, so compare maps
  built the same way.

```jsgo
function TestAdd {
    let sum: int64 = 2 + 3;
    testing.equal(sum, 5, "addition works");
    testing.assert(sum > 4, "sum is greater than four");
}

function main {
    testing.run("TestAdd", TestAdd);
    testing.finish;
}
```

## Planned packages (specification §9)

```text
std/
├── strings/ strconv/ math/ time/
├── collections/ errors/ json/ encoding/ io/
├── os/ filesystem/ path/ crypto/ sync/ context/
├── net/ http/ url/ database/ testing/ logging/
├── regex/ process/ environment/ runtime/
```

`io`, `regex`, `url`, and `context` wrappers are not implemented
and are recorded as such rather than half-built.

## Conventions (enforced as packages land)

- Package names are lowercase, single-word where possible.
- Functions are explicitly typed; no hidden global state.
- Errors are returned as values: `Result<T, E>` with the contextual
  `Ok(v)`/`Error(msg)` constructors and the `isOk`/`value`/
  `error` methods (implemented); no
  exceptions. `Error` is a builtin alias of `string`; structured error
  types are still blocked because a `Result` payload needs a zero
  value and class references are never null (`E0201` on a class
  payload), so the alias remains until another mechanism lands.
- Runtime faults inside library code (negative `repeat`, radix out of
  range) print `jsgo: runtime error: ...` to stderr and exit 1, like
  the rest of the runtime.
- Every package is verified by its own test suite; a package is
  listed as implemented only once those tests pass.
