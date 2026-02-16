# Urweb Core OCaml Port Blueprint

## Objective
Estimate the complexity of porting the Ur/Web implementation (excluding standard library code) from SML to OCaml, and define a phased plan that maximizes trivial/mechanical rewrites first while isolating hard, non-trivial areas.

## Context And Invariants
- Scope excludes the language standard library under `urweb/lib/ur` and `urweb/lib/js`.
- Preserve compiler pipeline behavior and output compatibility (IR transforms, SQL/C generation, and runtime interaction).
- Keep the existing backend contracts with generated C/runtime APIs unless explicitly redesigned.
- Maintain feature parity for supported protocols and DB backends (`http`, `cgi`, `fastcgi`, `static`; `sqlite`, `mysql`, `postgres`).

## Baseline Complexity (Measured)
- Core implementation code (excluding stdlib):
  - `urweb/src` (`.sml`, `.sig`, `.grm`, `.lex`, plus `src/c/*.c/*.h`): `72,936` LOC across `208` files.
  - Build-graph files in `src/sources` only: `64,397` LOC across `193` files (excluding generated `config.sml` and `xml/entities.sml`).
- Standard library (excluded from core estimate):
  - `urweb/lib/ur` (`.ur/.urs`): `4,742` LOC across `19` files.
  - `urweb/lib/js/urweb.js`: `3,434` LOC.

## Portability Breakdown

### Trivial/Mechanical Port Candidates (Mostly Direct SML -> OCaml)
- Approximate size: `47,302` LOC (`~64.9%` of `src` top-level SML/sig/lex/grm set).
- Characteristics:
  - Mostly pure AST/type transformations and compiler passes.
  - Heavy use of algebraic data types, pattern matching, recursive transforms.
  - Module/signature structure maps naturally to OCaml modules/module types.
- Representative high-volume files:
  - `urweb/src/elaborate.sml` (5228 LOC)
  - `urweb/src/monoize.sml` (4683 LOC)
  - `urweb/src/iflow.sml` (2184 LOC)
  - `urweb/src/elab_env.sml` (1721 LOC)
  - `urweb/src/corify.sml` (1332 LOC)

### Non-Trivial Port Areas (Cannot Be Purely Mechanical)
- Approximate size: `25,634` LOC (`~35.1%`).
- Sub-buckets:
  - Backend/runtime-coupled SML: `12,077` LOC
  - Parser/driver/toolchain coupling: `5,558` LOC
  - C runtime in `urweb/src/c`: `7,999` LOC
- Why non-trivial:
  - MLton/SML runtime dependencies (`MLton`, `Posix`, `Socket`, daemon/background threading files).
  - ML-Yacc/ML-Lex toolchain (`urweb.grm`, `urweb.lex`, generated parser interfaces).
  - C code generation surface (`cjr*`, `sql*`, DB-specific code emitters).
  - Runtime C integration and ABI assumptions (`urweb/src/c/*.c`, OpenSSL, ICU, pthreads, FastCGI).

## Architecture

### Data Structures / Types
- Keep existing IR ladder and type families as first-class OCaml modules:
  - `Source -> Elab -> Expl -> Core -> Mono -> Cjr`
- Keep pass interfaces as typed transforms to preserve incremental validation.
- Preserve explicit environment/state modules (`*_env`, `*_util`) during first port pass.

### Public Interface Changes
- CLI and compiler orchestration (`main.mlton.sml`, `compiler.sml`) should expose equivalent flags/workflow first, then optionally modernize.
- Runtime/API boundaries to C should remain stable initially to avoid simultaneous semantic and ABI churn.

### File Structure
- Phase 1: one-to-one OCaml mirror of current `src/` module boundaries.
- Phase 2: optional consolidation/refactor after behavior parity.
- Suggested compatibility layer modules:
  - `Basis_compat` (SML Basis API shims)
  - `Map_compat` (for `BinaryMapFn`, `IntBinaryMap` call sites)
  - `Parser_compat` (token/location plumbing replacement)

## Implementation Steps (Atomic, Verifiable)
1. Freeze inventory and classify modules into `mechanical`, `adapter-needed`, `runtime-coupled`.
2. Build OCaml compatibility shim for key SML library idioms (maps, sets, IO/time/string helpers).
3. Port AST/type declaration modules (`source`, `elab`, `expl`, `core`, `mono`, `cjr`) and printers.
4. Port pure transformation passes in dependency order (largest first for risk burn-down).
5. Replace parser stack (`.grm/.lex`) with OCaml parser tooling while keeping token/AST contract stable.
6. Port orchestration/CLI and incremental build logic (`compiler.sml`, `main.mlton.sml`) with OCaml/Unix equivalents.
7. Validate DB/C emission modules (`mysql/sqlite/postgres/sql/cjr*`) against golden outputs.
8. Preserve or wrap existing C runtime (`src/c`) before considering runtime rewrite.
9. Run parity test matrix on demos/tests and generated SQL/C diffs.

## Verification Strategy
- Structural parity:
  - Match module count and pass ordering to current compiler pipeline.
- Behavioral parity:
  - Compare generated C/SQL artifacts on a fixed test corpus.
  - Run existing demo/tests as regression harness.
- Runtime parity:
  - Validate HTTP/CGI/FastCGI startup and request handling against baseline.
  - Validate DB integration on sqlite/mysql/postgres smoke suites.
- Risk checkpoints:
  - Parser replacement accepted only when AST diffs are within expected normalization deltas.
  - Backend accepted only when generated C compiles and passes existing integration tests.

## Complexity Conclusion
- Practical answer to "how much is trivially rewritable to OCaml":
  - Around two-thirds (`~65%`) looks mechanically portable with a compatibility layer.
  - Around one-third (`~35%`) is not trivial and needs targeted redesign/adaptation (toolchain, runtime coupling, parser generation, backend/C/DB boundaries).
