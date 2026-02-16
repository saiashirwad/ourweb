rewriting urweb (standard ml) in ocaml. keeping the type system, dropping the web framework.

## keep

- row polymorphism with disjointness constraints (the core innovation)
- type classes (eq, ord, show, read, num)
- functors + full module system
- kind polymorphism / higher-order kinds
- monomorphization pipeline
- whole-program compilation to C

## drop

- XML/HTML literals ("jsx stuff")
- database/SQL (postgres, mysql, sqlite, typed queries, all of it)
- javascript codegen
- RPC / client-server split
- FRP / signals / sources
- cookies, CSS system, task scheduling, URL routing
- information flow analysis, security policies
- effectization, script/dbmode checking, path validation
- sql caching, prepared statements, query fusion

## pipeline

```
Source → Elab → Expl → Core → Mono → C
```

no branching, no web-specific passes. ~25K lines to port from the original ~65K.
elaborate.sml (~5,200 lines) is the hard part — type checker, unification, row types, kind inference, type classes, module system.

## result

"ML with row types and type classes that compiles to C"
