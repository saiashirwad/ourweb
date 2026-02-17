# refinement types

phase 2/3 feature. not needed for the initial port but the type system is a natural foundation for it.

## approach: liquid types

Liquid Haskell proved you can add refinement types to an ML-family type checker without rewriting it. the refinement layer sits on top — generate verification conditions, ship them to an SMT solver (Z3).

the core type checker (elaborate.sml) stays mostly unchanged.

## basic refinements

predicates on values in a decidable fragment (linear arithmetic, booleans, equalities):

```
type Pos = Int where > 0
type Age = Int where >= 0, < 150
type NonEmpty = String where len > 0
type Port = Int where >= 1, <= 65535

fn divide(a: Int, b: Int where != 0) -> Int {
  a / b
}

divide(10, 3)    // ok
divide(10, 0)    // compile error

let x = parse_int(input)
divide(10, x)    // compile error: can't prove x != 0

if x != 0 {
  divide(10, x)  // ok: branch condition refines x
}
```

## refinements + rows

refined record fields — validate once at the boundary, carry the proof everywhere:

```
type ValidUser = {
  name: String where len > 0,
  age: Int where >= 0, < 150,
  email: String where contains("@"),
}

fn register(user: ValidUser) -> UserId { ... }

fn from_input(raw: User) -> Result(ValidUser, String) {
  if raw.name.len == 0 then return Err("name required")
  if raw.age < 0 || raw.age >= 150 then return Err("bad age")
  if !contains(raw.email, "@") then return Err("bad email")
  Ok(raw)  // compiler knows raw satisfies ValidUser here
}
```

## refinements mapped across records

combine with `*` to refine every field at once:

```
type NonDefault(t) = match t {
  String => String where len > 0
  Int    => Int where != 0
  Bool   => Bool where == true
  _      => t
}

type StrictConfig = NonDefault * ServerConfig

// comprehension: every numeric field must be positive
type AllPositive(r) = { k: v where > 0 for k: v in r if v is Num }
```

## measures

pure functions the solver can reason about (Liquid Haskell's trick):

```
measure len(s: String) -> Int
measure sorted(l: List(t)) -> Bool

type SortedList(t) = List(t) where sorted

fn binary_search(haystack: SortedList(Int), needle: Int) -> Option(Int) {
  // compiler knows haystack is sorted
}

fn sort(l: List(t)) -> SortedList(t) where t : Ord {
  // compiler checks output satisfies `sorted`
}
```

## implementation cost

| approach | complexity | dependency |
|----------|-----------|------------|
| liquid types (SMT) | moderate — separate pass | Z3 |
| simple range types (Ada-style) | low — bounds checking | none |
| full dependent types (Idris-style) | massive — rewrite everything | none but pain |

liquid types are the practical path. compositional inference, check per-function, SMT handles the reasoning.

## phasing

1. **phase 1**: base type system — row polymorphism, type classes, type-level computation, C output. this is the port.
2. **phase 2**: simple refinements — `Int where > 0`, `String where len > 0`. decidable fragment, Z3 dependency.
3. **phase 3**: measures, refined record fields, mapped refinements. the novel stuff.

## what doesn't exist yet

no language has: row polymorphism + type-level map + refinement types + native compilation.

- Liquid Haskell: refinements, no rows
- PureScript: rows, no refinements
- F*: refinements + dependent types, no rows, targets OCaml/F#/Wasm
- ourweb with refinements: all of the above, compiles to C
