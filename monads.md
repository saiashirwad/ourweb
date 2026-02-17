# monads

## type class hierarchy

```
class Functor(f) {
  fn map(x: f(a), transform: a -> b) -> f(b)
}

class Monad(m) : Functor(m) {
  fn pure(value: a) -> m(a)
  fn bind(x: m(a), then: a -> m(b)) -> m(b)
}
```

no separate `Applicative` — derive `apply` from `bind` + `pure`. keeps the hierarchy flat. can add it later
if perf matters for some instances.

`Functor.map` is derivable from `Monad` but useful standalone — lots of things are functors that aren't
monads (mapped types, for instance).

## do-notation

`do { x <- expr; ... }` desugars to nested `bind` calls. last expression is the return value.

```
do {                          bind(expr1, fn(x) =>
  x <- expr1                    bind(expr2, fn(y) =>
  y <- expr2                      bind(expr3, fn(_) =>
  _ <- expr3                        pure(x + y))))
  pure(x + y)
}
```

rules:

- `x <- expr` desugars to `bind(expr, fn(x) => ...rest)`
- `_ <- expr` desugars to `bind(expr, fn(_) => ...rest)` (sequence, discard result)
- `expr` without `<-` as the last line is the return value
- `let x = expr` inside `do` is just a regular let binding (no bind)
- the monad is inferred from the return type or first `<-` expression

## instances

### Option

```
instance Monad(Option) {
  fn pure(v) = Some(v)
  fn bind(x, then) = match x {
    Some(v) => then(v)
    None    => None
  }
}
```

short-circuits on `None`. replaces null-checking chains.

### Result

```
instance Monad(Result(e)) {
  fn pure(v) = Ok(v)
  fn bind(x, then) = match x {
    Ok(v)  => then(v)
    Err(e) => Err(e)
  }
}
```

short-circuits on first `Err`. the error type `e` is fixed per chain.

### List

```
instance Monad(List) {
  fn pure(v) = [v]
  fn bind(xs, then) = xs |> flat_map(then)
}
```

nondeterministic computation. `do { x <- [1,2,3]; y <- [10,20]; pure(x + y) }` gives `[11,21,12,22,13,23]`.

### State

```
type State(s, a) = s -> (a, s)

instance Monad(State(s)) {
  fn pure(v) = fn(s) => (v, s)
  fn bind(m, then) = fn(s) => {
    let (a, s2) = m(s)
    then(a)(s2)
  }
}

fn get() -> State(s, s) = fn(s) => (s, s)
fn put(s: s) -> State(s, Unit) = fn(_) => ((), s)
fn modify(f: s -> s) -> State(s, Unit) = fn(s) => ((), f(s))
```

### Reader

```
type Reader(r, a) = r -> a

instance Monad(Reader(r)) {
  fn pure(v) = fn(_) => v
  fn bind(m, then) = fn(r) => then(m(r))(r)
}

fn ask() -> Reader(r, r) = fn(r) => r
fn asks(f: r -> a) -> Reader(r, a) = fn(r) => f(r)
```

## interaction with row types

monads compose naturally with the record system:

```
// Option on every field
type Partial(r) = Option * r
// { name: Option(String), age: Option(Int), ... }

// monadic traversal of record fields
fn sequence_record(r: Monad(m) * R) -> m(R)
  where R : Monad(m) *
// turns { name: Option("ada"), age: Option(36) } into Some({ name: "ada", age: 36 })
// any None → whole thing is None
```

## interaction with pipes

`|>` is for pure transformations. `do` is for monadic chains. they complement each other:

```
// pure
user |> .name |> to_upper

// monadic
do {
  user <- lookup_user(id)
  pure(user |> .name |> to_upper)
}
```

## implementation

monads monomorphize completely. `bind` for `Option` becomes a pattern match in C. `do` blocks become nested
function calls that inline to flat control flow. zero overhead after monomorphization — no vtables, no
dictionary passing, no heap-allocated closures for known types.

## what we don't want

- **IO monad**: no. effects are implicit like OCaml. see `effects.md` for typed effect tracking.
- **monad transformers**: no. they compose badly and the perf is terrible. the effect system handles this
  instead.
- **MonadPlus/Alternative**: maybe later. not needed for the core use cases.
