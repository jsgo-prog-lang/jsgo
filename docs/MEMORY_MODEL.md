# JSGo Memory Model

Status: **implemented and tested.** Value semantics, immutable
strings, reference-handle arrays/maps, user-visible pointers with
`nil`, and a conservative mark–sweep collector
are implemented and tested. This document describes what the
language guarantees today and what remains designed-but-deferred.

## What exists today (implemented)

- **Value types.** All primitive values (`bool`, integers, floats) are
  plain values copied on assignment, on argument passing, and on
  return. There is no aliasing and no observable sharing.
- **Structs are values too**. Assigning,
  passing, or returning a struct copies its fields; writes to `q.x`
  after `let q = p;` never touch `p`. Structs are laid out as plain C
  structs — no heap allocation, no handles. Two caveats follow from
  that: a struct's array/map **fields** are still reference handles
  (copying the struct copies the handles, so field mutations through
  the copy are shared), and methods receive a **copy** of the receiver,
so mutations inside a method body do not escape it — unless
  the method declares a **pointer receiver** (below).
- **Strings are immutable and heap-allocated.** Every `string` is a
  NUL-terminated UTF-8 byte array allocated by the runtime
  (`jsgo_str_concat`, `jsgo_*_to_string`, string literals).
  Assignment shares the reference; `+` allocates a fresh string. There
  is no string mutation API, so sharing is safe. Concatenated and
  converted strings are managed heap objects the collector reclaims
  when unreachable; **string literals are static C literals baked into
  the executable** — immortal, never collected.
- **Arrays and maps are reference handles**. A
  `[]T` or `map[K]V` value is a handle to a heap structure: assigning
  or passing one copies the **handle**, and `push`, element
  assignment, `delete`, etc. are visible through every alias:

  ```jsgo
  let a: []int32 = [1, 2];
  let b = a;          // same buffer, not a copy
  b.push(3);
  // a.length is now 3 as well
  ```

  Value-semantics copies (`let b = []int32{...}`-style deep copy) do
  not exist yet; if you need an independent array, build a new one.
  Buffers are managed heap objects: an unreachable array/map is
  reclaimed by the collector.
- **Classes are heap references and are never null**. `new C(...)` allocates a zeroed instance and runs its
  constructor; class-typed variables, parameters, and fields hold the
  reference, so assignment and passing share the instance (mutating
  fields through one alias is visible through the other — that is how
  classes differ from structs). The non-NULL invariant stands: there is
  no `nil` of class type — the compiler rejects a valueless
  class-typed `let` (`E0218`) and requires constructors to assign
  their class-typed fields (`E0210`). Instances are managed heap
  objects (`jsgo_class_new`) the collector reclaims when unreachable.
- **Pointers and `nil`**. `*T` is a first-class
  type for every element type; `nil` is only a `*T`-typed value and
  needs an expected type (assignment, annotation, argument, or a
  comparison against a typed pointer). `&x` takes the address of a
  variable, a variable-rooted field chain (`&o.inner.v`), or an array
  element (`&a[i]`); `*p` reads and writes through a pointer
  (`*p = v`, `*p += 1`). Key guarantees:
  - **No dangling pointers from safe code.** An address-taken
    *variable* is lowered into a **GC box** — a heap cell holding the
    variable — so returning or storing `&x` keeps the data alive and
    reachable for the collector. Class-field and array-element
    addresses need no box: their storage is already on the managed
    heap (those addresses are *interior* pointers, which the collector
    marks by address range).
  - **`nil` dereference is a checked runtime error**, never memory
    corruption: `jsgo: runtime error: nil pointer dereference`
    (exit 1).
  - Pointers are otherwise restricted: not printable (`E0213`), no
    arithmetic, not indexable; the only `nil` comparisons are
    `p == nil` / `p != nil` (pointer identity). There is no
    auto-dereference sugar — `p.f` on a `*T` is `E0216` with the help
    `(*p).f`.
  - Pointer receivers (`function (p: *Point) m`) mutate the pointee;
    receiver kinds match exactly at call sites, and classes reject
    `*C` receivers (class references are already pointers).
- **Garbage collection**. The v0.1 collector is
  a **stop-the-world, single-threaded, conservative mark–sweep** over
  a registered heap, inside the embedded C runtime:
  - **Registry.** All runtime allocations (`jsgo_alloc`,
    `jsgo_class_new`, `jsgo_realloc`, `jsgo_gc_free`, and the box
    allocator) append `(base, size)` records to an append-only log;
    collection folds it to one live record per address.
  - **Roots.** Registers via `setjmp` at collection entry; the
    executing region — the current task's stack up to its live top,
    or the C stack from `stack_top` when no task is running — plus
    the scheduler's own frames when a task executes;
    **every live task's whole stack** (a parked task's 1 MiB stack is
    registered as a conservative range on spawn and released when the
    task exits); and registered root slots (every generated
    static plus the runtime's cached `os.args`).
  - **Conservative on both ends.** Any scanned machine word inside a
    registered block — base or interior — marks it, and marked blocks
    are scanned for more words. False positives only retain garbage
    temporarily; live data is never freed.
  - **Trigger and sweep.** Collection runs before `malloc` once the
    bytes allocated since the last pass exceed 16 MiB; sweep compacts
    the registry and frees unmarked blocks. `JSGO_GC_STATS=1` reports
    collections and freed bytes at exit (used by the integration
    tests).
  - No finalizers and no weak references: deterministic release of
    files/sockets is `defer`'s job, not the GC's.

## Limitations of the current runtime

- The collector is conservative, so garbage can be retained until the
  next pass (a stale word that looks like a pointer keeps its block
  alive), and it is single-threaded stop-the-world. Concurrent,
  precise collection with per-task arenas stays designed-not-
  implemented (below) and is superseded for v0.1.
- Small programs may allocate under 16 MiB and never trigger a
  collection at all — they behave exactly as before, with memory
  reclaimed only at process exit.
- External resources (files, sockets) do not exist yet; when they land
  they will require explicit cleanup (`defer` exists for top-level
  calls; conditional cleanup needs the runtime defer stack, still
  future), not GC.
- Channels are live: a `Channel<T>` handle and its
  buffer (a buffered channel's ring) are managed heap objects
  allocated through the collector, so they are reclaimed like arrays —
  see `CONCURRENCY_MODEL.md`.

## Designed but superseded for v0.1: concurrent precise collection

The original design intent — recorded so the swap is one reviewed
change:

- A single managed heap for all runtime-allocated objects (strings,
  arrays, maps, class instances, boxes, channel handles and buffers).
- A concurrent mark-and-sweep collector driven by the task scheduler
  (see `CONCURRENCY_MODEL.md`), with allocation on per-task arenas to
  reduce contention and exact roots (stack maps, write barriers)
  instead of conservative scanning. The conservative collector
  remains the v0.1 implementation —
  now scanning live task stacks (above); precise per-task-arena
  collection stays future work.
- No finalizers for correctness-critical cleanup. Deterministic
  release of files/sockets is the job of `defer`, not the GC.
- Reachability from task stacks and global roots only — no weak
  references in v1.

## Ownership rules (what the language guarantees)

1. A value is never mutated behind your back: primitives are values,
   strings are immutable. Arrays and maps are the exception — they are
   explicitly shared handles whose mutations are visible through every
   alias (this is documented, not hidden).
2. There is no way to create a dangling reference from safe JSGo code:
   address-taken variables live in GC boxes, arrays/maps/classes
   allocate on the managed heap, interior pointers are marked by
   address range, and dereferencing `nil` is a checked runtime error.
  Memory is reclaimed by reachability — the language has
   no manual free.
3. Class instances are allocated by the runtime and reclaimed by the
   collector when unreachable. When external resources arrive, they
   are `defer`-released. The language will not mix the two.
