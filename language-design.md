# language design

## identity

ML with row types, type classes, and type-level computation that compiles to C.

the core innovation from urweb is type-level record operations — map, fold, concat, pick, omit — with disjointness constraints that make them safe. no other production language has this combination with native compilation.

## what makes it unique

| language | has | lacks |
|----------|-----|-------|
| TypeScript | mapped types, conditional types | unsound, no real row types, JS runtime |
| PureScript | row polymorphism | no type-level map/match, JS target |
| OCaml | modules, functors | row types only on objects (barely used), no type-level computation |
| Haskell + GHC | type families, constraints | row types are a library hack, slow compiles, runtime weight |
| Rust | zero-cost, no GC | no row types, no type-level record ops |
| **ourweb** | rows + mapped types + modules + native | — |

## type system features

### row polymorphism
records and variants are open by default. `{ name: String, .. }` accepts any record with at least a `name: String` field.

### record algebra
- `+` merge (must be disjoint)
- `-` omit fields
- `|` override (right side wins)
- `R[.a, .b]` pick fields
- `R[.field]` indexed access (get a field's type)
- `keyof R` field name extraction

### type-level map
`*` applies a type function across every field of a record.
`Option * User` = `{ name: Option(String), age: Option(Int), ... }`

### type-level comprehensions (inspired by TS mapped types)
```
{ k: Option(v) for k: v in R }           // map
{ k: v for k: v in R if v is String }    // filter
```

### type-level pattern matching (inspired by TS conditional types)
```
type Unwrap(t) = match t {
  Option(inner) => inner
  _ => t
}
```

### record constraints
`where R : Show *` means every field in R satisfies Show.

### first-class field accessors
`.name` is a function `{ name: t, .. } -> t`. no need for a `project` function.

### extensible variants
same algebra as records: `+` extends, `..` for open, `-` for narrowing.

### functors
modules parameterized by record types. `module Crud(S: { .. })` generates typed CRUD ops for any schema.

### kind polymorphism
inferred. `type Fst(t: (_, _)) = t.0` works at any pair of kinds.

### first-class polymorphism
`for t.` quantification. values can be polymorphic, not just bindings.

## syntax principles

drawn from ReasonML and Gleam:
- `let` for bindings, `fn` for functions
- `type` for type-level definitions (not `con`)
- `match` for pattern matching
- `|>` for piping
- `{ }` for records and blocks
- `//` for comments
- lowercase field names
- uppercase type names
- no semicolons
- minimal keyword set

key syntax decisions:
- `*` for type-level map (operator, not function call)
- `+` for record merge (implies disjointness)
- `-` for record omit
- `|` for record override
- `..` for open record/variant types
- `.field` as first-class accessor
- `for t.` for universal quantification
- `where` for constraints
- `match t { ... }` for type-level pattern matching
- no `$` operator (type-level records ARE record types)
- kind annotations inferred by default

## what we explicitly don't want

- XML/HTML literals
- embedded SQL
- javascript codegen
- RPC / client-server split
- FRP / signals / sources
- cookies, CSS, task scheduling, URL routing
- information flow analysis

these are all web framework features baked into the compiler. we want the type system, not the framework.

## comparisons to existing syntax

### urweb (original)
```
fun concat [f :: Type -> Type] [r1 :: {Type}] [r2 :: {Type}] [r1 ~ r2]
    (r1 : $(map f r1)) (r2 : $(map f r2)) : $(map f (r1 ++ r2)) = r1 ++ r2
```

### ourweb
```
fn concat(a: f * r1, b: f * r2) -> f * (r1 + r2) = a + b
```

same types, same guarantees, 1/4 the syntax.
