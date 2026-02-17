# architecture

## compilation pipeline

```
Source → Elab → Expl → Core → Mono → CJR → C output → gcc → binary
```

no branching, no optional passes, no web-specific checking.

## what got cut

### entire files deleted (~11,500 lines)

- **DB backends**: postgres.sml, mysql.sml, sqlite.sml (3,645)
- **JS/frontend**: jscomp.sml, name_js.sml (1,542)
- **SQL infra**: sql.sml, iflow.sml, sqlcache.sml, fuse.sml, prepare.sml, checknest.sml (5,121)
- **RPC/protocol**: rpcify.sml, cgi.sml, fastcgi.sml (273)
- **Caching**: toy_cache.sml, lru_cache.sml, filecache.sml (647)
- **Web checks**: scriptcheck.sml, dbmodecheck.sml (268)
- **Web features**: css.sml, effectize.sml, tag.sml, pathcheck.sml, sidecheck.sml, endpoints.sml (~1,100)

### heavily simplified

- **monoize.sml** (4,683 → ~2,000): EQuery/EDml/ESignal*/EJavaScript/EServerCall all gone
- **cjr_print.sml** (3,959 → ~2,500): no SQL, no prepared stmts, no JS embedding
- **cjrize.sml** (746 → ~350): query/task/JS handling removed
- **compiler.sml** (1,788 → ~1,400): fewer passes to orchestrate

### mono pipeline collapse

```
before (27 passes):
  monoize → endpoints → opt → untangle → reduce → shake → opt
  → iflow → namejs → scriptcheck → dbmodecheck → jscomp → opt
  → fuse → untangle → reduce → shake → opt → reduce → fuse
  → untangle → shake → pathcheck → sidecheck → sigcheck
  → filecache → sqlcache → cjrize → prepare → checknest

after (~10 passes):
  monoize → opt → untangle → reduce → shake
  → opt → sigcheck → cjrize
```

## what remains (~25,000 lines)

the type system and core compilation:

- **elaborate.sml** (5,228) — the hard part. type checker, unification, row types, kind inference, type classes, module system
- **elab_env.sml** (1,721) — type checking environment with de Bruijn lifting
- **corify.sml** (1,332) — module system elimination
- **monoize.sml** (~2,000 after cuts) — type erasure, monomorphization
- **cjr_print.sml** (~2,500 after cuts) — C code generation
- **reduce.sml / mono_reduce.sml** (~1,900) — beta reduction, inlining
- **especialize.sml / specialize.sml** (~1,050) — monomorphization helpers
- **shake / mono_shake** (~400) — dead code elimination
- **parser/lexer** (urweb.grm 2,404 + urweb.lex 578) — menhir/sedlex rewrite
- **IR definitions + utilities** (~3,000) — source, elab, expl, core, mono, cjr + *_util, *_print
- **everything else** — disjoint.sml, core_untangle, settings, error reporting, etc.

## stripped IR shapes

every IR loses the same set of web constructs. the core expression language is untouched.

### removed from every IR's decl type
- DTable, DSequence, DView, DIndex, DDatabase
- DCookie, DStyle, DTask, DPolicy, DOnError

### removed from Mono.exp (12 constructors gone)
- EQuery, EDml, ENextval, ESetval
- EJavaScript, ESignalReturn, ESignalBind, ESignalSource
- EServerCall, ERecv, ESleep, ESpawn

### removed from Mono.typ (2 constructors gone)
- TSource, TSignal

### what stays in every IR
- kinds: KType, KArrow, KName, KRecord, KUnit, KTuple, KFun, KVar
- constructors: TFun, TCFun, TRecord, TDisjoint, CApp, CAbs, CRecord, CConcat, CMap, CName, CUnit, CTuple, CProj, CKAbs, TKFun
- expressions: EPrim, ERel, ENamed, EApp, EAbs, ECApp, ECAbs, EKAbs, ERecord, EField, EConcat, ECut, ECutMulti, ECase, ELet, EClosure, EWrite

## compilation target

urweb generates a single .c file, compiles with gcc, links against liburweb.a (the C runtime).

the runtime provides:
- request-scoped memory regions (no GC — alloc per request, free at end)
- output buffering via uw_write()
- URL routing dispatch
- thread pool

with web features cut, the runtime shrinks to basically:
- region allocator
- string output buffering
- main() scaffolding

alternative targets (future): wasm (edge compute, cloudflare workers), or just emit a library and let users bring their own main.

## SML → OCaml translation

- `ref` for mutable state (679 occurrences — keep as-is)
- exceptions for error handling (keep as-is)
- `Map.Make` / `Set.Make` for collections
- menhir + sedlex for parser/lexer
- PPrint for pretty-printing
- all 7 SML functors translate directly to OCaml
