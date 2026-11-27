# JSGo Concurrency Model

Status: **implemented**. One cooperative scheduler runs every
task on a single OS thread today; an M:N worker pool is the next
step. `go`, `async`, `await`, and
`select` are live syntax; `chan` stays reserved with an `E0020` help
that points at `Channel<T>`, and `range` waits for a later phase.
Timer deadlines and descriptor `poll` waits joined the park/
resume sources, backing `time.sleep`, `net`, and `http.serve`.

## Core decision: one scheduler, two syntaxes

JSGo does **not** implement goroutine-style tasks and async/await as
two separate systems. There is one task model and one scheduler:

- An `async function` is a function whose declared result is logical:
  calling it eagerly spawns a task and yields a `Task<T>`, and the
  function body may `await` other tasks.
- `go f(x)` starts a task running `f(x)` on the same scheduler.
- `await` is a scheduler-aware suspension point; channels and `select`
  are scheduler-aware blocking points. Both suspend the task, never a
  thread.

## Scheduler (implemented, v0.1)

- Single-threaded cooperative scheduler over `ucontext` task stacks
  (1 MB each). Every live task runs to a suspension point or to
  completion before another runs; there are no OS threads behind the
  program yet.
- **Preemption points.** The compiler splices a tick into every
  function: before each call-family instruction, at every loop
  back-edge, and at least every 256 straight-line instructions, so a
  compute-bound task cannot starve the others. `jsgo_tick` is a no-op
  when the scheduler is inactive or nobody else is ready, so a plain
  program pays nothing.
- **Blocking operations park, never block.** An `await` on an
  unfinished task, a channel `send`/`recv` with no partner, a
  `select` with no ready arm, a `time.sleep` deadline, and a socket
  `accept`/`read`/`write` with nothing ready suspend the current task
  and hand control to the scheduler, which runs ready tasks until none
  remain. When only timers and descriptor waits remain, the scheduler
  fires due timers, then `poll`s the registered descriptors with a
  timer-bounded timeout instead of reporting a deadlock.
- **Entry point.** `main` runs synchronously as task 0 inside the
  scheduler — which is exactly why an `async main` is `E0500`. The
  process exits when every task has finished; if every live task is
  parked with **no timer and no descriptor wait pending**, the runtime
  reports `jsgo: deadlock: all tasks are parked` and exits 1. Closing
  a handle aborts whatever was parked on it with an error (the
  close-while-parked rule), so a server's `net.close` is a clean
  shutdown, not a hang.
- **Cancellation.** A `Context` token passed as a task's first argument
  becomes the task's context. Cancelling it wakes awaiters with the
  zero value; `await` also re-raises immediately when the context is
  already cancelled.

## Tasks and async (implemented)

- `async function f(...): T` — the annotation is the *logical* result.
  The call spawns eagerly (no `.then` scheduling) and the call
  expression has type `Task<T>`; a `void` body yields `Task<void>`
  (`Task<T>` is synthesized by the compiler and allows `void`).
- `await t` (only inside an async function) waits for the result. On a
  cancelled context it resumes with `T`'s zero value instead of the
  result.
- `go f(args)` accepts **named calls only**: plain functions, methods,
  statics, `super`, generic instances, and module functions. A
  function-value callee is `E0011` (no stable declaration to target),
  and so are the print builtins, constructors, and handle/collection
  operations — there is no function to run in a task.
- Async class methods are `E0011`; an async entry point is `E0500`.
- A spawn whose first argument is a `Context` passes it as the task's
  context (and to the callee, which declares it as its first
  parameter).

## Channels (implemented)

- Constructed with the builtin `Channel<T>(capacity)`; an omitted
  capacity is 0 (unbuffered). `chan` is reserved and never becomes
  syntax — the `E0020` help points at `Channel<T>`.
- Method set: `ch.send(v)`, `ch.recv: T`, `ch.close`,
  `ch.isClosed: bool`, `ch.length: int32`.
- Semantics: `send` blocks when the buffer is full; `recv` on a closed
  and drained channel yields the zero value immediately; `send` on a
  closed channel and a double `close` are runtime errors; the runtime
  reports deadlock when all tasks are parked and no timer or
  descriptor wait is pending. Element types may be any
  type except `void`.

## Select (implemented)

```
select {
case let v = ch.recv:   // receive into a bound variable
    ...
case ch.send(x):          // send
    ...
default:                  // optional, non-blocking
    ...
}
```

- One runtime probe picks uniformly at random among the ready arms
  (deterministic xorshift sequence), performs it without yielding in
  between, and returns its index; the compiled comparison chain jumps
  to that arm's body.
- With no ready arm: the `default` body runs immediately; without a
  `default`, the task parks until an arm fires.
- There is no `<-` operator anywhere in the language.

## Synchronization (implemented)

`WaitGroup`, `Mutex`, `RWMutex`, and atomic operations ship as
`std/sync`: opaque `int32` handles from the
constructors, wait/lock operations parking the calling task on the
same park/resume path channels use, FIFO wait queues (channels stay
LIFO), writer-priority rw mutex with a reader re-check loop, plain
atomics under the v0.1 single-threaded scheduler, and misuse as
runtime faults (invalid handle, wrong kind, foreign/reentrant
unlock, negative counter). Race-detection tooling is planned
for a future release.

