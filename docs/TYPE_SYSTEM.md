# JSGo Type System

JSGo has strict static typing. There is **no implicit conversion of any
kind** — not between numbers, not between strings and numbers, not
between unrelated types. Every rule below is enforced by the
type checker and covered by the test suite.

## Primitive types

| Category | Types |
| --- | --- |
| Void | `void` |
| Boolean | `bool` |
| Signed integers | `int8` `int16` `int32` `int64` |
| Unsigned integers | `uint8` `uint16` `uint32` `uint64` |
| Floating point | `float32` `float64` |
| Text | `string` |

## Composite types

| Form | Example | Notes |
| --- | --- | --- |
| Array | `[]int32`, `[][]string` | dynamic, any nesting; `.length` is `int32` |
| Map | `map[string]int32`, `map[int32][]int32` | keys must be scalar in literals; `.length` is `int32` |
| Struct | `Point`, `Bag` | declared with `struct Name { ... }`; nominal, value semantics |

- Composite type names are structural: two `[]int32` types are
  identical wherever they appear; `[]int32` and `[]int64` are not.
  **Struct types are nominal**: identity is the declared name, so two
  structs with identical fields but different names are different
  types, and one name may not be declared twice (`E0103`).
- Empty literals (`[]`, `{}`) have no element to infer from and
  require an annotation (`E0218`): `let a: []int32 = [];`.
- An array literal's elements must all be assignable to one element
  type; the first element sets the inferred type (`E0201` on
  mismatch). Map keys/values follow the same rule, and map keys must
  be scalar (int, float, string, bool — `E0202`).
- Arrays and maps are **reference handles** at runtime: assigning one
  variable to another copies the handle, and mutations are visible
  through every alias (see `MEMORY_MODEL.md`). Structs are the opposite: **values** that copy
  on assignment, argument passing, and return.
- Index rules: array indices must be integers (`E0202`), map keys
  must be assignable to the key type (`E0201`). Reading a missing map
  key or an out-of-bounds array index is a **runtime error** (message
  plus exit status 1), not silent corruption — test with `.has`.

## Pointer types

```jsgo
let x: int32 = 5;
let p: *int32 = &x;     // *T: address-of a variable
*p = 10;                // dereference (assignable: *p = v, *p += 1)
let q: *int32 = nil;    // nil: only a *T value, needs an expected type
```

- `*T` is a first-class type for every element type and nests like the
  composites (`*int32`, `**Point`, `*[]int32`, `*map[string]int32`).
  The spelling is canonical everywhere (`types.String` and the parser's
  `ResolveType` agree), and pointer types compare like scalars
  (`*int32` ≡ `*int32`). `*void` is rejected (`E0006`).
- **`nil` is only a pointer value**: it takes its type from the
  expected type — assignment, annotation, argument, return, or
  comparison against a typed pointer — and is a targeted error
  anywhere else. Classes and interfaces keep their non-NULL invariant
  (above), and `Result<T, E>` remains the optional-result tool.
- Comparisons: `p == nil` / `p != nil` and pointer-to-pointer identity
  are the only equality forms; pointers are not printable (`E0213`),
  not indexable, and support no arithmetic (`E0202`).
- **Addressability.** `&x` accepts variables (locals, parameters),
  variable-rooted field chains (`&o.inner.v`), and array elements
  (`&a[i]`); map elements are not addressable (`E0212` — growth would
  move them). `&*p` simplifies to `p`. There is no auto-deref sugar:
  `p.f` on a `*T` is `E0216` with the help `(*p).f`.
- **Dangling is ruled out.** Address-taken variables are lowered into
  GC boxes (heap cells) that the conservative collector keeps while
  reachable; class fields and array elements already live on the
  managed heap, so their addresses are interior pointers the collector
  marks by range (`MEMORY_MODEL.md`).
  Dereferencing `nil` is a checked runtime error (`jsgo: runtime
  error: nil pointer dereference`, exit 1).
- **Pointer receivers.** `function (p: *Point) m(...)` is legal on
  structs and mutates the pointee (the value-receiver copy rule, applies only to `(p: Point)`). Receiver kinds match exactly:
  a value receiver is called on values, a pointer receiver only on
  `*T` (`E0201` on mismatch — write `(*q).m` or `(&x).m`
  explicitly). Classes reject `*C` receivers (`E0104`): class
  references are already pointers.

## Struct types

```jsgo
struct Point { x: int32; y: int32; }

let p = Point(3, 4);   // positional constructor, declaration order
p.x = 10;              // field read/write; chains: p.inner.v
```

- A struct field may be any non-`void` type, including other structs
  (declared later — no forward-reference restriction), arrays, and
  maps. A struct that contains **itself by value** is rejected
  (`E0219`) because the layout would be infinite; cycles through
  `[]T`/`map[K]V` fields are legal because those are fixed-size
  handles.
- Construction is positional in declaration order: wrong arity is
  `E0204`, wrong field type is `E0201`, unknown fields report the
  available ones (`E0216`).
- Structs are **values**: `let q = p;` copies `p`, and later writes to
  `q.x` do not affect `p`. Arrays and maps inside a struct are still
  handles — copying the struct copies the handles, not the buffers.
- Methods receive a **copy** of the receiver (Go value-receiver
  semantics); mutation through a method needs a **pointer
  receiver** — `function (p: *Point) m(...)` (below).
- **Value equality**: `==`/`!=` on two identical non-interface
  struct types compare **field by field, deeply** — scalars by value,
  `string`s by content, nested structs recursively, and every
  reference-shaped field (`*T`, arrays, maps, class references,
  functions) **by identity**, matching JS `===` on
  objects. An interface field compares by the object it wraps: the
  same object stamped at any of its hierarchy's implementing classes
  is equal. Classes keep identity equality (above) and
  interfaces stay non-comparable (`E0202`); a struct is never a
  valid condition (`E0203`).
- **Printing**: `print`/`println`
  accept a struct whose fields are all printable. It writes a JS
  container — `{x: 1, name: "hi"}`, `{}` when empty — with string
  fields quoted and escaped and nested structs recursed. A struct
  holding a class, interface, array, map, pointer, function, or tuple
  field is not printable (`E0213`).
- A struct name used as a value (`let v = Point;`) is `E0209` —
  construct one: `Point(...)`.

## Class types

```jsgo
class User {
    private id: int32;
    public name: string;
    constructor(id: int32, name: string) {
        this.id = id;
        this.name = name;
    }
    public getName: string { return this.name; }
}

let u: User = new User(1, "ada");   // heap reference, never null
```

- A class is a **nominal reference type**. `new C(args)` heap-
  allocates a zeroed instance and runs the constructor; variables,
  parameters, fields, and results of class type hold the reference
  (one pointer per hierarchy — all subclasses share their root's C
  representation).
- **Non-NULL invariant:** a class-typed value is never null — there is
  no `nil` of class or interface type (`nil` exists only for `*T`
  types, below). A class-typed `let`
  without a value is `E0218`; structs and tuples that embed a class
  reference therefore have no zero value (their `let`s need an
  initializer too); a `Result` payload may not contain a class
  reference (`E0201`, because the failed case materializes a zero
  value); every class-typed field a class declares must be assigned by
  a top-level `this.f = ...;` statement in its constructor (`E0210`,
  checked at the constructor's end), and `return` in a constructor is
  `E0011`.
- **Inheritance is single** (`extends`). Fields are flattened
  root-first in the layout, so a subclass starts with exactly its
  root's prefix. A subclass value is assignable to any superclass
  variable (upcast, below); the reverse needs no conversion either —
  they are the same reference — but method lookup on the static type
  governs what is visible (`E0216` for unknown members, `E0107` for
  access violations).
- **All non-constructor methods are virtual**, including private ones
  (private is a compile-time access check, not dynamic hiding — a
  documented difference from Java). An override must match the parent
  signature exactly: arity `E0204`, parameter/result types `E0201`,
  and may not reduce visibility (`E0107`). Constructors are never
  virtual and never inherited by name: `new C(...)` always runs `C`'s
  constructor, which chains `super(...)` explicitly (first statement)
  or by an inserted zero-argument `super`.
- **Identity equality:** `==`/`!=` compare references when one side's
  class is assignable to the other's (`classIdentity`, checked before
  nominal equality); unrelated class types are `E0202`, and comparing
  a class with a non-class is `E0202` as usual. Classes are not
  printable (`E0213`), not valid conditions (`E0203`), and not valid
  map keys (`E0202`).
- Inheritance cycles are `E0219`; a class name colliding with another
  class/struct/builtin/function is `E0103`; redeclaring an inherited
  field is `E0103`.
- **Static members**: `static` fields and methods belong to
  the class itself, not to instances. They live in a separate
  namespace — a static field never appears in the instance layout,
  and a static method never enters the method table (no virtual
  dispatch, no interface satisfaction). The class name used in
  `C.m` / `C.f` position resolves to the declaring class walking up
  the superclass chain; the class name there is a *type*, not a value
  (`let x = C;` is still `E0209`). Static fields are zero-initialized
  before `main`, may not be `void` (`E0201`) or hold a class
  reference (`E0201` — the non-NULL invariant has no initializer to
  uphold it), and may not shadow a visible field (`E0103`). A static
  and an instance member of the same name in one class collide
  (`E0103`); across a hierarchy the static simply hides the other
  namespace's member, with call sites resolving to the declaring
  class.
- **Abstract classes**: an `abstract` class marks methods
  whose signature has no body. The type is never constructible
  (`new A` → `E0201`) but is a normal class type for assignment,
  parameters, and interface satisfaction. Its abstract signatures
  take part in override checking exactly like concrete ones (result
  mismatch `E0201`, arity `E0204`, visibility `E0107`), and a
  concrete class must supply every inherited abstract method
  (`E0204` at the class declaration — a concrete implementation
  anywhere in its own chain satisfies the requirement).

## Interface types

```jsgo
interface UserRepository {
    count: int32;
    findById(id: int32): User;
    save(user: User): void;
}

interface AuditedRepo extends UserRepository {
    lastAudit: string;
}

class MemoryRepo implements AuditedRepo { ... }

let r: AuditedRepo = new MemoryRepo;   // implicit wrap
let u: UserRepository = r;               // widening, re-stamped
```

- An interface is a **structural type**: named properties with types
  plus a name → signature map of methods, with no bodies and no
  instances (properties and `extends` added by). One name
  is a property or a method, never both (`E0103`). Interfaces share
  the struct/class namespace (`E0103`), and their members declare no
  body and no access modifier (`E0011`); interface names are never
  values (`E0209`), never constructors (`E0011`), and never
  static-call targets (`E0011`).
- **`extends A, B`** inherits every property and method of the listed
  interfaces, recursively: the child's flattened member set is the
  union of its parents' and its own, and the child's own declaration
  wins on a shared name. An unknown or non-interface target is
  `E0104`; an inheritance cycle — including an interface extending
  itself — is `E0219`, the code a class inheritance cycle uses.
- **Satisfaction is structural** and public-only on both sides: a type
  satisfies an interface when every interface property and method is
  present with identical types (and, for members that live on a
  class, public visibility). An explicit `implements` clause is a
  *checked assertion* on top of the same rule — missing member
  `E0204`, arity `E0204`, signature or property-type mismatch
  `E0201`, unknown or non-interface target `E0104`. A `private` or
  `protected` member never satisfies an interface (`E0107` on a
  clause, `E0201` structurally), because interface members are always
  public. Only classes and other interfaces can satisfy; value structs
  cannot.
- **Assignment:** a class value is assignable to any interface it
  satisfies (checked in both directions — the class's static type at
  the assignment site is what must satisfy), and an interface value is
  assignable to any *wider* interface it satisfies, which is exactly
  what `extends` guarantees. The reverse — narrowing to an interface
  that declares members the value's type lacks — is rejected. There is
  no conversion syntax; widening re-derives the target interface's
  metadata from the same wrapped object. Interfaces participate in
  every coercion site: `let`/`const`, assignments (including
  multi-assign), call arguments, `return`, struct/class fields, array
  elements, map values, and `defer` captures.
- **Dispatch and properties:** an interface value carries the wrapped
  object plus the interface metadata stamped from the *static class*
  the value was wrapped at. Method slots hold root-casting
  trampolines, so dispatch runs through the wrapped instance's own
  vtable and a subclass override wins through the interface even when
  the static type is an ancestor (dynamic dispatch). Property
  slots hold the byte offset of the property measured from that same
  class, so a property read or write touches exactly the slot the
  class itself would — including on a subclass instance, whose fields
  share the static class's root-first prefix.
- **No null interface value:** an interface-typed `let` without a
  value is `E0218`, and an interface-typed field must be assigned by
  a top-level `this.f = ...;` in its constructor (`E0210`). `new I`
  is `E0201` (an interface is not constructible).
- **Restrictions:** interfaces are not comparable (`E0202`), not
  printable (`E0213`), not valid conditions (`E0203`), and not valid
  map keys (`E0202`); reading a member an interface does not declare
  is `E0216`, whose note lists the properties and methods it does
  declare (a method must be called with parentheses). An empty
  interface is legal and satisfied by every class.

## Generic types

A plain function, struct, or interface may declare type parameters.
The declaration is a *template*: it exists only at
its call sites and type mentions. Each distinct application —
`Box<int32>`, `Box<string>`, `Pair<int32, string>` — **materializes
once** as an ordinary nominal type with the substitution applied, and
the template body is checked once more per instance with concrete
types (monomorphization: specialized clones, no runtime dictionaries).

- **Nominality and invariance.** An instance is a distinct nominal
  type per argument list: `Box<int32>` is unrelated to
  `Box<string>`, and assignment, argument passing, and returns require
  identical spellings (see Struct types) — there is no covariance,
  e.g. `Container<int32>` does not accept a `Container<string>`.
- **Inference.** A call without explicit arguments binds each
  parameter by unifying the written parameter types with the argument
  types: `ident("hi")` binds `T = string`, `Pair(a, b)` binds both
  parameters, and `Box(5)` binds the constructor's field type. A
  parameter no argument mentions must be written explicitly
  (`E0222` naming it), a structural mismatch (`[]T` against `int32`)
  is `E0222` naming the parameter and the pattern, and a mismatch in a
  parameter type that mentions no parameter is an ordinary `E0201`.
- **Abstract checking.** A template body is checked once with type
  parameters behaving as permissive unknown types: name, arity, and
  definite-assignment errors still report, while assignments,
  conversions, and operator uses through `T` wait for an instance.
  This is what keeps a template that is never instantiated from
  producing false positives, and what lets each instance catch real
  errors (`widen("x")` where the body does `int64(v)` reports
  `E0208` on the `string` instance).
- **Instances everywhere.** An instance type is usable in every type
  position: annotations, `[]T` and `map[K]V` element/key/value slots,
  parameters, results, struct fields, interface-typed slots, and
  `implements` clauses (`class IntBox implements Container<int32>`).
  A template mentioned without arguments is `E0214` in a type
  position, a template in value position is `E0214`, and a generic
  function named as a type is `E0214`.
- **Structural checks.** Instances participate in the by-value cycle
  check like any struct (`E0219` for
  `struct Node<T> { next: Node<T>; }`), and a method receiver may
  name an instance (`function (b: Box<int32>) ...`) while an unbound
  template receiver — bare `Box`/`*Box` or `Box<T>` — is rejected with
  the targeted `E0011`; methods themselves are never generic.

## Function types

A function type spells a parameter list and an optional result:

```jsgo
function(int32, string): bool
function: void          // empty parameter list; omitted result = void
function(function(int32): int32): int32
```

- **Positions:** anywhere a type is allowed — parameters, results,
  `let` annotations, struct/class (static) fields, array element
  types, and map value types. Map *keys* are `E0202`.
- **Assignability is invariance:** two function types are assignable
  only when their parameter and result types are identical
  (`function(int32): bool` and `function(int64): bool` are unrelated) —
  no contravariance/covariance.
- **Restrictions:** function values are not printable (`E0213`), not
  comparable (`E0202`), not valid conditions (`E0203`), and not map
  keys (`E0202`).
- **Values:** a plain function name used where a function type is
  expected (`let f = add;`) becomes a value; calls through any
  function-typed expression are checked like named calls
  (`E0204` arity, `E0201` argument types, `E0212` when the callee is
  not a function).
- **Zero value:** the unset pair `{NULL, NULL}`; calling it is a
  checked runtime error (`call of an unset function value`, exit 1) —
  never a crash.
- **Literals:** the only anonymous-function syntax is the arrow
  (`(a: int32): int32 => expr` or a block body); there is no
  `function(...) {...}` literal.

### Arrow functions

```jsgo
const add = (a: int32, b: int32): int32 => { return a + b; };
let sq = (x: int32): int32 => x * x;          // expression body
let hi = (name: string) => "hi " + name;      // result inferred
let inc: function(int32): int32 = x => x + 1;     // bare param from context
```

- **Parameters** in the parens form are annotated, except a lone bare
  `(x)` or `x =>`, whose type comes from the expected function type;
  with no context it is `E0011` (help: annotate). Multiple
  unannotated parameters are `E0006`; a `void` parameter is `E0006`.
- **Result precedence:** an explicit `: T` wins and is checked against
  the body (`E0201` on mismatch); otherwise an expression body
  infers the result; otherwise an expected function type supplies it;
  otherwise a block body is `void` (missing return on a non-void
  result is `E0205`, synthesized signature named "arrow function").
- **Capture by value:** an arrow snapshots every named local visible
  at the arrow into a heap environment when it is *created* — the
  closure keeps seeing the value from that moment, while the outer
  binding keeps changing. Primitives and structs are copied;
  array/map/class/interface handles are copied as references, so the
  shared referent (e.g. a pushed-to array) is still mutated in
  place. `this` is captured like any local. Per-loop-iteration
  captures behave like JavaScript `let` (each iteration snapshots its
  own value).
- **Captured variables are read-only inside the body:** assignment to
  a captured local is `E0011` (what the body reads is the snapshot).
  A variable is not in scope inside its own initializer, so
  `let f = ... f ...` cannot see `f` (self-reference is an unknown
  identifier error, not a silent zero).
- **Lowering:** two-word type-erased pair `{f_fp, f_env}` — see
  Named-function references pair with a
  generated thunk (no capture); closures pair with the synthesized
  `closure.N` function and its `(closure env N)` struct.

## Multi-result tuples and `Result`

Two synthesized nominal types complete the core:

```jsgo
function swap(a: int32, b: int32): (int32, int32) { return b, a; }

let x, y = swap(1, 2);              // one variable per component
x, y = y, x;                        // parallel: values read first
let t: (int32, int32) = swap(3, 4); // tuples can be annotated
function take(v: (int32, int32)): int32 { ... }  // and passed as parameters

function divide(a: float64, b: float64): Result<float64, Error> {
    if (b == 0.0) { return Error("Division by zero"); }
    return Ok(a / b);
}
```

- A **result list** `(A, B)` names a tuple of its components:
  nominal, identified by the canonical spelling of its component
  types (`(int32, string)`). Components may be any non-`void` type.
  `return a, b;` builds one, `return f;` forwards a matching one,
  `let a, b = f;` and `a, b = f;` split one into its components
  (field reads, after **all** right-hand side values are evaluated —
  so `a, b = b, a;` swaps). Counts must match (`E0204`), component
  types must be assignable (`E0201`), and a tuple in any
  single-value position — an inferred single `let`, a single
  `return`, a value-list element — is `E0220` with both counts named.
- Tuples inherit struct rules: they compare field-wise like any
  struct but are not printable (`E0213` — a multi-result
  transport is not user data), not valid conditions (`E0203`), and
  take no user methods (`E0104` — their names are not valid C
  symbols).
- **`Result<T, E>`** has fields `ok: bool`, `value: T`, `error: E`.
  `Ok(v)`/`Error(msg)` type only where the expected type is that
  Result (`E0221` otherwise; payload mismatch `E0201`; arity `E0204`).
  Consumption is explicit: `r.isOk` (bool), `r.value` (T),
  `r.error` (E) — builtin methods lowered as field reads; the
  fields read directly too. `value` on a failed Result is the zero
  value of `T` (documented; gate on `isOk`).
- `Error` is a **builtin alias of `string`**: `Result<float64,
  Error>` and `Result<float64, string>` are the same type (the
  canonical name resolves the alias), and `struct Error` / `class
  Error` are `E0103`. Structured error types remain blocked: a failed
  `Result` materializes a zero value, and class references are never
  null (a class payload is `E0201`).
- Struct cycles mediated by a tuple or a Result are `E0219` — the
  cycle check walks resolved field types, not just source names.

## Concurrency handle types

`Task<T>`, `Channel<T>`, and `Context` are builtin
handles synthesized like `Result<T, E>`:

```jsgo
async function square(n: int32): int32 { return n * n; }

let t: Task<int32> = square(7);              // an async call yields Task<T>
let ch: Channel<int32> = Channel<int32>(8);  // builtin construction
let ctx: Context = newContext;             // root context
```

- **`Task<T>`** is produced only by calling an `async` function — the
  written result is the logical result `await` yields (`void` is
  allowed: `Task<void>`). Direct construction is `E0011` ("tasks are
  produced by calling an async function"), `Channel`/`Task` are
  reserved top-level names (`E0103`), and a task has **no methods**:
  any `t.foo` is `E0216` with a help pointing at `await t`, which
  unwraps `Task<T>` to `T` — only inside an async function and only
  on a `Task` (`E0011` otherwise).
- **`Channel<T>`** is a builtin handle built by `Channel<T>`
  (unbuffered) or `Channel<T>(n)` (buffer capacity `int32`): exactly
  one type argument (`E0214`; a missing one is `E0011` with the
  spelling in the help), the element type may not be `void`
  (`E0201`), and other argument counts are `E0204`. Its fixed method
  set is `.send(v)` (one argument assignable to `T`), `.recv`
  (→ `T`), `.close`, `.isClosed` (→ `bool`), `.length`
  (→ `int32`); any other method is `E0216` listing them.
- **`Context`** is a builtin token type built by `newContext`
  (root) or `newContext(parent)` (one `Context` argument; `Context(...)`
  as a constructor is `E0011`). Its method set is `.cancel` and
  `.isCancelled` (→ `bool`); a `nil` Context is a valid,
  never-cancelled root. Passed as a task's **first argument** it
  becomes that task's context (`go worker(ctx)`), which is how `await`
  resumes with the zero value on cancellation.
- Like classes, interfaces, and tuples, the three handles are **not
  printable** (`E0213`); as conditions they are `E0203` (conditions
  must be `bool`).

## Inference

- `let x = 5;` infers `int32`; `let f = 1.5;` infers `float64`;
  `let s = "hi";` infers `string`; `let b = true;` infers `bool`.
- An explicit annotation (`let x: int64 = 5;`) always wins and is
  checked against the initializer.
- Parameters and results are never inferred; they are always written.
- Inference never binds a multi-value result implicitly: `let only =
  pair;` where `pair` returns `(int32, string)` is `E0220`. List
  the variables (`let a, b = pair;`) or annotate the tuple
  (`let t: (int32, string) = pair;`). A `Result`, unlike a tuple,
  is an ordinary single value and binds normally
  (`let r = divide(...)`).
- Generic calls infer **type arguments** alongside the value type:
  `ident("hi")` binds `T = string` from the argument, `Box(5)` binds
  the constructor's field type, and the result type is the
  instantiated one. A parameter no argument mentions requires the
  explicit form (`f<int32>(...)`) — otherwise `E0222` (see Generic
  types).

## Literal width allowance

An untyped integer literal is accepted by any integer type it **fits
in** (this is a constant-literal allowance, not a conversion):

```jsgo
let a: uint16 = 400;         // OK: 400 fits in uint16
let b: int8 = 300;           // error E0211: out of range
let f: float64 = 10;         // error E0201: integer literal is not a float
```

In binary expressions an untyped literal takes the type of the other
operand: `uint32_var + 5` is fine because `5` is typed `uint32` by
context. Two typed operands of different types are an error
(`int32_var + int64_var` → `E0202`).

## Assignability

`value of type V` is assignable to type `T` iff `V` and `T` are
identical, **or** both are classes and `V` inherits from `T`
(upcasting a subclass instance), **or** `T` is an interface
and `V` is a class (or subclass) that satisfies `T` structurally
(wrapping the reference). There are no other widening or
narrowing rules; convert explicitly: `int64(x)`, `float32(f)`. Numeric
conversions and `string(value)` (numbers and bools only) are the
conversions that exist today (`E0208` for `int32("x")` and
`string("x")`). Identity of the synthesized types follows the same
rule: two result lists match component-wise by spelling, and two
`Result` types match after resolving their arguments —
`Result<float64, Error>` is identical to `Result<float64, string>`
(the `Error` alias). Function types are invariant: assignable only
when spelled identically (§Function types). Pointer types are
invariant like scalars — assignable only when spelled identically
(`*int32` is not `*int64`) — with one addition: `nil` is assignable to
any `*T` when a `*T` is expected. Generic instances are
nominal and invariant the same way: `Box<int32>` is assignable only to
`Box<int32>`, never to `Box<string>` (see Generic types).

## Definite assignment

A `let` binding declared without an initializer must be assigned on
every path before it is read (`E0210`). Parameters count as assigned on
entry. `const` bindings must be initialized at declaration and can
never be assigned (`E0206`). A **field assignment** (`p.x = v`) reads
the base aggregate, so the base variable must be assigned first:
construct the whole struct (`let p = Point(0, 0);`) before writing its
fields.

## Conditions

`if`, `while`, and `for` conditions must have static type `bool`
(`E0203`). There is no truthy/falsy coercion — function, interface,
and struct values are never conditions either:

```jsgo
let username: string = "Ada";
if (username) { ... }              // error E0203
if (username.length > 0) { ... }   // OK: .length is int32
let f = (x: int32): int32 => x;
if (f) { ... }                     // error E0203
```

## Operators

| Operator | Valid operands | Result |
| --- | --- | --- |
| `+` | identical numerics, or string + string | operand type / string |
| `- * / %` | identical numerics (`%` integers only) | operand type |
| `< <= > >=` | identical numerics or identical strings | `bool` |
| `== !=` | identical ints, floats, bools, or strings; identical pointers (and `nil` against a typed pointer) | `bool` |
| `&& \|\|` | bool, bool | `bool` |
| unary `- +` | numeric | operand type |
| unary `!` | bool | `bool` |
| unary `*` | pointer (`*T`) | pointee `T` |
| unary `&` | an addressable place: variable, rooted field chain, array element | `*T` of the place's type |

`"2" + 2`, `true + 1`, and `5 == "5"` are all compile errors — this is
a core acceptance criterion of the specification and is enforced by
`TestSpecStringPlusNumberRejected`.

## Object identity and subtyping

Class references have **identity**: `==`/`!=` compare whether two
variables point at the same instance, and are allowed when one side's
class is assignable to the other's (a subclass instance against its
supertype); unrelated classes are `E0202`. Subtyping today is
the single-inheritance class chain: a subclass value is assignable to
any superclass variable, and method dispatch is virtual through the
chain (see "Class types"). Interface satisfaction is a second,
structural relation (see "Interface types"): a class value flows into
any interface it satisfies, an interface value flows into any wider
interface it satisfies, and dispatch goes through the interface's
method slots into the instance's vtable. Interfaces
themselves are never comparable, so `==` never applies across them
(`E0202`).

## Overflow and precision

Integer arithmetic wraps at the type's width at runtime (C semantics in
the current backend). This is a documented, deliberate choice pending a
language-level decision. Floats follow IEEE-754
(`float32` computations are performed in `float` precision, `float64`
in `double`).
