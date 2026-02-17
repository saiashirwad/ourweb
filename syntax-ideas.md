# syntax ideas

exploring different syntax philosophies for urweb's type system.
all examples based on the type-level computation tutorial.

## Style A: "Open by default"

Records are structurally typed. If you access `r.x`, the compiler knows `r` has field `x`. No row variables, no explicit rest types. `..` marks open record types when you want to be explicit.

```
// field projection — just write it, types are inferred
fn project(r, field) = r[field]

// explicit signature if you want one
fn project(r: { field: t, .. }, field: Name) -> t = r[field]

// type-level functions
type Pair(t) = (t, t)
type Compose(f, g, t) = f(g(t))

// map: the * operator applies a type function across a record
type R = { a: Int, b: Float, c: String }

let x: Option * R = { a: Some(1), b: None, c: Some("hi") }
let x: Pair * R   = { a: (1, 2), b: (3.0, 4.0), c: ("5", "6") }

// concat: + merges records, disjointness is implicit
fn concat(a, b) = a + b
// inferred: (r1, r2) -> r1 + r2  [compiler errors if fields overlap]

// church nats
type Nat = for t. (t, t -> t) -> t

let zero: Nat = fn(z, _) => z
let one: Nat  = fn(z, s) => s(z)

fn succ(n: Nat) -> Nat = fn(z, s) => s(n(z, s))
fn plus(a: Nat, b: Nat) -> Nat = fn(z, s) => a(b(z, s), s)
```

Key ideas: `*` for type-level map, `+` for record merge (disjointness checked structurally), `..` for open records, `for t.` instead of `forall`. Type application on polymorphic values is inferred — `n(z, s)` works because the compiler knows `z: t` so `n` must be instantiated at `t`.

---

## Style B: "Comprehensions"

Type-level map uses comprehension syntax, like TypeScript's mapped types or Python's list comprehensions.

```
// field projection
fn project(r: { .field: t, ..rest }) -> t {
  r.field
}
project(.b, my_record)

// type functions
type Pair(t) = (t, t)

// map via comprehension — read it like english
type R = { a: Int, b: Float, c: String }

let x: { k: Option(v) for k: v in R } = { a: Some(1), b: None, c: Some("hi") }
let x: { k: Pair(v)   for k: v in R } = { a: (1, 2), b: (3.0, 4.0), c: ("5", "6") }
let x: { k: String    for k in R }     = { a: "1", b: "2", c: "3" }

// concat
fn concat(a: r1, b: r2) -> { k: v for k: v in r1 | r2 } {
  { ...a, ...b }
}

// church nats — same as before
type Nat = for t. (t, t -> t) -> t
```

The comprehension `{ k: Option(v) for k: v in R }` reads as "a record where each field `k` with value type `v` from `R` becomes `Option(v)`." It's longer than `Map(Option, R)` but self-documenting — you don't need to know what `Map` does.

`{ k: String for k in R }` is the constant case — every field becomes `String` regardless of its original type.

---

## Style C: "Inference-maximal"

Almost nothing is annotated. The compiler infers kinds, types, constraints, everything. Types read like documentation, not requirements.

```
// just write the code. types follow.
project(r, field) = r[field]

// if you WANT to document the type:
// project : { field: t, .. } -> Name -> t

// type aliases
Pair(t) = (t, t)
Compose(f, g, t) = f(g(t))

// records
R = { a: Int, b: Float, c: String }

x : Map Option R = { a: Some(1), b: None, c: Some("hi") }
x : Map Pair R   = { a: (1, 2), b: (3.0, 4.0), c: ("5", "6") }
x : Map (_ => String) R = { a: "1", b: "2", c: "3" }

// concat — disjointness inferred from ++
concat(a, b) = a ++ b

// church nats — no annotations at all
Nat = for t. t -> (t -> t) -> t

zero(z, s) = z
succ(n)(z, s) = s(n(z, s))
plus(a, b)(z, s) = a(b(z, s), s)

to_int(n) = n(0, _ + 1)
```

No `fn`, no `let`, no `type`, no braces. Definitions are `name = value` or `name(args) = body`. Type annotations are `name : Type`. Everything else inferred.

The risk: when inference fails, error messages are brutal because there's nothing to anchor on.

---

## Style D: "Dot-oriented"

Field names are just `.name`. Records are the central abstraction. Everything is a method or projection.

```
// .field is a first-class accessor function
let name = .name(person)
// or with pipe
let name = person |> .name

// .name has type { name: t, .. } -> t  automatically
// no need to define project at all

// type functions via method-like syntax
type R = { a: Int, b: Float, c: String }

let x: R.map(Option) = { a: Some(1), b: None, c: Some("hi") }
let x: R.map(Pair)   = { a: (1, 2), b: (3.0, 4.0), c: ("5", "6") }
let x: R.map(_ => String) = { a: "1", b: "2", c: "3" }

// concat via spread
fn concat(a: r1, b: r2) -> r1.merge(r2) {
  { ...a, ...b }
}

// record operations chain
type UserInput = Schema
  .pick(.name, .email, .age)
  .map(Option)

// church nats — dot doesn't apply here, back to normal
type Nat = for t. (t, t -> t) -> t
```

`.name` as a first-class function eliminates the need for `project` entirely. `R.map(Option)` reads like calling a method on the record type. You could chain: `R.pick(.a, .b).map(Option)`.

---

## Comparison on the hard example

The `concat` function is the real test — it needs disjointness, type-level map, and record concatenation all at once.

```
// Ur/Web original
fun concat [f :: Type -> Type] [r1 :: {Type}] [r2 :: {Type}] [r1 ~ r2]
    (r1 : $(map f r1)) (r2 : $(map f r2)) : $(map f (r1 ++ r2)) = r1 ++ r2

// Style A: "Open by default"
fn concat(a: f * r1, b: f * r2) -> f * (r1 + r2) = a + b

// Style B: "Comprehensions"
fn concat(a: { k: f(v) for k:v in r1 }, b: { k: f(v) for k:v in r2 })
  -> { k: f(v) for k:v in r1 | r2 }
  = { ...a, ...b }

// Style C: "Inference-maximal"
concat(a, b) = a ++ b

// Style D: "Dot-oriented"
fn concat(a: r1.map(f), b: r2.map(f)) -> r1.merge(r2).map(f) = { ...a, ...b }
```

---

## Recommended mix

Pick the best from each:

- **`*` for map** from Style A — `Option * R` is hard to beat for brevity
- **`+` for concat** from Style A — implies disjointness, no `where` clause
- **`..` for open records** from Style A — `{ name: t, .. }` is clean and well-known
- **`.field` as first-class accessor** from Style D — eliminates `project` and `#name`
- **`for t.` for quantification** — shorter than `forall`, reads naturally
- **Inference-heavy** from Style C — kind annotations only when the compiler asks

```
type Pair(t) = (t, t)
type R = { a: Int, b: Float, c: String }

let x: Pair * R = { a: (1, 2), b: (3.0, 4.0), c: ("5", "6") }

fn concat(a: f * r1, b: f * r2) -> f * (r1 + r2) = a + b

let person = { name: "ada", age: 36 }
person |> .name    // => "ada"

type Nat = for t. (t, t -> t) -> t
fn succ(n: Nat) -> Nat = fn(z, s) => s(n(z, s))
```

---

## Ideas stolen from TypeScript

TS has a surprisingly powerful type-level language built on mapped types, conditional types, and indexed access. Several ideas port well to a language with real row polymorphism.

### Indexed access — get a field's type

```typescript
// TS
type Name = Person["name"]  // string
```

```
// ours
type Name = Person[.name]

// or just dot syntax at the type level
type Name = Person.name
```

Urweb can do this (it's just type-level record projection) but has no clean syntax for it. This is useful for extracting types from records without having to name them separately.

### keyof — get field names as a type

```typescript
// TS
type Fields = keyof Person  // "name" | "age" | "email"
```

```
// ours — keyof returns a set of field names
type Fields = keyof Person  // .name | .age | .email

// useful for constraining a function to only accept valid field names
fn get(r: R, field: keyof R) -> R[field]
```

Urweb's Name kind already supports this conceptually but there's no `keyof` extraction. Adding it makes generic field access trivial.

### Pick / Omit — record subsetting

```typescript
// TS
type Summary = Pick<Person, "name" | "age">
type Rest = Omit<Person, "name">
```

Urweb has `ECut` (remove field) and `ECutMulti` (remove multiple) at the value level, but the syntax is gnarly. TS-inspired alternatives:

```
// operator style
type Summary = Person[.name, .age]           // pick fields
type Rest = Person - .name                    // omit one field
type Rest = Person - (.name, .email)          // omit multiple

// or method style (Style D)
type Summary = Person.pick(.name, .age)
type Rest = Person.omit(.name)

// at the value level too
let summary = person[.name, .age]             // => { name: "ada", age: 36 }
let rest = person - .name                     // => { age: 36, email: "..." }
```

The `-` operator for omit is clean — it's the inverse of `+` for merge. Record algebra: `+` merges (must be disjoint), `-` removes fields.

```
// these identities hold:
(a + b) - keyof b  ==  a
person[.name] + person[.age]  ==  person[.name, .age]
```

### Partial / Required — bulk optionality

```typescript
// TS
type Draft = Partial<Person>     // all fields optional
type Full = Required<Draft>      // undo Partial
```

```
// ours — just map with Option
type Draft = Option * Person
// { name: Option(String), age: Option(Int), email: Option(String) }

// "Required" is the inverse — strip Option from each field
type Full = unwrap * Draft
// where unwrap(Option(t)) = t

// or with a builtin
type Full = Required * Draft
```

This falls out naturally from `*` (map). No special syntax needed.

### Conditional types — type-level if/match

This is the big one TS has that urweb doesn't.

```typescript
// TS
type IsString<T> = T extends string ? true : false
type NonNullable<T> = T extends null | undefined ? never : T
type ReturnType<F> = F extends (...args: any) => infer R ? R : never
```

In our syntax, two options:

**Option 1: ternary**
```
type IsString(t) = t is String ? True : False
type NonNull(t) = t is Null ? never : t
```

**Option 2: match (more powerful, scales better)**
```
type Unwrap(t) = match t {
  Option(inner) => inner
  List(inner) => inner
  _ => t
}

type ReturnOf(f) = match f {
  (..args) -> r => r
}

type ElementOf(t) = match t {
  List(e) => e
  _ => never
}
```

`match` at the type level is pattern matching on type constructors. The `infer` keyword from TS becomes implicit — variables in patterns are automatically bound.

### Filtering mapped types — the killer combo

TS can filter fields during mapping. This combines mapped types + conditional types.

```typescript
// TS — keep only string fields
type StringFields<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K]
}
```

In our syntax, using comprehension style:

```
// keep only string fields
type StringFields(r) = { k: v for k: v in r if v is String }

// keep only numeric fields, make them optional
type OptionalNums(r) = { k: Option(v) for k: v in r if v is Num }

// rename fields (TS key remapping)
type Getters(r) = { "get_{k}": () -> v for k: v in r }
```

The `if` clause in comprehensions is natural and readable. Combined with `*` for simple cases:

```
// simple map — use *
let x: Option * R = { ... }

// map with filter or transform — use comprehension
let x: { k: v for k: v in R if v is String } = { ... }
```

`*` for the 80% case, comprehensions for the 20%.

### Satisfies — check without widening

```typescript
// TS
const config = { port: 3000, host: "localhost" } satisfies Config
// type is { port: 3000, host: "localhost" }, not Config
// but it's checked against Config
```

```
// ours
let config = { port: 3000, host: "localhost" } : Config
// if : widens, use a different operator for checking without widening:
let config = { port: 3000, host: "localhost" } as! Config  // check but keep literal type
```

Honestly this might not be needed if the type system is structural and doesn't widen by default. In urweb's system, records are already structurally typed — `{ port: 3000, host: "localhost" }` has type `{ port: Int, host: String }` and is assignable to any open record type that's a subset.

### Record — homogeneous records from a key set

```typescript
// TS
type Flags = Record<"debug" | "verbose" | "strict", boolean>
// { debug: boolean, verbose: boolean, strict: boolean }
```

```
// ours — use * with a name set
type Flags = Bool * (.debug, .verbose, .strict)

// or a function
type Flags = fill(Bool, (.debug, .verbose, .strict))

// or comprehension
type Flags = { k: Bool for k in (.debug, .verbose, .strict) }
```

---

## Revised recommendation (with TS ideas)

```
// type-level map (simple)
Option * R

// type-level map (complex, with filtering)
{ k: Option(v) for k: v in R if v is String }

// record merge
a + b

// record subset
R[.name, .age]

// record omit
R - .name

// indexed access
R[.name]        // type of the name field
R.name          // same thing

// keyof
keyof R         // .name | .age | ...

// type pattern matching
type Unwrap(t) = match t {
  Option(inner) => inner
  _ => t
}

// field accessor
person |> .name

// open records
fn get(r: { name: t, .. }) -> t

// quantification
type Nat = for t. (t, t -> t) -> t
```

The TS additions that matter most:
- **`R - .field`** for omit (inverse of `+`)
- **`R[.field]`** for indexed access
- **`match t { ... }`** for type-level pattern matching
- **`if` in comprehensions** for filtered mapping
- **`keyof`** for field name extraction
