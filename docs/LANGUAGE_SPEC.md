# JSGo Language Specification

This is the normative reference for the JSGo language **as
implemented today**. Features that are designed but not yet
implemented are clearly marked as such.

Nothing is specified here that is not tested.

## 1. Source files

- Source files use the `.jsgo` extension.
- Encoding is UTF-8. Non-ASCII characters are allowed only inside
  string literals and comments.
- Statements are terminated by `;`. The semicolon is optional when the
  statement is followed by `}` or end of file (automatic semicolon
  insertion at line boundaries, like JavaScript).
- `//` line comments and `/* ... */` block comments are supported.
  Block comments do not nest. A `/** ... */` block written directly
  above a declaration is a doc comment (§1.2).

### 1.1 Imports

```jsgo
import "strings";       // standard library module
import "std/strings";   // canonical prefix: the same module

import "./util.jsgo";                    // pull in a file (no bindings)
import { helper, Point as P } from "./util.jsgo";   // named bindings
import * as util from "./util.jsgo";      // namespace
```

Import declarations come first: they must appear before any
top-level declaration. In v0.1 the resolvable modules are the
seven standard packages of §6 (`"strings"`, `"strconv"`, `"math"`,
`"errors"`, `"os"`, `"filesystem"`, `"path"`), each also accepted under
the canonical `"std/"` prefix; any other path is an error (`E0106`,
with a note listing the known modules). A module may be imported once
per file (`E0103` on duplicates).

**File imports.** An import path starting with `./` or `../` and ending
in `.jsgo` refers to a source file in the program:

- The file is pulled into the build automatically, transitively and
  cycle-safe — `jsgo run main.jsgo` compiles every file main names,
  and the whole-project mode already compiles them all. A missing file
  is `E0108`; a path that is neither a known module nor a `./`-file is
  still `E0106`.
- `import "./f.jsgo";` imports for side effects only (validation and
  pull-in; no bindings).
- `import { a, b as c } from "./f.jsgo";` binds `a` and (under the
  alias `c`) the target file's top-level declarations — plain
  functions, structs, classes, and interfaces (`E0105` if the file has
  no such declaration). Aliases may be used in every position,
  including types (`let p: P = ...`, `new C`, `let f = plus;`), and
  are program-wide like every other top-level name; a local variable
  of the same name shadows the alias. An alias that collides with an
  existing declaration, a builtin type, a builtin function
  (`print`/`println`/`Ok`/`Error`), or a module imported by the same
  file is `E0103`.
- `import * as ns from "./f.jsgo";` registers a file-scoped
  namespace: `ns.member(...)` calls the file's top-level declaration
  (`E0105` otherwise). Namespaces are like stdlib modules — visible
  only in files that imported them — and may be used only as call
  callees; using a namespace or its member as a value is `E0209`.
- `from` and `as` are contextual keywords; a malformed import form
  (default imports, missing `from`/`as`) is `E0006`. Named or
  namespace forms on a standard-module path are `E0108`.

Cross-file calls remain visible without any import (the lenient model
of §6); file imports add pull-in, documentation, and validation —
they never hide a name that used to resolve.

### 1.2 Doc comments

A declaration's **doc comment** is the comment block immediately
above it:

- a contiguous run of `//` lines with no blank line between the
  block and the declaration, or
- a `/** ... */` block comment ending on the line directly above it.

The same rule applied to the comment block that opens a file whose
first token is an import gives the file its header; a file that
opens with a declaration has no header, because its opening comment
documents that declaration. Text extraction strips `//` plus one following
space (relative indentation survives, so an indented code block in a
comment keeps its shape) and, in a `/** */` block, the `/**`, `*/`,
and the leading `*` of each line; blank lines at the edges are
dropped while interior blank lines keep paragraphs together. A plain
`/* ... */` block is ordinary commentary and never documentation; a
comment that shares its line with code (a trailing comment) belongs
to that line and documents nothing; a blank line between comment and
declaration severs the pair. `jsgo doc` renders these comments into
the project's API reference; nothing else in the language reads
them.

## 2. Declarations

Top-level declarations are functions, structs, classes, and
interfaces:

```jsgo
function name(param: type, param: type): result {
    ...
}

struct Point {
    x: int32;
    y: int32;
}
```

- The return type may be omitted and defaults to `void`. A result
  list of two or more types — `function swap(a: int32, b: int32):
  (int32, int32)` — makes the function multi-result: `return a, b;`
  supplies one value per type, `return f;` forwards a call with a
  matching list (counts `E0204`, components `E0201`,
  multi-value in a single-value position `E0220`).
- Parameters require type annotations; a parameter may use a result
  list type (`v: (int32, int32)`) or `Result<T, E>`.
- Functions may call each other regardless of declaration order.
- `async function f(...): T` declares an asynchronous
  function: calling it starts a task immediately and the call yields
  `Task<T>` — the body still returns `T`, and omitting the result
  gives a `Task<void>`. `async` is valid only before a top-level
  function, never on a method (`E0011`), and an async `main` is
  `E0500` (`main` runs as task 0).
- A struct field list separates fields with `;` (the final one is
  optional). Field types may reference structs declared later, in any
  file. A struct name may not collide with a builtin type, a function,
  or another struct (`E0103`), and a struct may not contain itself by
  value (`E0219`; `[]T`/`map[K]V` handles break the cycle legally).
- Structs are **nominal value types**: identity is the name, and
  assignment/argument passing/return copy the value (§5).
- Construction is positional: `Point(1, 2)` — arity (`E0204`) and
  field types (`E0201`) are checked in declaration order. Using the
  struct name as a value instead of calling it is `E0209`.
- `==`/`!=` compare two identical structs **field by field, deeply**:
scalars and strings by value, nested structs
  recursively, and reference fields (`*T`, arrays, maps, classes,
  functions) by identity — an interface field by the object it wraps.
Interfaces stay
  non-comparable (`E0202`) and a struct is never a condition
  (`E0203`).
- Printing a struct writes a JS container — `{x: 1, name: "hi"}` —
  when every field is printable; a struct holding a
  class, interface, array, map, pointer, function, or tuple field is
  `E0213`.

Methods attach to a struct through a receiver:

```jsgo
function (p: Point) scale(k: int32): Point {
    return Point(p.x * k, p.y * k);
}
```

- The receiver must be a struct type (`E0104`); the receiver name is
  in scope for the body like a leading parameter.
- The receiver of a **value-receiver** method is a **copy** (Go
  value-receiver semantics): writes to receiver fields inside the
  method do not escape. Update state by assigning fields at the call
  site or returning new values — or declare a **pointer receiver**,
  `function (p: *Point) shift(dx: int32) { ... }` (§5), which mutates
  the pointee. Receiver kinds match exactly at call sites: a value
  receiver is called on values, a pointer receiver only on `*T`
  (`E0201` on mismatch — write `(*q).m` or `(&x).m` explicitly);
  classes reject `*C` receivers (`E0104`) because class references are
  already pointers.
- Method calls use the same syntax as field access: `p.scale(2)`.
  Unknown methods report the available ones (`E0216`); duplicate
  methods on one struct are `E0103`.
- A method named `main` is not an entry point (`E0500` checks only
  plain functions).

Classes declare reference types with fields, a constructor, methods,
and access modifiers:

```jsgo
class User extends Base {
    private id: int32;
    public name: string;
    constructor(id: int32, name: string) {
        this.id = id;
        this.name = name;
    }
    public getName: string {
        return this.name;
    }
}
```

- Members are `[access] field: T;`, at most one `constructor(...)` —
  which may not declare a return type (`E0011`) or use `return`
  (`E0011`) — and methods `[access] name(params)[: R] { ... }`. The
  access modifier is `public` (default), `private`, or `protected`;
  violating it is `E0107` (private: declaring class only; protected:
  declaring class and subclasses).
- `new C(args)` allocates and constructs (constructor arity `E0204`,
  argument types `E0201`); calling `C(...)` directly is `E0011` and
  using the class name as a value is `E0209`. `this` outside a class
  is `E0011`.
- `extends` gives single inheritance. Overrides must match the parent
  signature exactly (`E0204`/`E0201`) and may not reduce visibility
  (`E0107`); inherited fields may not be redeclared (`E0103`);
  cycles are `E0219`. A class without a constructor is rejected when
  its parent's constructor takes parameters (`E0204`), and a class
  with class-typed fields but no constructor is `E0210`.
- `super(...)` constructs the parent (first statement when written;
  otherwise an implicit zero-argument `super`), and
  `super.method(args)` calls the parent implementation. Both require
  `extends` (`E0011`).
- Class references are never null: a valueless class-typed `let` is
  `E0218`, and constructors must assign their own class-typed fields
  at the top level (`E0210`). `==`/`!=` compare identity (related
  classes only; unrelated `E0202`); classes are not printable
  (`E0213`), not conditions (`E0203`), and not map keys (`E0202`).
- Interfaces are implemented (properties and `extends`
  added by): `interface Name { count: int32; m: T; }` declares
  a structural type — named properties with types plus methods — and
  `interface Child extends Parent { ... }` inherits every member of
  the listed interfaces recursively (unknown/non-interface `E0104`,
  a cycle or self-extension `E0219`). `class C implements I1, I2`
  adopts one or more interfaces. Satisfaction is structural and
  public-only on both sides — the clause is a checked assertion:
  missing member `E0204`, signature or property-type mismatch
  `E0201`, unknown or non-interface name `E0104`, private/protected
  member `E0107` (with a clause) or `E0201` (structurally). A
  value of one interface is usable wherever a wider one it extends is
  expected; the reverse is rejected. Assigning a class value to an
  interface-typed slot wraps it implicitly, interface property reads
  and writes go through the property's byte offset, and interface
  calls dispatch dynamically through the instance's vtable, so
  subclass overrides win. Interface values are never null (`E0218`),
  not printable/comparable/conditions/map keys
  (`E0213`/`E0202`/`E0203`/`E0202`), and `new I` is `E0201`. Full
  rules: `TYPE_SYSTEM.md` §Interface types.
- **Static class members**: `static` fields hold class-wide
  storage zero-initialized before `main` (never `void` `E0201`, never
  a class reference `E0201`, never shadowing a visible field
  `E0103`), and `static` methods have no receiver — called as
  `C.m(args)`, never through an instance (`E0216`), with `this`/
  `super` rejected (`E0011`). Statics never participate in virtual
  dispatch or interface satisfaction (`E0204` when that is all a
  class offers); a same-named static and instance member of one class
  collide (`E0103`).
- **Abstract classes**: `abstract class A` declares
  signature-only methods (`E0011` for misuse such as an abstract
  method in a concrete class or an abstract constructor) and is never
  constructible (`new A` → `E0201`). Every concrete subclass must
  implement the inherited abstract methods (`E0204`); `super.m` on
  an abstract parent method is `E0011` (no implementation exists).
  Full rules: `SYNTAX.md` §Static members and abstract classes.

Plain functions, structs, and interfaces may be **generic**.
A parameterized declaration is a *template*; every use materializes a
concrete instance (monomorphization — no runtime dictionaries):

```jsgo
struct Pair<A, B> {
    first: A;
    second: B;
}

function pair<A, B>(a: A, b: B): Pair<A, B> {
    return Pair<A, B>(a, b);
}

struct Box<T> {
    value: T;
}

interface Container<T> {
    get: T;
}
```

- Type parameters `<T...>` sit between the name and the parameter
  list (functions, interfaces) or before the body (structs). They may
  be declared only on plain functions, structs, and interfaces:
  classes, constructors, and methods reject them (`E0011`), and so
  does the entry point (`main<T>` → `E0011`). The list may not be
  empty (`E0006`), a parameter may be named once and never after a
  builtin type (`E0103`), and a base name is claimed by exactly one
  declaration, generic or not (`E0103` — `function f` and
  `function f<T>` collide).
- A call site supplies type arguments explicitly —
  `pair<int32, string>(a, b)` — or lets inference bind them from the
  arguments — `Box("text")` binds `T = string` (§4.1). A parameter no
  argument mentions, or a shape mismatch such as `[]T` against
  `int32`, is `E0222` naming the parameter; a mismatch in a parameter
  type that mentions no parameter is an ordinary `E0201`. A `<...>` is
  read as type arguments only when its closing `>` is immediately
  followed by `(` — everywhere else `<` stays the comparison operator,
  so `i < len(arr)` parses unchanged. Type arguments on a target that
  is not a template, or in the wrong count, are `E0214`.
- Instances are ordinary nominal types (§5): `Box<int32>` may appear
  in every type position (`let b: Box<int32> = ...`, `[]Box<int32>`,
  `map[string]Box<string>`, parameters, results, fields). A bare
  template name in a type position or a template in value position is
  `E0214`, and a generic function named as a type is `E0214`.
- Templates are checked once in the abstract — type parameters behave
  as permissive unknown types, so name and arity errors in a body that
  is never instantiated still report while conversions and assignments
  through `T` wait for an instance — and each materialized instance is
  then checked again with real types (a bad conversion inside
  `widen<string>` reports `E0208` normally). Each distinct argument
  list materializes once, keyed canonically (`f<int32>`).
- A generic interface is implemented with a concrete clause —
  `class IntBox implements Container<int32>` — and used through
  interface-typed slots (§2 above); calling a generic interface is
  `E0011`. Methods stay non-generic (no type parameters on a
  receiver's method), but a receiver may name an instance type:
  `function (b: Box<int32>) twice: int32 { ... }`. A receiver that
  *is* the bare template — `Box` or `*Box` — or reuses the template's
  own type parameter — `Box<T>` — is `E0011` with concrete-receiver
  help (one diagnostic, no cascade); only materialized instances and
  plain structs serve as receivers.
- A materialized instance joins the by-value cycle check like any
  struct (`E0219` — `struct Node<T> { next: Node<T>; }` is reported
  on `Node<int32>`).
- Constraints on type parameters (`where`-style) are not in the
  language yet — reserved for future work.

The entry point of an executable is a top-level `function main` with no
parameters and result `void` (`E0500` otherwise).

## 3. Statements

```jsgo
let x: int32 = 0;        // mutable binding, type inferred if omitted
const s: string = "hi";  // immutable binding
let a, b = f;          // multi-value declaration (types from the values)
let a, b = 1, "two";     // or from a value list
x = 1;                   // assignment
x += 1;                  // compound assignment: + - * / %
a, b = b, a;             // parallel assignment — a real swap
x++;  x--;               // desugar to x += 1 / x -= 1
p.x = 5;  p.y += 1;      // struct field assignment (chains: p.a.b)
if cond { ... } else if cond { ... } else { ... }
while cond { ... }
for (init; cond; post) { ... }   // parentheses optional
switch x {               // tag may be parenthesized; no fallthrough
case 0, 1: ...
default: ...
}
break; continue; return value;
return a, b;             // one value per result-list type
defer cleanup;         // register a call for function exit (LIFO)
go worker(arg);          // spawn a task running a named call
select {                 // wait for one ready channel arm
case let v = ch.recv: ...
case ch.send(v): ...
default: ...             // optional: runs when nothing is ready
}
{ ... }                 // nested block with its own scope
```

- Conditions of `if`/`while`/`for` must have type `bool`. There is no
  truthy/falsy coercion (`E0203`).
- Multi-value `let` and `a, b = ...` require matching counts
  (`E0204`) and assignable component types (`E0201`); all right-hand
  side values are evaluated before any target is written. Targets are
  simple variables — compound assignment on a value list, an
  annotation on a multi-value `let`, and target lists containing
  `a[i]`/`p.x` are `E0011` (the parser suggests the fixed spelling).
  Inference never binds a multi-value result to a single variable
  (`E0220`); annotate the tuple instead: `let t: (A, B) = f;`.
- Every variable must be assigned before it is read (`E0210`,
  definite-assignment analysis). A field assignment requires the base
  variable to be assigned first — construct the whole struct before
  writing its fields.
- `break`/`continue` outside a loop (or, for `break`, a switch) are
  errors (`E0215`).
- Code after a `return`/`break`/`continue` produces warning `W0001`.

### 3.1 `switch`

- The tag is evaluated exactly once.
- Case values must be compile-time constant literals assignable to the
  tag type; the tag type must be `int*`, `uint*`, `float*`, `string`,
  or `bool` (`E0202`).
- Non-constant case values are `E0217`; type mismatches are `E0201`;
  duplicate values or arms are `E0103`.
- **There is no fallthrough**: an arm's body ends with an implicit jump
  past the switch. `break` is optional and exits the switch, not the
  enclosing loop.
- All arms share one scope. A `switch` with a `default` arm where every
  arm returns satisfies the missing-return check; without `default` it
  does not.

### 3.2 `defer`

- `defer f(args);` registers the call for function exit. The target
  must be a call expression (`E0011`).
- Allowed only as a **direct statement of a function body** — not
  inside `if`, loops, `switch`, or nested blocks (`E0011`; the
  conditional subset needs the runtime defer stack — still future).
- Arguments and method receivers are evaluated **at the defer
  statement** and captured (Go semantics).
- Registered calls run at every exit — each `return` (after the return
  value is evaluated) and fall-off-the-end — in **LIFO** order.

### 3.3 `go`

- `go f(args);` spawns a task running a **named** call: a plain
  function, method, `static`, `super`, generic instance, or module
  function. A function value, print builtin (`print`/`println`),
  constructor, or handle/collection operation is
  `E0011` — there is no declaration for a spawn thunk to run.
- `main` runs as task 0; the program exits when every task has
  finished. A run in which every live task is parked is a runtime
  deadlock (`jsgo: deadlock: all tasks are parked`).

### 3.4 `select`

- `select { case let v = ch.recv: ... case ch.send(v): ... default: ... }`
  waits for one **ready** arm — chosen randomly when several are
  ready — and runs its body; a receive arm's `let` binding is scoped
  to that arm.
- Every arm must be a channel operation (`E0011`), a send arm cannot
  bind a value (`E0011`), and there is at most one `default` arm
  (`E0103`). `default` runs immediately when nothing is ready (making
  the select non-blocking); without it the task parks until an arm is
  ready.

## 4. Expressions

Literals: integer (`42`, `0xFF`, `0b1010`, `0o17`, `1_000_000`),
floating (`3.14`, `1e10`), string (`"hi\n"`), boolean (`true`, `false`),
array (`[1, 2, 3]`), map (`{"a": 1}`).

Operators, highest precedence first:

| Level | Operators |
| --- | --- |
| 1 | postfix `f(...)`, member `a.b`, index `a[i]` |
| 2 | unary `-`, `+`, `!`, `&`, `*`, `await` |
| 3 | `*` `/` `%` |
| 4 | `+` `-` |
| 5 | `<` `<=` `>` `>=` |
| 6 | `==` `!=` |
| 7 | `&&` |
| 8 | `\|\|` |
| 9 | assignment `=` `+=` `-=` `*=` `/=` `%=` |

- `&&` and `||` short-circuit (verified by evaluation order).
- `+` on two strings concatenates; mixing string and number is an error
  (`E0202`). Convert with `string(value)` (§4.1).
- `==`/`!=` require identical operand types and are defined for
  integers, floats, bools, and strings. Class references compare by
  **identity** when one side's class is assignable to the other's
  (§2; unrelated classes are `E0202`). Pointers compare by identity as
  well: `p == nil` / `p != nil` and pointer-vs-pointer (§5).
- Address-of and dereference: `&x` takes the address of a
  variable (local or parameter), a variable-rooted field chain
  (`&o.inner.v`), or an array element (`&a[i]`); map elements are not
  addressable (`E0212`) and `&*p` simplifies to `p`. Address-taken
  variables are lowered into GC boxes, so a returned or stored `&x`
  can never dangle (`MEMORY_MODEL.md`). `*p` reads the pointee and is
  an assignable target (`*p = v`, `*p += 1`); dereferencing `nil`
  aborts at runtime with `jsgo: runtime error: nil pointer
  dereference` (exit 1). There is no auto-dereference sugar: `p.f` on
  a `*T` is `E0216` whose help shows `(*p).f`.
- `nil` is a literal that is only ever a `*T`-typed value: it
  takes its type from an expected type — assignment, annotation,
  argument, return, or comparison against a typed pointer — and is a
  targeted error anywhere else. Class and interface values keep their
  non-NULL invariant (§2); `Result<T, E>` remains the optional-result
  tool.
- Indexing `a[i]` reads an array element or map entry; `a[i] = v`
  writes one (inserting for maps). The index must be an integer for
  arrays (`E0202`) and assignable to the key type for maps (`E0201`).
- Member access `a.b`: `.length` (type `int32`) on arrays, maps, and
  strings; module function calls such as `strings.contains(x, y)`; file-namespace
  calls such as `util.helper(x)` (§1.1); struct and
  class fields (`p.x`, `u.name` — access-checked, `E0107`); methods
  called with parentheses (`a.push(v)`, `m.has(k)`, `m.delete(k)`,
  `u.getName`); `super.field` and `super.method(...)` inside a class
  (§2). Anything else is `E0216`.
- Arrow functions (spec §5): `(a: int32, b: int32): int32 => { ... }`
  with a block body and explicit `return`, or `x => expr` with an
  expression body. Parameters are annotated except a lone bare
  `(x)`/`x`, which takes its type from the expected function type
  (`E0011` without one). The result annotation is optional when it can
  be inferred from the body. Arrows are values: assign them, pass
  them, return them. An arrow captures every visible local **by
  value** at creation; assigning to a captured variable is `E0011`.
  There is no anonymous `function` literal — arrows are the only
  anonymous-function syntax.
- `await t` unwraps a `Task<T>` to `T`, suspending the current
  task until the spawned call finishes. It is a unary operator that
  binds tighter than the binary ones (`await f + 1` is
  `(await f) + 1`) and is valid only inside an `async` function and
  only on a `Task<T>` (`E0011` otherwise). When the relevant
  `Context` is cancelled, `await` resumes immediately with the zero
  value of `T`.

### 4.1 Calls and conversions

- User function calls: `add(1, 2)`.
- Calls through function values: `f(args)` where `f` is any expression
  of function type (a local, struct/class field, static field, or
  index). Arity and argument types are checked exactly like named
  calls (`E0204`/`E0201`); calling a non-function is `E0212`. Calling
  an unset function value aborts at runtime with
  `call of an unset function value` (exit 1).
- Class construction: `new C(args)` — the class must exist and be a
  class (`E0201` for a struct), and the arguments must match its
  constructor (`E0204`/`E0201`); the result is a non-null reference
  (§2).
- Numeric type conversions: `int64(x)`, `float32(x)` — exactly one
  numeric argument (`E0208` otherwise).
- String conversion: `string(v)` accepts any numeric type or `bool`
  (`E0208` otherwise), e.g. `"count: " + string(n)`.
- Printing: `println(value)` and `print(value)` are builtins
  requiring no import — `println` writes
  `value` followed by a newline, `print` writes it with no trailing
  newline. Exactly one argument (`E0204`), and the argument must be
  printable (`E0213`). A user function, struct, or local of the same
  name shadows them; they cannot be taken as values (`E0101` with a
  builtin help).
- Generic calls: `f<int32>(args)`, `Box<int32>(args)`, and the
  inferred form `f(args)` (§2). Explicit type arguments parse only
  when the closing `>` is immediately followed by `(`, so `a < b`
  and `i < len(arr)` remain comparisons; wrong count or a
  non-template target is `E0214`.
- File-namespace calls: `util.helper(...)` where `util` is an
  `import * as util` namespace (§1.1) — resolves the target file's
  top-level declaration and is equivalent to calling it unqualified;
  anything else is `E0105`.

## 5. Types

`void`, `bool`, `int8`, `int16`, `int32`, `int64`, `uint8`, `uint16`,
`uint32`, `uint64`, `float32`, `float64`, `string`, the composite
types `[]T` (array, any nesting) and `map[K]V` (map; keys must be
scalar in literals), pointer types `*T` (any element type, nesting
allowed — `**Point`, `*[]int32`; `*void` is `E0006`), function types
`function(T1..., Tn): R` (an empty
parameter list is `function: void`; an omitted result means `void`),
declared struct types (`Point`), generic instance types (`Box<int32>`
— §2; instances are nominal and invariant, so `Box<int32>` and
`Box<string>` are unrelated types), declared
class types (`User`, `User extends Base`) — class types hold
non-null references (§2) — and the concurrency handles `Task<T>`,
`Channel<T>`, and `Context` (§3.3, §4, `TYPE_SYSTEM.md`).

Pointer types are values like scalars: assignable only when
spelled identically, with `nil` assignable to any `*T` where a `*T`
is expected. The zero value of a pointer-typed slot or field is
`nil`; pointers are not printable (`E0213`), not indexable, and have
no arithmetic. `&x`/`*p` and the addressability rules are in §4.

Function types are **invariant**: assignability requires identical
spelling (`function(int32): bool` is not assignable to `function(int64): bool`),
and they are otherwise values like any other — storable in locals,
parameters, results, struct fields, arrays, and map values. Function
types are not printable (`E0213`), not comparable (`E0202`), not valid
conditions (`E0203`), and not valid map keys (`E0202`). The zero value
is an unset pair whose call is a checked runtime error.

The synthesized nominal types complete the core:

- **Result lists** `(A, B)` — the type of a multi-result function.
  The components are ordinary types, so `(int32, string)` and
  `map[string](int32, int32)` are valid. Inference never binds a
  tuple implicitly (annotate: `let t: (A, B) = ...`); tuples are not
  printable (`E0213`) but compare field-wise like any struct.
- **`Result<T, E>`** — success/failure values for error handling
  (spec §7). `Ok(v)` and `Error(msg)` construct one where a Result is
  expected (`E0221` otherwise); `r.isOk`, `r.value`, `r.error`
  (and the fields `r.ok`, `r.value`, `r.error`) consume one.
  `Error` is a builtin alias of `string`, so `Result<float64, Error>`
  and `Result<float64, string>` are the same type, and `struct Error`
  or `class Error` is `E0103`.
- **`Task<T>`, `Channel<T>`, `Context`** — the builtin concurrency
  handles. A `Task<T>` comes only from calling an `async` function
  and is consumed by `await` (§4); `Channel<T>` is built by
  `Channel<T>([capacity])` and moves `T` values through
  `send`/`recv`/`close`/`isClosed`/`length`; `Context` comes from
  `newContext([parent])` and carries `cancel`/`isCancelled`. All
  three are not printable (`E0213`), not conditions (`E0203`), and
  take no user methods (`E0216`).

See [`TYPE_SYSTEM.md`](TYPE_SYSTEM.md) for the full rules. The core
guarantees: no implicit conversions of any kind, no truthy/falsy, no
reading uninitialized variables, explicit `T(value)` conversions only
between numeric types (plus `string(value)` from numbers/bools).
Structs are values (copied on assignment); arrays, maps, and class
references are references (never null).

## 6. Standard library

Seven modules are implemented (complete signature
tables and documented limitations in
[`STANDARD_LIBRARY.md`](STANDARD_LIBRARY.md)):

Printing needs no module: the builtins `println(value)` and
`print(value)` are available in every file without an import.

- `strings` — search/transform (`contains`, `indexOf`, `replace`,
  `replaceAll`, `toUpper`, `toLower`, `repeat`...), `split`/`join`,
  `trim` family, and JS-style `substring`, `charAt`, `padStart`,
  `padEnd` (19 functions).
- `strconv` — `parseInt`, `parseInt64`, `parseFloat` return
  `Result<T, string>`; `formatInt(n, radix)` formats in bases 2–36.
- `math` — 15 `float64` functions (`sqrt`, `floor`, `round` with JS
  tie-breaking, `sin`/`cos`/`tan`, `log`, `pow`, `pi`, `e`...).
- `errors` — `new(message)`, `message(e)`, `is(a, b)`; errors remain
  values (`Result`), never exceptions.
- `os` — `args` (arguments after the program name), `exit(code)`,
  `getenv(name)`.
- `filesystem` — `readFile` returns `Result<string, string>`;
  `writeFile`, `appendFile`, `exists`, `makeDir`, `remove` return
  `bool`.
- `path` — POSIX lexical helpers: `join`, `basename`, `dirname`,
  `ext`, `clean`, `isAbs`.

Module calls are checked against the fixed signatures (arity `E0204`,
argument types `E0201`, unknown member `E0105`); module names are not
first-class values (`E0209`). A class or struct may share a module's
name: the imported module wins for calls, the type wins when the
module is not imported. Any keyword may appear as a member name after
a dot (`errors.new`), following JavaScript's `obj.new` grammar.
`jsgo run file.jsgo a b` passes `a b` to the program (`os.args`).

## 7. Reserved but unimplemented syntax

The following are reserved keywords. Using them produces a targeted
`E0020` diagnostic with a planned-release note, never a silent misparse or
a confusing cascade: `chan` (the help points at `Channel<T>`),
`range`, `enum`, `type`. Template strings (backticks) are likewise
reserved (`E0020`). `func`, the former spelling of `function`, also
stays reserved: it always produces a targeted error with a `function`
help (`E0010` in declaration position, `E0006` in type position).
`package` is no longer part of the language and stays reserved the
same way: a leftover `package ...;` line yields one targeted `E0010`
("'package' is no longer part of JSGo") carrying a delete-this-line
help, and the rest of the file still parses.
Arrow functions (`=>`), `switch`, `case`,
`default`, `struct`, `defer`, the class family — `class`, `extends`,
`super`, `constructor`, `this`, `new`, `private`, `public`,
`protected`, `static`, `abstract`, the interface family —
`interface`, `implements` — and the concurrency keywords `async`,
`await`, `go`, `select` are implemented (see §2, §3, §4, §5).
`nil` (and the pointer syntax `*T`/`&`/`*p`) is implemented: it is a `*T`-only literal, never an `E0020` keyword.

## 8. Diagnostics

Errors carry stable codes: `E00xx` syntax, `E01xx` names and imports,
`E02xx` types, `E05xx` build/backend, `W0xxx` warnings, `E0020` planned
feature. Every diagnostic carries a source location and, where
possible, a note or help text. Compilation never proceeds to code
generation when errors exist.

The `W0xxx` family is `W0001` (unreachable code), `W0002` (unknown
escape sequence), and — since the linter — `W0003`
unused variable, `W0004` unused parameter, `W0005` unused import,
`W0006` shadowed declaration, `W0007` empty block, `W0008`
always-true/false condition, and `W0009` redundant operation.
`jsgo lint` reports them as warnings (exit 0; `--strict` promotes
them to failures for CI), and they never change the exit code of
`check`, `build`, or `test`.

Two `E0102` (undefined function) behaviors are worth stating here.
First, `E0102` does not special-case `print`: bare `print` and
`println` are builtins resolved before the undefined-name check
(§4.1), so they are never reported as undefined, and a user
function, struct, or local of the same name shadows them. Second, a
generic function body is checked once in an abstract pass where the
type parameters are opaque: scope, arity, and name errors there —
`E0102` and its fellows — report as if the body were concrete, so
an uninstantiated template still surfaces an undefined call.

