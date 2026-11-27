# JSGo Syntax

Formal grammar of the implemented language, in EBNF. `[ ]` marks
optionality, `{ }` repetition, `|` alternation, `` grouping.

```ebnf
file        = { import_decl } { decl } ;

import_decl    = "import" STRING ";"
               | "import" "{" [ import_spec { "," import_spec } [ "," ] ] "}"
                 "from" STRING ";"
               | "import" "*" "as" IDENT "from" STRING ";" ;
                (* "from"/"as" are contextual keywords: IDENT text *)
import_spec    = IDENT [ "as" IDENT ] ;

decl        = func_decl | struct_decl | class_decl | interface_decl ;
                (* top level only — a nested declaration is E0010 *)
func_decl   = [ "async" ] "function" [ receiver ] IDENT [ type_params ]
              "(" [ params ] ")" [ ":" type ] block ;
                (* async: top level only, never on a method (E0011) *)
receiver    = "(" IDENT ":" type ")" ;   (* (p: Point) or (p: *P) — struct
                                            methods only *)
params      = param { "," param } ;
param       = IDENT ":" type ;
type_params = "<" IDENT { "," IDENT } ">" ;  (* never empty: <> is E0006;
                                                plain functions, structs,
                                                and interfaces only —
                                                class<T> is E0011 *)

struct_decl = "struct" IDENT [ type_params ] "{" [ field_list ] "}" ;
field_list  = field { ";" field } [ ";" ] ; (* final ";" optional *)
field       = IDENT ":" type ;

class_decl  = [ "abstract" ] "class" IDENT
               [ "extends" IDENT ]
               [ "implements" IDENT { "," IDENT } ]
               "{" { class_member } "}" ;      (* classes are never generic *)
class_member = class_field | class_ctor | class_method ;
access      = "public" | "private" | "protected" ;   (* default: public *)
class_field = { access | "static" } IDENT ":" type ";" ;
class_ctor  = [ access ] "constructor" "(" [ params ] ")" block ;
class_method = { access | "static" | "abstract" } IDENT "(" [ params ] ")"
               (block | ";") ;   (* ";" = an abstract signature only;
                                      valid modifier combinations are
                                      checked: abstract needs an abstract
                                      class; static/abstract exclude each
                                      other and the constructor *)
interface_decl = "interface" IDENT [ type_params ] [ extends ] "{"
                 { iface_member } "}" ;   (* no bodies *)
extends        = "extends" IDENT { "," IDENT } ;
iface_member   = iface_prop | iface_method ;
iface_prop     = IDENT ":" type ";" ;
iface_method   = IDENT "(" [ params ] ")" [ ":" type ] ";" ;

type        = "void" | "bool"
            | "int8" | "int16" | "int32" | "int64"
            | "uint8" | "uint16" | "uint32" | "uint64"
            | "float32" | "float64" | "string"
            | IDENT                          (* struct/class/interface/
                                               Context type name *)
            | "[" "]" type                (* array: []int32, [][]T *)
            | "map" "[" type "]" type      (* map: map[K]V, nested allowed *)
            | "*" type                     (* pointer: *int32, **P, *[]T;
                                              never *void — E0006 *)
            | "(" type "," type { "," type } ")"   (* result list: (A, B) *)
            | "function" "(" [ params ] ")" [ ":" type ]
                                              (* function types *)
            | IDENT "<" type { "," type } ">" ;    (* Result<T, E>,
                                                      Task<T>, Channel<T>,
                                                      Box<T> and other
                                                      instances *)

block       = "{" { stmt } "}" ;

stmt        = let_decl
            | simple_stmt ";"
            | if_stmt | while_stmt | for_stmt
            | switch_stmt | select_stmt
            | return_stmt | break_stmt | continue_stmt
            | defer_stmt | go_stmt
            | block ;

let_decl    = ("let" | "const") (let_single | let_multi) ";" ;
let_single  = IDENT [ ":" type ] [ "=" expr ] ;
let_multi   = IDENT "," IDENT { "," IDENT } "=" expr_list ;
                                             (* types come from the initializer *)
simple_stmt = assign_stmt | incdec_stmt | expr_stmt ;
                                             (* no trailing ';' here: the
                                                for clauses own the boundary
                                                on both sides *)
assign_stmt = lvalue assign_op expr
            | ident_list "=" expr_list ;      (* parallel assignment *)
incdec_stmt = lvalue ("++" | "--") ;        (* sugar for += 1 / -= 1 *)
expr_stmt   = expr ;
lvalue      = "*" expr                        (* deref target: *p = v,
                                                 *p += v *)
            | IDENT { "[" expr "]" | "." member_name } ;
ident_list  = IDENT { "," IDENT } ;
assign_op   = "=" | "+=" | "-=" | "*=" | "/=" | "%=" ;
expr_list   = expr { "," expr } ;
defer_stmt  = "defer" expr ";" ;            (* expr must be a call *)
go_stmt     = "go" expr ";" ;               (* expr must be a call:
                                                go worker(arg) *)
if_stmt     = "if" expr block [ "else" (if_stmt | block) ] ;
while_stmt  = "while" expr block ;
for_stmt    = "for" [ "(" ] (";" | let_decl | simple_stmt ";")
              [ expr ] ";" [ simple_stmt ] [ ")" ] block ;
                (* the ")" pairs with the "("; the init clause is empty,
                   a let declaration, or an assignment / increment /
                   expression statement *)
switch_stmt = "switch" [ "(" ] expr [ ")" ] "{" { case_arm } "}" ;
case_arm    = ("case" expr { "," expr } | "default") ":" { stmt } ;
select_stmt = "select" "{" { select_arm } "}" ;
select_arm  = "case" [ "let" IDENT "=" ] expr ":" { stmt }
            | "default" ":" { stmt } ;
                (* arm expressions are channel operations only:
                   'case let v = ch.recv:', 'case ch.send(v):' *)
return_stmt = "return" [ expr_list ] ";" ;
break_stmt  = "break" ";" ;
continue_stmt = "continue" ";" ;

expr        = or_expr ;
or_expr     = and_expr { "||" and_expr } ;
and_expr    = eq_expr { "&&" eq_expr } ;
eq_expr     = rel_expr { ("==" | "!=") rel_expr } ;
rel_expr    = add_expr { ("<" | "<=" | ">" | ">=") add_expr } ;
add_expr    = mul_expr { ("+" | "-") mul_expr } ;
mul_expr    = unary { ("*" | "/" | "%") unary } ;
unary       = ("-" | "+" | "!" | "*" | "&" | "await") unary
                                              (* *p dereference, &x
                                                 address-of; &*p folds to p;
                                                 await binds tighter than
                                                 the binary operators *)
            | postfix ;
postfix     = primary { call | "." member_name | "[" expr "]" } ;
call        = [ "<" type { "," type } ">" ] "(" args ")" ;
                (* the <...> is type arguments only when ">" is
                   followed immediately by "(" — 'i < len(arr)'
                   stays a comparison *)
primary     = INT | FLOAT | STRING | "true" | "false" | "nil"
            | "this" | "super"                (* receiver and
                                                  parent chain *)
            | "new" IDENT "(" [ args ] ")"     (* class construction *)
            | arrow                            (* shares its prefixes with
                                                  IDENT and "(" expr ")";
                                                  the parser looks ahead
                                                  for "=>" *)
            | IDENT
            | "(" expr ")"
            | "[" [ expr { "," expr } ] "]"        (* array literal *)
            | "{" [ expr ":" expr { "," expr ":" expr } ] "}" ; (* map literal *)

arrow       = arrow_params [ ":" type ] "=>" (block | expr) ;
arrow_params = "(" [ arrow_param { "," arrow_param } ] ")" | IDENT ;
arrow_param = IDENT [ ":" type ] ;   (* annotation required except for a
                                        lone bare "(x)" or "x =>", typed
                                        from the expected function type *)

args        = [ expr { "," expr } ] ;

member_name = IDENT | KEYWORD ;  (* after a dot any keyword is a plain
                                    name: errors.new, obj.type *)
```

## Lexical notes

- Identifiers: `[A-Za-z_][A-Za-z0-9_]*`.
- Integer literals support `_` separators and the `0x`, `0b`, `0o`
  radix prefixes.
- Semicolons separate statements and are optional between them (and
  before `}` / at end of input) — only the `for` clauses require them
  (`for (init; cond; post)`).
- `++` and `--` are postfix operators that desugar to `+= 1` / `-= 1`.

## `switch`

The tag may be parenthesized (`switch (x) {`) or bare (`switch x {`).
Each arm is `case CONST {, CONST}* :` or `default :`. Case values must
be compile-time constant literals assignable to the tag type
(int/float/string/bool); duplicates are rejected. There is **no
fallthrough** — control jumps past the switch at the end of an arm's
body, and `break` is optional. The tag is evaluated exactly once. See `LANGUAGE_SPEC.md`.

## Arrays and maps

Types are written `[]T` (any nesting: `[][]int32`) and `map[K]V`
(composite key/value types allowed: `map[string][]int32`). `map` is a
**contextual** keyword — it starts a map type only when directly
followed by `[`, so `map` remains usable as an identifier.

```jsgo
let a: []int32 = [1, 2, 3];   // array literal
a.push(4);                    // append
a[0] = 9;   a[0] += 1;        // element assignment
let n: int32 = a.length;      // int32 element count

let m: map[string]int32 = {"a": 1};  // map literal
m["b"] = 2; m["b"] += 10;    // entry assignment (inserts if absent)
let has: bool = m.has("a");   // existence test
m.delete("a");                // remove an entry (no-op if absent)
let size: int32 = m.length;   // int32 entry count
```

A map literal in expression position is written `{k: v...}`; a `{`
that begins a **statement** still opens a block. Empty literals (`[]`,
`{}`) cannot be inferred and require a type annotation (`E0218`).
Array/map values are reference handles. Map
keys in literals must be scalar (int, float, string, bool); reading a
missing key is a runtime error — test with `.has` first.

## Pointers and `nil`

Pointer types are written `*T` for any element type (nesting allowed:
`**Point`, `*[]int32`); `*void` is rejected (`E0006`). The zero value
of a pointer-typed slot or field is `nil`, and `nil` is **only** a
pointer value — it needs an expected type (assignment, annotation,
argument, return, or a comparison against a typed pointer) and is a
targeted error anywhere else.

```jsgo
let x: int32 = 5;
let p: *int32 = &x;      // address-of a variable
*p = 10;                 // write through the pointer
*p += 7;                 // compound write (reads + writes the pointee)
println(x);          // 17 — same cell

let q: *int32 = nil;     // nil needs the *int32 annotation
if q == nil { ... }      // identity comparisons: == nil, != nil
q = p;
if q != nil { println(*q); }

let a: []int32 = [1, 2];
let e: *int32 = &a[1];   // array elements are addressable
let same: *int32 = &*p;  // &*p folds to p
```

- `&x` accepts variables (locals, parameters), variable-rooted field
  chains (`&o.inner.v`), and array elements. **Map elements are not
  addressable** (`E0212`) — growth would move them.
- Address-taken *variables* are lowered into heap boxes, so a pointer
  can safely escape its scope (see `MEMORY_MODEL.md`).
- Dereferencing `nil` is a checked runtime error
  (`jsgo: runtime error: nil pointer dereference`, exit 1) — never
  memory corruption.
- There is **no auto-dereference sugar**: `p.f` on a `*T` reports
  `E0216` with the help `(*p).f`; write `(*p).f` explicitly.
- Pointers are not printable (`E0213`), not indexable, and have no
  arithmetic — the only operators are dereference and `==`/`!=` with
  `nil` or another pointer.
- Struct methods may declare a **pointer receiver**
  (`function (p: *Point) shift(dx: int32) { ... }`) to mutate the
  pointee. Receiver kinds match exactly at call sites: call a pointer
  receiver on a `*T` (`pp.shift(10)`), a value receiver on a value
  (`p.area`), writing `(*pp).m` or `(&x).m` when you must cross
  kinds (`E0201`). Classes reject `*C` receivers (`E0104`) — class
  references are already pointers.

## Structs and methods

A struct declaration names a **nominal** value type; field types may
reference structs declared later in the file (or another file).

```jsgo
struct Point {
    x: int32;
    y: int32;
}

let p = Point(3, 4);        // positional constructor (call syntax)
println(p.x);           // field read
p.x = 10;                   // field write (base must be assigned: E0210)
p.y += 5;                   // compound field write
```

Methods attach functions to a struct through a receiver:

```jsgo
function (p: Point) area: int32 {
    return p.x * p.y;
}
println(p.area);      // method call
```

A receiver may also be a pointer — `function (p: *Point) shift` —
which mutates the pointee instead of a copy; see "Pointers and `nil`"
above for the exact receiver-kind matching rules.

Notes: construction is positional in declaration order (`E0204` on
arity mismatch, `E0201` on field type mismatch); unknown fields list
the available ones (`E0216`); a struct name may not collide with a
builtin type, function, or another struct (`E0103`); using the struct
name as a value is `E0209`; a struct that contains itself by value is
`E0219` (cycles through `[]T`/`map[K]V` handles are fine). The
receiver of a **value-receiver** method is a **copy** (Go value-
receiver semantics), and structs are values:
assignment and argument passing copy them (unlike arrays/maps, which
are handles). `==`/`!=` compare identical structs field by field —
deeply, with reference fields (`*T`, arrays, maps, classes,
interfaces, functions) compared by identity — and printing
writes a JS container, `{x: 1, name: "hi"}`, when every field is
printable, otherwise `E0213`.

## Classes

A class declares a **reference type** with fields, a constructor,
methods, and access modifiers. The specification's §5
example in full:

```jsgo
class User {
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

let u: User = new User(1, "ada");
println(u.getName);
```

- Declaration: `class Name [extends Super] { members }`. Members are
  fields (`[access] name: T;`), at most one `constructor(...) { ... }`,
  and methods (`[access] name(params)[: result] { ... }`). The access
  modifier is `public` (the default), `private`, or `protected`.
- Construction is `new Name(args)` — calling the class name directly
  is `E0011`, using it as a value is `E0209`. `this` names the
  receiver inside methods and the constructor.
- Inheritance is **single** (`extends`): a subclass inherits fields
  and methods and may override methods — an override must match the
  parent signature exactly (arity `E0204`, types `E0201`) and may not
  reduce visibility (`E0107`). Redefining an inherited field is
  `E0103`; an inheritance cycle is `E0219`. A subclass instance is
  assignable to any superclass-typed variable (§5 assignability).
- `super(...)` must be the first statement of a constructor when
  written; otherwise a zero-argument `super` is inserted before the
  body. A class without a constructor gets a synthesized one that
  chains up the hierarchy (a constructor-less subclass of a
  parameterized parent is `E0204`). `super.method(args)` calls the
  parent's implementation statically.
- Access control: `private` is visible only in the declaring class,
  `protected` in the declaring class and its subclasses (`E0107`
  otherwise). Private members remain **virtual** when overridden —
  private is a compile-time access check, not dynamic hiding.
- Class references are never null (see `MEMORY_MODEL.md`): a `let`
  without a value is `E0218`, every class-typed field the class
  declares must be assigned by a top-level `this.f = ...;` in its
  constructor (`E0210`), and `return` in a constructor is `E0011`.
- `==`/`!=` compare **identity** when one side's class is assignable to
  the other's (unrelated classes are `E0202`). Classes may not be
  printed (`E0213`), used as conditions (`E0203`), or used as map keys
  (`E0202`).
- `static` and `abstract` class members are documented in the next
  section; interfaces follow.

## Static members and abstract classes

Class members may be marked `static` (class-wide, no receiver) or,
inside an `abstract class`, `abstract` (signature only):

```jsgo
class Counter {
    static count: int32;                  // one storage slot for all
    constructor { }
    static next: int32 {                // no receiver, no `this`
        Counter.count = Counter.count + 1;
        return Counter.count;
    }
    reset: void { Counter.count = 0; }  // instance method, same slot
}

abstract class Shape {
    constructor { }
    abstract area: int32;               // signature only: `;`
    describe: string { return "shape"; }
}

class Circle extends Shape {
    r: int32;
    constructor(r: int32) { super; this.r = r; }
    public area: int32 { return this.r * 3; }
}
```

- Modifier order on a member is free: `[public|private|protected]
  [static] [abstract]` in any combination valid for that member.
- Static fields live **outside the instance**: read and written as
  `Counter.count` (also as the target of `+=`, and through nested
  paths such as `Geo.origin.x`), never through an instance
  (`c.count` is `E0216` with a hint). They are zero-initialized
  before `main` runs, may not be `void` (`E0201`), may not hold a
  class reference (`E0201` — nothing would initialize it), and may
  not shadow a visible field (`E0103`).
- Static methods are called as `Counter.next` — never through an
  instance (`c.next` is `E0216`), never as a non-static member
  through the class name (`Counter.reset` is `E0216` with a hint).
  They have no receiver, so `this` and `super` are `E0011` inside
  them; access control, arity (`E0204`), and argument types
  (`E0201`) apply as usual. A static method and an instance method
  of the same class may not share a name (`E0103`); subclasses may
  hide an inherited static (separate namespaces — the call always
  names the declaring class).
- Static members do **not** take part in virtual dispatch or
  interface satisfaction: a class satisfying an interface only via a
  same-named static method is `E0204`.
- `abstract class A { ... }` cannot be constructed (`new A` is
  `E0201`). Its abstract methods have no body and emit nothing; a
  **concrete** subclass must provide every inherited abstract method
  or the build fails with `E0204` at the class declaration. Override
  signature checks apply to abstract methods like any other (result
  mismatch `E0201`, visibility reduction `E0107`), and
  `super.abstractMethod` is `E0011` — there is no parent
  implementation to call. Misplaced `static`/`abstract` (top level,
  in a struct, on a constructor, on a field, combined with each
  other, or an abstract method in a concrete class) is a targeted
  `E0011`.

## Interfaces

An interface declares a **structural type**: named properties with
types plus signature-only methods, with no bodies and no instances
(properties and `extends` added by):

```jsgo
interface UserRepository {
    count: int32;
    findById(id: int32): User;
    save(user: User): void;
}

interface AuditedUserRepository extends UserRepository {
    lastAudit: string;
}

class MemoryRepo implements AuditedUserRepository {
    count: int32;
    lastAudit: string;
    constructor { this.count = 0; this.lastAudit = "never"; }
    public findById(id: int32): User { return new User(id, "ada"); }
    public save(user: User): void { this.count = this.count + 1; }
}

let r: AuditedUserRepository = new MemoryRepo;   // class -> interface wrap
println(r.count);                                 // property read
r.lastAudit = "today";                            // property write
println(r.findById(1).getName);                 // dynamic dispatch
```

- Declaration: `interface Name [extends A, B] { members }`. Each
  member is either a property `name: type;` or a method
  `name(params)[: result];` — neither may have a body and neither may
  carry an access modifier (either is `E0011`), and both need their
  terminating `;` (`E0006`). One name is a property or a method,
  never both (`E0103`). Interfaces share the struct/class name
  namespace (`E0103`).
- `extends A, B` inherits every property and method of the listed
  interfaces, recursively: a child's flattened member set is exactly
  the union of its parents' and its own, with the child's own
  declaration winning. An unknown or non-interface target is `E0104`,
  and an inheritance cycle — including an interface extending itself
  — is `E0219`, the code a class inheritance cycle already uses.
- A class adopts an interface with `implements I1, I2...` (multiple
  interfaces are allowed; multiple *class* inheritance is not).
  Satisfaction is **structural**: the clause is optional and acts as
  a checked assertion — a missing method or property is `E0204`, a
  signature or property-type mismatch `E0201`, an unknown or
  non-interface name `E0104`. Only **public** members satisfy an
  interface; interface members are implicitly public, so a
  private/protected method or field never does (`E0107` with a
  clause, `E0201` structurally).
- Substitution follows the same rule between interfaces: a value of
  type `AuditedUserRepository` may be used wherever `UserRepository`
  is expected, because `extends` has already flattened the parent's
  members into the child. The reverse is rejected — an interface
  without the extended members cannot stand in for the extended one.
- An interface-typed value pairs the object with its interface
  metadata; assigning a class value performs the wrap implicitly (no
  `as`/cast syntax), and widening one interface into another
  re-derives that metadata from the same object.
- Interface property reads and writes (`r.count`, `r.lastAudit = ...`)
  go through the byte offset of the property as measured from the
  class the value was wrapped at, so a subclass instance reads and
  writes the same slot the class itself would.
- Interface method calls `r.findById(x)` dispatch dynamically through
  the runtime instance's vtable, so a subclass override wins even
  when the static type is an ancestor class.
- An interface value is never null: a `let` without a value is
  `E0218`. Interfaces may not be printed (`E0213`), compared
  (`E0202`), used as conditions (`E0203`), or used as map keys
  (`E0202`); `new I` is `E0201` and using the interface name as a
  value is `E0209`.

## Generic declarations

Plain functions, structs, and interfaces may declare type parameters.
A parameterized declaration is a template; every use
materializes a concrete, monomorphic instance:

```jsgo
function ident<T>(v: T): T {
    return v;
}

struct Box<T> {
    value: T;
}

interface Container<T> {
    get: T;
    size: int32;
}

function main {
    let a: int32 = ident<int32>(5);  // explicit type arguments
    let b = ident("hi");             // inferred: T = string
    let x = Box<int32>(42);          // constructor with arguments
    let y = Box("text");             // constructor, inferred
    let list: []Box<int32> = [];     // instances in any type position
}
```

- Syntax: `<T, U>` sits between the name and the parameter list
  (functions, interfaces) or before the body (structs). The list is
  never empty (`<>` → `E0006`).
- Call sites: `f<int32, string>(args)` and `Box<int32>(field values)`.
  A `<...>` is parsed as type arguments only when the closing `>` is
  immediately followed by `(` — `i < len(arr)`, `a < b && c > d`, and
  every other comparison keep their meaning. Wrong count or a
  non-template target: `E0214`. A parameter inference cannot bind:
  `E0222` (naming the parameter), and a plain argument mismatch inside
  a generic call: `E0201`.
- Rejected with `E0011`: generic classes (`class Box<T>`), generic
  constructors and methods, type parameters on a receiver method, and
  `main<T>`. Rejected with `E0103`: duplicate or builtin-named type
  parameters, and one base name claimed by two declarations (plain
  `f` vs `f<T>`). Bare template names in type positions and templates
  as values are `E0214`.
- A receiver may name a materialized instance —
  `function (b: Box<int32>) twice: int32 { ... }` — exactly like a
  plain struct type; methods themselves cannot be generic. A receiver
  that is the bare template (`Box`, `*Box`) or reuses the template's
  own type parameter (`Box<T>`) is rejected with one targeted `E0011`
  (help: name a concrete instance); a misspelled argument like
  `Box<Bad>` stays an ordinary `E0104`.
- `class IntBox implements Container<int32>` implements one
  instantiation; `Container<int32>` as a constructor is `E0011`
  (interfaces have no implementation — construct with `new`).
- Bodies are checked twice by design: once abstractly (a never-
  instantiated template still reports name and arity errors, while
  operations through `T` wait) and once per instance with concrete
  types. Instantiation is per distinct argument list, so
  `ident<int32>` exists once however many call sites use it.
- No `where`-style constraints yet (future work).

## Arrow functions and function types

Function types name a parameter list and an optional result:

```jsgo
function(int32, string): bool     // two parameters, bool result
function: void                  // no parameters, no result (void)
function: int32                 // no parameters, int32 result
function(function(int32): int32): int32   // a function parameter
```

The parameter list may be empty (`function: void`); an omitted result
means `void`. Function types are allowed anywhere a type is allowed:
parameters, results, `let` annotations, struct fields, static fields,
array element types, and map value types (but not map keys — see
`TYPE_SYSTEM.md`). Assignment requires the two types to be spelled
identically (invariance).

Arrow functions are the anonymous-function syntax (spec §5):

```jsgo
const add = (a: int32, b: int32): int32 => {
    return a + b;
};
let sq = (x: int32): int32 => x * x;      // expression body
let hi = (name: string) => "hi " + name;  // result inferred
let inc: function(int32): int32 = x => x + 1; // bare param from context
```

- The parens form `(p: T...)` requires every parameter annotated,
  except a lone bare `(x)` (or bare `x =>`), which takes its type
  from the expected function type; without one it is `E0011`.
  Multiple unannotated parameters are `E0006`.
- The result annotation is optional: an expression body infers it; a
  block body without annotation and without an expected type is
  `void`, so a non-void block body needs `: T` (missing return is
  `E0205`).
- There is no anonymous `function` literal — arrows only.
- An arrow captures every visible local **by value** at creation time
  (arrays/maps/classes/interfaces keep their shared handles), and
  assigning to a captured variable inside the body is `E0011`.
  A plain function used as a value (`let f = add;`) never captures —
  it pairs with a generated thunk.

## Multi-return values

A function may declare a result list of two or more types; `return`
then takes one value per type:

```jsgo
function swap(a: int32, b: int32): (int32, int32) {
    return b, a;
}

let x, y = swap(1, 2);      // multi-value declaration
x, y = y, x;                // parallel assignment (a real swap)
let t: (int32, int32) = swap(3, 4);   // tuples may be annotated
```

`return f;` forwards a call whose result list matches. All
right-hand side values are evaluated before any target is written, so
`a, b = b, a;` swaps. Notes: counts must match (`E0204`), component
types must be assignable (`E0201`), a multi-value result in a
single-value position — an inferred single `let`, a single `return`
— is `E0220` with a message that names both counts; annotations are
not allowed on multi-value `let` and compound assignment is not
allowed on a value list (`E0011`, with a "write `let a, b = ...`"
help). Tuples are ordinary structs internally: they compare
field-wise like any struct but are not printable (`E0213`),
and user methods may not use a tuple receiver (`E0104`).

## `Result<T, E>`

Error handling is values, not exceptions (spec §7).
`Result<T, E>` carries an `ok` tag, the value payload, and the error
payload; `Ok(v)` and `Error(msg)` are **contextual constructors** —
they type only where a Result is expected:

```jsgo
function divide(a: float64, b: float64): Result<float64, Error> {
    if (b == 0.0) {
        return Error("Division by zero");
    }
    return Ok(a / b);
}

let r = divide(2.0, 0.0);
if !r.isOk {
    println(r.error);
}
println(divide(6.0, 3.0).value);
```

`Error` is a builtin alias of `string` (so `struct Error` is `E0103`);
the fields `r.ok`, `r.value`, `r.error` are directly readable and the
methods `r.isOk`, `r.value`, `r.error` are builtin (lowered as
field reads — no user method may be declared on a Result receiver,
`E0104`). `Ok`/`Error` without a Result expectation is `E0221`;
payload mismatch is `E0201`; `value` on a failed Result yields the
zero value of `T` — check `isOk` first.

## `defer`

`defer call(args);` registers a call to run at function exit, in LIFO
order, after the return value is evaluated:

```jsgo
function main {
    defer println("cleanup 1");
    defer println("cleanup 2");
    println("body");
}
// prints: body, cleanup 2, cleanup 1
```

The deferred expression must be a call (`E0011`), and `defer` is
allowed only as a **direct statement of the function body** — not
inside `if`, loops, `switch`, or nested blocks (`E0011`; the
conditional subset needs the runtime defer stack — still future). Arguments
and method receivers are evaluated **at the defer statement** and
captured (Go semantics: later mutations of a captured variable are not
visible to the deferred call).

## Multi-file projects

With no file argument, `jsgo build`/`jsgo check` compile every `.jsgo`
file under the project root (the directory containing `jsgo.mod`, else
the working directory), excluding hidden directories and
`packages/`, and build output. Functions may call each other across
files; imports are file-scoped. Exactly one `main` may be declared.

### File imports (ES6 style)

A path starting with `./` or `../` and ending in `.jsgo` names a file
in the program:

```jsgo
import "./util.jsgo";                       // pull in, no bindings
import { helper, Point as P } from "./util.jsgo";
import * as util from "./util.jsgo";

function main {
    println(helper);       // bare println: no import needed
    let p: P = P(1, 2);     // aliases work in type positions
    println(util.add(2, 3)); // file-scoped namespace call
}
```

- The target file is **pulled into the build automatically** —
  transitively and cycle-safe — so `jsgo run main.jsgo` compiles
  everything main names, even files outside the project walk
  (`../shared/x.jsgo`). A missing file is `E0108`.
- Named imports bind the target's top-level functions, structs,
  classes, and interfaces (`E0105` otherwise). `as` aliases resolve in
  calls, values, `new`, and type annotations; locals shadow them.
  Aliases are program-wide like every top-level name; colliding with
  an existing name, a builtin type, `print`/`println`/`Ok`/`Error`, or
  a module imported by the same file is `E0103`.
- Namespaces (`* as`) are **file-scoped** like stdlib modules: only
  files that imported them may call `util.member(...)`, which is
  equivalent to calling the file's declaration unqualified. Unknown
  member `E0105`; using the namespace or its member as a value
  `E0209`.
- Cross-file calls keep working without any import — file imports add
  pull-in and validation, they never hide a name that used to resolve.

### Printing

`println(value)` and `print(value)` are builtins: no import, exactly
one printable argument (`E0204`/`E0213`), `print` writes without a
trailing newline. A user function, struct, or local of the same name
shadows them.
Printable values are booleans, numbers, strings, and structs whose
fields are all printable — a struct prints as a JS container,
`{x: 1, name: "hi"}`.

## Module calls

An imported module is a member-access base — never a value:

```jsgo
import "strings";
import "os";
function main {
    println(strings.contains("abc", "b"));
    defer os.exit(0);          // module calls may be deferred
}
```

- Signatures are fixed: arity mismatches are `E0204`, argument types
  `E0201`, unknown members `E0105`, using the module as a value
  `E0209`, calling a module that is not imported in the file `E0106`.
- Fallible functions return `Result<T, string>` values (see
  `Result<T, E>` above): `let n = strconv.parseInt(s);`.
- A class or struct may share a module's name. The **imported module
  wins for calls**; without the import the type's static/constructor
  path runs instead.

After a dot, **any keyword token is an ordinary member name**
(JavaScript's `obj.new` grammar) — this is what makes
`errors.new(...)` legal even though `new` is a keyword (and `obj.type`,
`cfg.default` parse the same way):

```jsgo
let e = errors.new("boom");    // needs import "errors"
```

A non-name token after a dot is still `E0006`.

## Concurrency

One cooperative scheduler runs every task on a single thread; see
`CONCURRENCY_MODEL.md` for the semantics. An `async` function's
declared result is **logical**: calling it starts a task immediately
and the call yields a `Task<T>`.

```jsgo
async function square(n: int32): int32 {
    return n * n;
}
async function run: int32 {
    let t: Task<int32> = square(7);  // task already running
    let v: int32 = await t;          // suspend until it finishes
    return v;
}
function main {
    go run;                        // spawn a named call
}
```

- `async function f(...): T` — the result may be omitted (`void`), and
  a `void` body yields `Task<void>`. `await` is valid only inside an
  async function and only on a `Task<T>` (`E0011`); an async `main`
  is `E0500` and an async class method is `E0011`.
- `go f(args)` starts a task running a **named** call — plain
  functions, methods, statics, `super`, generic instances, and module
  functions. A function value, a print builtin (`print`, `println`),
  a constructor, or a handle/collection operation is
  `E0011`: there is no declaration to spawn.

Channels are values of `Channel<T>` built by the builtin
`Channel<T>([capacity])` (omitted capacity = unbuffered; `T` may not
be `void`; the `chan` keyword stays reserved with an `E0020` help
pointing at `Channel<T>`):

```jsgo
let ch: Channel<int32> = Channel<int32>(8);   // buffered
let pipe: Channel<string> = Channel<string>; // unbuffered
ch.send(42);
let v: int32 = ch.recv;   // blocks; yields the zero value once the
                            // channel is closed and drained
ch.close;                 // send on closed: runtime error
let open: bool = ch.isClosed;
let n: int32 = ch.length;
```

`select` waits for one arm to be ready (random among the ready ones),
or runs `default` immediately when nothing is ready:

```jsgo
select {
case let v = ch.recv:     // receive into a bound variable
    println(v);
case ch.send(v):            // send
    println("sent");
default:                    // optional; without it the task parks
    println("nothing");
}
```

- Arms are channel operations only (`E0011` otherwise), a send arm
  cannot bind a value (`E0011`), and there is at most one `default`
  (`E0103`). There is no `<-` operator anywhere.

A `Context` token scopes cancellation; passed as a task's first
argument it becomes that task's context, and a cancelled context
resumes `await` with the zero value:

```jsgo
let ctx: Context = newContext;      // or newContext(parent)
let t: Task<int32> = worker(ctx);
ctx.cancel;
let stopped: bool = ctx.isCancelled;
```

The program exits when every task has finished; if every live task is
parked, the runtime reports `jsgo: deadlock: all tasks are parked`.

## Command-line arguments

`jsgo run file.jsgo a b` compiles `file.jsgo` and runs it with `a b`
as its program arguments — `os.args` returns them after the program
name. Put `--` before the program's own flags
(`jsgo run file.jsgo -- -v`); everything after `--` is passed
through. `jsgo build` takes at most one file argument and rejects
extras with exit code 2.

## Reserved keywords

Implemented: `import function let const if else for while return
break continue switch case default true false nil struct defer class
extends super constructor this new private public protected interface
implements static abstract`.

Reserved and rejected: `func`, the Go-style spelling replaced by
`function` everywhere — declarations, methods, and function types.
It still lexes as a keyword so the parser can report
a targeted error with a `function` help. `package` is no longer
part of the language: a leftover `package ...;` line reports one
targeted `E0010` with a delete-this-line help, then parsing continues
after it.

Reserved for planned features (using them raises `E0020` with a
planned-release note): `chan` (the help points at `Channel<T>`),
`range`, and `enum type`.

Reserved non-keyword syntax: backtick template strings (`E0020`). The
`=>` arrow operator is implemented (see "Arrow functions and function
types" above).

## Deliberately not in the grammar (v0.2)

Top-level variables, nested functions, object literals, destructuring,
spread/rest, `for-of`/`for-in` loops, and `try/catch`. All
are planned; none parse today, and each raises a diagnostic rather than
being silently ignored. (Array and map literals *do* parse — see
above. Generics were removed from this list when they shipped — see
"Generic declarations" above.)
