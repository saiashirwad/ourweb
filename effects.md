# effects

typed effect tracking without an IO monad. inspired by Effect-TS but built on row types instead of
intersection types.

## core idea

`Effect(value, error, requirements)` — three type parameters:

- **value** (a): what you get on success
- **error** (e): extensible variant of possible failures
- **requirements** (r): record of services needed (row type)

```
type Effect(a, e, r)
```

the key insight: **record `+` is Effect-TS's `&` for requirements, variant `+` is Effect-TS's `|` for
errors.** the whole effect system falls out of row polymorphism we already have.

## Effect-TS mapping

| Effect-TS                         | ourweb                    | mechanism               |
| --------------------------------- | ------------------------- | ----------------------- |
| `Effect<A, E, R>`                 | `Effect(a, e, r)`         | same shape              |
| `R1 & R2` (requirements)          | `r1 + r2` (record merge)  | row polymorphism        |
| `E1 \| E2` (errors)               | `e1 + e2` (variant merge) | extensible variants     |
| `Layer<ROut, E, RIn>`             | `Layer(out, e, inp)`      | same shape              |
| `Effect.gen(function*() { ... })` | `do { ... }`              | do-notation             |
| `yield*`                          | `<-`                      | bind                    |
| `Effect.provide`                  | `Effect.provide`          | eliminates requirements |
| `pipe(eff, Effect.map(...))`      | `eff \|> Effect.map(...)` | pipe operator           |

## defining effects

### errors as extensible variants

```
type DbError   = [| .not_found: { id: Int } | .connection_failed: String]
type AuthError  = [| .unauthorized | .expired: { token: String }]
type SmtpError  = [| .invalid_address: String | .send_failed: String]
```

### services as record types

```
type Database = {
  query: (String) -> Effect(List(Row), DbError, {}),
  find:  (Int) -> Effect(Row, DbError, {}),
}

type AuthService = {
  verify: (String) -> Effect(UserId, AuthError, {}),
}

type SmtpClient = {
  send: (String, String) -> Effect(Unit, SmtpError, {}),
}
```

### effects declare what they need

```
fn get_user(id: Int) -> Effect(User, DbError, { db: Database }) = do {
  row <- Effect.service(.db) |> .find(id)
  pure(user_from_row(row))
}

fn verify_token(tok: String) -> Effect(UserId, AuthError, { auth: AuthService }) = do {
  uid <- Effect.service(.auth) |> .verify(tok)
  pure(uid)
}

fn send_welcome(email: String) -> Effect(Unit, SmtpError, { smtp: SmtpClient }) = do {
  _ <- Effect.service(.smtp) |> .send(email, "welcome!")
  pure(())
}
```

## composition

composing effects merges requirements (record `+`) and errors (variant `+`):

```
fn register(token: String, data: SignupData)
  -> Effect(User, DbError | AuthError | SmtpError,
            { db: Database, auth: AuthService, smtp: SmtpClient })
= do {
  uid   <- verify_token(token)       // needs { auth }, fails with AuthError
  user  <- get_user(uid)             // needs { db }, fails with DbError
  _     <- send_welcome(user.email)  // needs { smtp }, fails with SmtpError
  pure(user)
}
```

the compiler infers the merged requirement and error types. you can annotate them or let inference do the
work.

## layers — dependency injection

`Layer(provides, error, requires)` — given requirements, build services.

```
type Layer(out, e, inp)
```

### defining layers

```
type Config = { db_url: String, smtp_host: String, auth_secret: String }

let db_layer: Layer({ db: Database }, DbError, { config: Config }) =
  Layer.from(fn(env) => do {
    conn <- connect(env.config.db_url)
    pure({ db: make_database(conn) })
  })

let auth_layer: Layer({ auth: AuthService }, Never, { config: Config }) =
  Layer.from(fn(env) =>
    pure({ auth: make_auth(env.config.auth_secret) }))

let smtp_layer: Layer({ smtp: SmtpClient }, SmtpError, { config: Config }) =
  Layer.from(fn(env) => do {
    client <- connect_smtp(env.config.smtp_host)
    pure({ smtp: client })
  })
```

### composing layers

`+` merges layers — outputs merge, requirements unify, errors union:

```
let app_layer = db_layer + auth_layer + smtp_layer
// Layer({ db: Database, auth: AuthService, smtp: SmtpClient },
//       DbError | SmtpError,
//       { config: Config })
```

### providing layers

`Effect.provide` eliminates requirements:

```
let runnable = register(token, data) |> Effect.provide(app_layer)
// Effect(User, DbError | AuthError | SmtpError, { config: Config })

let program = runnable |> Effect.provide(Layer.value({ config: prod_config }))
// Effect(User, DbError | AuthError | SmtpError, {})
//                                                ^^ fully satisfied

let result = Effect.run(program)
// Result(User, DbError | AuthError | SmtpError)
```

## error handling

### catch specific variants

```
fn safe_register(token: String, data: SignupData)
  -> Effect(Result(User, AuthError), DbError | SmtpError,
            { db: Database, auth: AuthService, smtp: SmtpClient })
= register(token, data) |> Effect.catch(.unauthorized, .expired)
```

catching removes those variants from the error channel and wraps the value in `Result`.

### match on errors

```
register(token, data) |> Effect.catch_all(fn(err) => match err {
  .not_found(e)         => Effect.succeed(create_user(data))
  .unauthorized         => Effect.fail(.unauthorized)
  .connection_failed(s) => Effect.retry(3)
  other                 => Effect.fail(other)
})
```

### retry

```
get_user(id)
  |> Effect.retry({ times: 3, delay_ms: 100, on: .connection_failed })
```

### fallback

```
fn get_user_cached(id: Int)
  -> Effect(User, DbError, { db: Database, cache: Cache })
= get_from_cache(id) |> Effect.or_else(fn(_) => get_user(id))
```

## resource safety

scoped resources that clean up on exit:

```
fn with_connection(url: String) -> Effect(Connection, DbError, {})
  = Effect.acquire_release(
      acquire: fn() => connect(url),
      release: fn(conn) => close(conn),
    )

fn transact(id: Int) -> Effect(User, DbError, { config: Config }) = do {
  conn <- with_connection(config.db_url)  // released when scope exits
  row  <- conn |> .find(id)
  pure(user_from_row(row))
}
```

## why not algebraic effects?

algebraic effects (like OCaml 5 / Koka / Eff) are powerful but:

- hard to compile to C efficiently (need delimited continuations or CPS)
- monomorphization doesn't play well with resumable handlers
- the row-type-based approach gives us most of the benefits with simpler compilation

the Effect type is really just `r -> Result(a, e)` after monomorphization. layers become plain function
composition. zero runtime overhead.

## algebra

effects follow the same algebraic identities as records:

```
// requirement merging
Effect(a, e, r1 + r2)  ↔  compose effects needing r1 and r2

// error merging
Effect(a, e1 | e2, r)  ↔  compose effects failing with e1 and e2

// provide eliminates requirements
provide(Effect(a, e, r + provided), Layer(provided, e2, r2))
  = Effect(a, e | e2, r + r2)

// layer merging
layer1 + layer2  ↔  outputs merge, errors union, requirements unify

// fully satisfied effect can be run
Effect(a, e, {})  →  Result(a, e)
```
