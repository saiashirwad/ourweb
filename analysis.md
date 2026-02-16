# Ur/Web → OCaml Rewrite Analysis

## Codebase Overview

~65K lines across 243 files in `src/`. Excludes stdlib, demos, runtime C.

### Compilation Pipeline

6 IRs, 30+ passes:

```
Source → Elab → Expl → Core → Mono → CJR → C output
                                        ↘ JS output
```

### Largest Files

| File | Lines | Role |
|------|-------|------|
| elaborate.sml | 5,228 | type checker + unification + module system |
| monoize.sml | 4,683 | type erasure / monomorphization |
| cjr_print.sml | 3,959 | C code generation |
| iflow.sml | 2,184 | information flow analysis |
| compiler.sml | 1,788 | pipeline orchestrator |
| sqlcache.sml | 1,732 | SQL query caching |
| elab_env.sml | 1,721 | type checking environment |
| mysql.sml | 1,618 | MySQL backend |
| jscomp.sml | 1,369 | JS code generation |
| corify.sml | 1,332 | module system elimination |

## Portability Tiers

### Trivial (~60%)

Pure algebraic datatypes + pattern matching. Direct translation.

- IR definitions: source.sml, elab.sml, expl.sml, core.sml, mono.sml, cjr.sml
- Pretty-printers: all `*_print.sml`
- AST utilities: all `*_util.sml`
- Simple passes: shake (DCE), tag, explify, fuse, mono_shake, mono_inline
- Checkers: pathcheck, sidecheck, dbmodecheck, scriptcheck, sigcheck, marshalcheck
- Error reporting: elab_err.sml
- Utilities: list_util, order, utf8, sha1, fileio, prim

### Moderate (~25%)

Needs thought but no fundamental obstacles.

- elab_env.sml (1,721L) — environment with de Bruijn lifting, uses IntBinaryMap
- corify.sml (1,332L) — module elimination, ref-based state
- disjoint.sml (285L) — row type disjointness constraint solver
- reduce.sml / mono_reduce.sml — beta reduction with mutable state
- especialize.sml / specialize.sml — monomorphization helpers
- Database backends (postgres/mysql/sqlite) — SQL dialect string generation
- 7 SML functors (union_find_fn, multimap_fn, key fns)
- Parser/lexer (urweb.grm 2,404L, urweb.lex 578L) — Menhir/sedlex rewrite

### Hard (~15%)

The critical 15%. Complex algorithms + heavy mutable state.

- elaborate.sml (5,228L) — dependent types, row type unification, constraint postponement, blame tracking, module system interleaved with type checking. 123 ref usages.
- monoize.sml (4,683L) — type-directed compilation, mutable pvars state
- cjr_print.sml (3,959L) — C codegen, not algorithmically hard but massive
- iflow.sml (2,184L) — information flow analysis, specialized constraints
- sqlcache.sml (1,732L) — SQL caching optimization

## Type System Complexity

The type system is the hardest part:

- **Dependent types** via `TDisjoint` constraints (record field disjointness)
- **Row types** with concat, map, projection — specialized unification
- **Higher-order kind polymorphism** (`CKAbs`, `CKApp`, `TKFun`)
- **Unification variables with predicates** — `KUnknown of (kind -> bool)` validates solutions
- **Constraint postponement** — delayed unification via `mayDelay` ref + `delayedUnifs` list

## SML → OCaml Translation Guide

### Module System

| SML | OCaml | Notes |
|-----|-------|-------|
| `structure Foo :> FOO` | `module Foo : FOO` | OCaml `:` = SML `:>` (always opaque) |
| `where type t = int` | `with type t = int` | identical semantics |
| `sharing type A.t = B.t` | `with type t = A.t` | or destructive substitution `:=` |
| `BinaryMapFn(K)` | `Map.Make(K)` | or `patricia-tree` for int keys |
| `BinarySetFn(K)` | `Set.Make(K)` | same |
| `IntBinaryMap` | `Map.Make(Int)` | or `PatriciaTree.IntMap` (faster) |
| SML generative functors | OCaml applicative by default | add `()` arg for generative |

All 7 urweb functors translate directly. Example:

```sml
(* SML *)
functor UnionFindFn(K : ORD_KEY) :> sig
    type unionFind
    val union : unionFind * K.ord_key * K.ord_key -> unionFind
end = struct
    structure M = BinaryMapFn(K)
    structure S = BinarySetFn(K)
    ...
end
```

```ocaml
(* OCaml *)
module UnionFindFn (K : Map.OrderedType) : sig
  type t
  val union : t -> K.t -> K.t -> t
end = struct
  module M = Map.Make(K)
  module S = Set.Make(K)
  ...
end
```

### Parser/Lexer

**Lexer**: mllex → **sedlex**. Ur/Web is a web language, needs UTF-8. sedlex works on unicode codepoints via ppx.

**Parser**: mlyacc → **Menhir with `--table --inspection`**.

Key wins:
- Incremental parsing API for LSP (save/resume parser state)
- `$startpos` / `$endpos` replace mlyacc's `%pos` + `CONleft`/`cexpright`
- Named bindings (`name=SYMBOL`) instead of positional `$1`
- No automated converter exists — manual port required

Translation patterns:

```
# mlyacc                          # menhir
%term STRING of string      →     %token <string> STRING
%nonterm exp of exp         →     %type <exp> exp
%pos int                    →     (built-in $startpos/$endpos)
decl @ decls                →     d=decl; ds=decls { d @ ds }
```

Compile: `menhir --table --inspection --explain parser.mly`

### Mutable State (679 ref occurrences)

Keep using `ref`. OCaml's own type checker (`typing/ctype.ml`) uses mutable refs for unification. Not a problem to solve.

Better pattern from OCaml's compiler — indirection nodes for path compression:

```ocaml
type type_desc =
  | Tvar of string option
  | Tlink of type_expr    (* indirection for unification *)
  | ...
and type_expr = { mutable desc : type_desc; mutable level : int; id : int }
```

### Exception-Based Control Flow

SML and OCaml exceptions are nearly identical. One gotcha: SML local exceptions have dynamic identity (each `let exception E in ...` creates fresh constructor). OCaml 4.08+ has local exceptions but without fresh identity. Workaround via first-class modules if needed.

For urweb's `KUnify'` / `CUnify'`: keep as exceptions, direct translation.

### MLton-Specific Replacements

| MLton | OCaml | Effort |
|-------|-------|--------|
| `MLton.Thread.new/switch` | Eio fibers or raw effects | rewrite 65 lines |
| `MLton.Signal.setHandler` | `Sys.set_signal` | direct |
| `MLton.Itimer.set` | `Unix.setitimer` | direct |
| Whole-program optimization | Flambda2 (`-O3 -flambda2`) | less aggressive but improving |

### Basis Library Mapping

| SML | OCaml |
|-----|-------|
| `TextIO.openIn` | `open_in` |
| `OS.FileSys.isDir` | `Sys.is_directory` |
| `OS.Path.concat` | `Filename.concat` |
| `Time.now()` | `Unix.gettimeofday()` |
| `Word8/Word32` | `int` / `Int32` / `Bytes` (no unsigned) |
| `Vector` | `Array` (no immutable vector in OCaml) |
| `List.foldl` | `List.fold_left` (args reordered!) |

### Pretty-Printing

Ur/Web's `Print.PD` combinators → **PPrint** (Pottier). Closest 1:1 mapping. `Fmt` more ergonomic for new code.

## OCaml Effects Assessment

Effects are unstable API (OCaml 5.3+ has syntax, breaking changes expected).

### Use effects for

- **Cooperative concurrency** — replace `bg_thread.mlton.sml` (65 lines) with Eio fibers. Eio 1.0 shipped March 2024, production-ready.
- **Domains** for parallel compilation passes (stable API)

### Don't use effects for

- Type checker unification state — refs work fine, effects add overhead
- Error handling — keep exceptions, zero cost until thrown
- Simple mutable state — ref is simpler and faster

### Maybe later (once API stabilizes)

- Backtracking / constraint postponement — natural fit for `mayDelay`/`delayedUnifs`, but multi-shot continuations need separate library (`ocaml-multicont`)
- State scoping — effects can scope mutable state to a pass

## Recommended Libraries

| Need | Library |
|------|---------|
| Parser | Menhir (`--table --inspection`) |
| Lexer | sedlex (UTF-8 native) |
| Union-Find | `unionFind` opam package, or port existing |
| Row types reference | tomprimozic/type-systems/extensible_rows |
| Constraint solving | Inferno (Pottier) — delayed unification, generalization |
| Int maps (hot path) | `patricia-tree` — O(min(n,m)) set ops |
| Pretty-printing | PPrint for port, Fmt for new code |
| Concurrency | Eio (effects-based, 1.0 stable) |
| Structured IO | Eio |

## Porting Strategy

### Phase 1 — boring OCaml, get it compiling

- `ref` for state, exceptions for errors
- `Map.Make` / `Set.Make` for collections
- sedlex + Menhir for frontend
- PPrint for pretty-printing
- Eio for the 65-line threading module

### Phase 2 — optimize after it works

- `patricia-tree` for hot int-keyed maps in elab_env
- Flambda2 for whole-program-ish optimization
- Profile before doing anything else

### Phase 3 — effects adoption (when API stabilizes)

- Scoped state for compiler passes
- Backtracking for constraint postponement
- Only if benchmarks show no regression

## IR Details

### Source (194 lines)

Raw AST. Unresolved qualified names, full module system, wildcards.

- Kinds: `KType | KArrow | KName | KRecord | KUnit | KTuple | KWild | KFun | KVar`
- Has `TDisjoint` constraints on record types

### Elaborated / Elab (205 lines)

Type-checked with metadata. Explicit de Bruijn indices.

- `KRel(int)`, `CRel(int)`, `CNamed(int)`, `CModProj(...)`
- Unification variables: `KUnif(span, string, kunif ref)`, `CUnif(...)`
- Still polymorphic, module system intact

### Explicit / Expl (167 lines)

All implicit arguments made explicit. Bridge between elaboration and core.

- `CKApp(con, kind)`, `EKApp(exp, kind)` — explicit kind application
- No implicit quantification anywhere
- Module system still intact

### Core (147 lines)

System F-like. Modules eliminated, no dependent types.

- `EFfi(string, string)` for external functions
- `PConFfi{mod, datatyp, params, con, arg}` for FFI pattern constructors
- Database constructs: `DTable, DSequence, DView, DIndex, DDatabase, DCookie, DStyle, DTask, DPolicy`

### Mono (174 lines)

Monomorphic. No polymorphism, types simplified.

- `TFun | TRecord of (string * typ) list | TDatatype(int) | TFfi | TOption | TList | TSource | TSignal`
- `EQuery{exps, tables, state, query, body, initial}` — SQL formalized
- `EJavaScript(mode, exp)` — client-side code
- `EServerCall`, `ESignalBind`, `ESpawn` — client/server split

### CJR (141 lines)

C-ready. Records are struct indices, prepared statements.

- `TRecord(int)` — struct #N
- `EApp(exp, exp list)` — multi-arg application
- `EQuery{..., prepared: {id, query, nested}?}` — prepared SQL

## Optimization Passes

### Core Level (9 passes, applied multiple times)

1. core_untangle (237L) — untangle mutual recursion
2. shake (234L) — DCE (applied 5x)
3. especialize (717L) — datatype constructor patterns (3x)
4. rpcify (168L) — RPC boundaries
5. tag (356L) — constructor tagging
6. reduce (954L) — beta reduction + inlining
7. unpoly — remove unused polymorphism
8. specialize (340L) — polymorphic specialization (3x)
9. effectize (208L) — effect inference

### Mono Level (8 passes, applied multiple times)

1. mono_opt (713L) — constant folding (4x)
2. mono_reduce (923L) — reduction (3x)
3. mono_shake (166L) — DCE (3x)
4. mono_inline (28L) — inline small functions
5. fuse (152L) — query/DML fusion (2x)
6. reduce_local (398L) — local constant propagation
7. untangle (568L) — recursion untangling (3x)
8. jscomp (1,369L) — JS code generation

### Analysis Passes

1. iflow (2,184L) — information flow / attack surface
2. scriptcheck — JS placement validation
3. dbmodecheck — database access mode
4. pathcheck — URL path validity
5. sidecheck — server/client separation
6. sigcheck — signature matching
7. marshalcheck — FFI marshalling
8. sqlcache (1,732L) — SQL result caching
9. effectize — effect inference
10. endpoints (117L) — HTTP endpoint collection
11. css (322L) — CSS summarization

## External Dependencies

From configure.ac:

- OpenSSL (required)
- POSIX threads (required)
- ICU (unicode, required)
- PostgreSQL libpq (optional)
- MySQL libmysql (optional)
- SQLite3 (optional)

OCaml FFI via ctypes or hand-written stubs. All C deps port directly.
