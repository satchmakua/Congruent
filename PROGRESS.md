# Progress

Running log of where the build is and what's next. Keep this honest — it's the working memory between build sessions.

**Current phase:** M0–M6 complete ✅; M7 underway (✅ `break`/`continue`) — **next in M7:** CVC5 cross-check backend, bounded strings. See [ROADMAP.md](ROADMAP.md).

## State of the tree

| Component | File | Status |
| --- | --- | --- |
| Verdict / Counterexample data model | `src/congruent/equiv.py` | ✅ done |
| Orchestration / escalation | `src/congruent/equiv.py` | ✅ difftest → symbolic, sound UNKNOWN fallback |
| AST → typed IR + subset validation | `src/congruent/ir.py` | ✅ done (loud `UnsupportedConstruct`) |
| Fixed-width concrete interpreter | `src/congruent/difftest.py` | ✅ done (two's-complement, catches overflow) |
| Differential tester | `src/congruent/difftest.py` | ✅ done (boundary + random generation) |
| Symbolic interpreter → Z3 | `src/congruent/symbolic.py` | ✅ path-merge, bitvectors, floor //, **loop unrolling** |
| Z3 query + model decode | `src/congruent/solver.py` | ✅ UNSAT/SAT/unknown → Verdict; in-bound assumptions |
| Bounded loops (`for range`) | `ir.py` / `difftest.py` / `symbolic.py` | ✅ parse + capped concrete eval + symbolic unroll |
| Input preconditions (`assume`) | `ir.py` / `difftest.py` / `symbolic.py` | ✅ filters difftest + constrains solver; CLI `--assume` |
| `list[int]` arrays | `ir.py` / `difftest.py` / `symbolic.py` | ✅ bounded arrays, `len`, `for x in xs`, `xs[i]` (proven) |
| Runtime-error modeling | `src/congruent/symbolic.py` | ✅ OOB access + divide-by-zero as guarded errors (path-condition aware) |
| CLI | `src/congruent/cli.py` | ✅ parse → check → report, `--assume`, exit codes 0/1/2 |
| Verdict formatting | `src/congruent/report.py` | ✅ done (all four statuses) |
| Fixtures (eval set) | `tests/fixtures/` | ✅ 14 pairs (5 CX, 9 EQ; ints, loops, precondition, arrays, indexing, early-exit, list outputs) |
| Benchmarks | `benchmarks/` | ✅ recall (zero-unsound gate) + timing-vs-bound |
| Early exit (`return`/`break`/`continue`) | `src/congruent/symbolic.py` | ✅ state-threading pass; `broken`/`continued` per loop |
| Counterexample minimization | `src/congruent/solver.py` | ✅ incremental solver shrink (length, then scalars→0); `--no-minimize` |
| List outputs (build/return `list[int]`) | `ir.py` / `difftest.py` / `symbolic.py` / `solver.py` | ✅ literals + concat; map/filter verify; output length bounded |
| Demo gallery | `examples/` + `docs/demo.svg` | ✅ 8 realistic refactor pairs + runner, pinned by tests |
| Tests | `tests/` | ✅ 124 pass |

## What M0 delivers

- Parses the v1 Python subset to a typed IR; rejects everything outside it loudly.
- A **fixed-width two's-complement** concrete interpreter — so the differential
  stage catches overflow bugs (the midpoint `(lo + hi) // 2` case) that Python's
  unbounded ints would hide.
- `check()` returns `COUNTEREXAMPLE` (with the concrete input), `UNKNOWN` (no
  counterexample found — *not* a proof), or `ERROR` (mismatched signatures).
  `EQUIVALENT` is deliberately impossible until the symbolic stage exists.

### Scope note that surfaced during M0

Scalar `int` inputs range over the **whole** fixed-width domain, not the `--bound`.
So a function that is only equivalent for small `n` (e.g. a loop summed vs. a
closed form, which overflows at large `n`) is genuinely *non-equivalent* here and
should report a counterexample. Such pairs belong in M2 (bounded loops + bounded
input domains), not the M0 eval set — the original `sum_to_n` fixture was replaced
for this reason.

## What M1 delivers

- `symbolic.summarize` lowers a loop-free function to a single Z3 bitvector
  expression. Branches and early `return`s are handled by continuation-passing
  path merging (`if`-body that returns, then fall-through, becomes one `ite`).
- Semantics mirror the concrete interpreter: two's-complement wrap, signed
  comparisons, and **Python floor `//`/`%`** (Z3's `/` truncates, so floor is
  reconstructed explicitly — otherwise M0 and M1 would disagree on negatives).
- `solver.prove_equivalence` asserts the two outputs differ over shared inputs;
  `UNSAT` → EQUIVALENT (complete over the width, since no loops), `SAT` → model
  decoded to a `Counterexample`, `unknown` → UNKNOWN.
- **Soundness fallback:** anything not modeled (non-constant/zero divisor, list
  params) raises `UnsupportedForProof` and `check()` returns UNKNOWN — never a
  false EQUIVALENT.

## What M2 (loops) delivers

- `for <var> in range(...)` in the IR/parser (`range(stop)` and `range(start,
  stop)`); `return`/`break`/`continue` inside loops are rejected for now.
- **Symbolic unrolling** (`symbolic._unroll_for`): unroll `bound` times, each
  iteration guarded by `start + k < stop`; loop bodies run as environment
  transformers (no early return), with an `ite` merge per guarded step. Loop
  variables don't escape; variables written in the loop must be initialized
  before it (else `UnsupportedForProof`).
- **Bounded model checking**: an in-bound assumption (`Not(start + bound <
  stop)`) is added to the query, so a loop verdict means "EQUIVALENT *up to
  bound N*". Loop-free proofs stay complete (the verdict text distinguishes).
- **Concrete interpreter caps loops at `bound`** and difftest skips inputs that
  exceed it — so difftest explores exactly the in-bound domain the symbolic
  stage proves over, and the two never contradict.

## What preconditions deliver

- A leading `assume(<bool expr>)` in a function declares an input precondition
  (`assume` is a no-op at runtime, so files stay runnable). The CLI also takes
  repeatable `--assume "EXPR"`.
- difftest **filters** inputs violating any precondition; the solver **adds**
  them as constraints. So both stages reason over the same restricted domain.
- The verdict carries a `precondition: ...` note, and EQUIVALENT wording adjusts
  ("complete over inputs satisfying the precondition").

### Subtlety found while building this (loop-bound overflow)

The first in-bound condition (`Not(start + bound < stop)`) was unsound: at
`n = INT_MAX`, `range(n + 1)` overflows to an *empty* range, which it wrongly
counted as in-bounds, yielding a bogus counterexample. Fixed: in-bound now
requires `stop ∈ [start, start + bound]` with no index-window wrap, applied
identically in the concrete cap and the symbolic assumption. A nice consequence:
`sum_to_n` (`range(n + 1)`) proves EQUIVALENT up to bound *without* needing the
precondition, because the divergent negatives fall outside the in-bound window.
The precondition machinery is demonstrated by identity-vs-abs instead.

## What M2 arrays deliver

- `list[int]` inputs modeled as a Z3 `Array(BV, BV)` plus a symbolic `length`,
  with a well-formedness constraint `0 <= length <= bound` added to the query
  (the bounded-array domain). difftest generates Python lists of length 0..bound.
- `len(xs)`, `for x in xs` (length-bounded so always within the unroll bound),
  and `xs[i]` reads. Array verdicts read "holds within bound: lists up to length N".
- Merge fix: loop/if env merges touch only the (scalar) written variables, so
  immutable list params aren't run through the scalar `ite` merge.

## What runtime-error modeling delivers (sound `xs[i]` and division)

- Each function summary is now `(output, error)`. `error` is the disjunction of
  every **guarded** runtime error — an out-of-bounds `xs[i]` (`Not(0 <= i <
  len)`) or a divide/modulo by zero — each `And`-ed with the **path condition**
  under which it executes. The interpreter threads `pc` through every node,
  refining it on `if` branches, loop-iteration guards, and `and`/`or`
  short-circuits (so `i < len(xs) and xs[i] > 0` is error-free).
- Equivalence query: `error_o != error_c  OR  (¬error_o ∧ ¬error_c ∧ out_o !=
  out_c)`. A rewrite that crashes where the original didn't is a counterexample;
  one that *avoids* a crash the original had is too.
- This removed the earlier UNKNOWN fallbacks: non-constant divisors and `xs[i]`
  are now proven (or refuted) soundly — no "assume in-bounds" caveat. Total
  fixed-width `//`/`%` and `Select` are well-defined; the garbage value under an
  error is never compared (guarded by `¬error`).

## What M3 benchmarks deliver

- `benchmarks/bench_recall.py` — runs `check` over all fixtures, tabulates
  verdict vs. `EXPECTED`, and exits non-zero on any unsound verdict (false
  EQUIVALENT / false COUNTEREXAMPLE). Currently 11/11 match, 0 unsound.
- `benchmarks/bench_scaling.py` — solver time vs. `--bound` on the loop/array
  fixtures (sub-100ms through bound 32).
- `tests/test_benchmarks.py` locks the "no unsound verdicts / fully decided"
  invariant into the suite.

## Next actions (options)

- **Finish M3**: a README demo image of the midpoint-overflow catch; a curated
  gallery of real AI-refactor pairs beyond the unit fixtures.
- **Language coverage**: `return`/`break`/`continue` inside loops (thread an
  "already-returned" guard through unrolling); list *outputs* (functions that
  build and return a list); bounded strings.
- **M4 stretch**: counterexample minimization; a C-subset front end; pluggable
  CVC5 backend behind the solver interface.

## Open design decisions (resolve before/while building the symbolic core)

From the foundational doc §8. Recommendations noted; nothing is locked.

1. **Symbolic layer: build-your-own vs. existing tools.** ✅ **Resolved: built-our-own** mini symbolic interpreter (`symbolic.py`) over the IR subset, emitting Z3 directly. Owns the "from scratch" signal.
2. **Python subset grammar.** *(M0: settled for straight-line/branching code.)* Implemented: `def` (annotated positional params), `return`, name/aug assignment, `if/elif/else`, conditional expressions, int/bool arithmetic (`+ - * // %`, unary `-`), comparisons (incl. chained), `and/or/not`. Loops + arrays deferred to M2. Everything else → `UnsupportedConstruct`.
3. **Integer model.** ✅ **Resolved: fixed-width bitvectors** (catches overflow — the killer demo). M0's concrete interpreter wraps to `--int-width` two's-complement; M1's Z3 model must match.
4. **Counterexample decoding + minimization.** Decode Z3 model → concrete inputs for M1; minimization (shrink to smallest failing input) deferred to M4.

## Changelog

- **2026-06-25** — **M7: `break` / `continue`.** Each loop owns a `broken`
  (accumulates across iterations, stops the loop) and a per-iteration `continued`,
  threaded through the state-threading interpreter alongside `returned`; the
  concrete interpreter uses exceptions. Parser rejects them outside a loop. Added
  break/continue tests + `has_negative` example. Tests: 124 pass, 0 unsound.
- **2026-06-25** — **M6: list outputs.** List literals + `+` concatenation; built
  lists modeled as Z3 array + length (fast append path for `r + [x]`, general
  `concat`); output length bounded to `bound`; element-wise output equivalence;
  return-type-mismatch → ERROR. map/filter/identity-rebuild verify; off-by-one
  map, `>`/`>=` filter, non-commutative concat give counterexamples. Added
  `map_double` + `list_filter_bug` fixtures and `keep_positives` example. Tests:
  121 pass, 14/14 fixtures, 0 unsound.
- **2026-06-25** — **M5: counterexample minimization.** Symbolic counterexamples
  are shrunk (shortest list, then scalars→0) via cheap incremental solver calls;
  `--no-minimize` flag; `counterexample minimized` note. (Note: `z3.Optimize` was
  tried first and hung over bitvectors — replaced with the iterative approach.)
  difftest witnesses stay boundary-minimal. Tests: 108 pass.
- **2026-06-25** — **M4: early exit (`return` in loops).** Rewrote the symbolic
  interpreter as one state-threading pass carrying `(env, returned, return_value)`,
  so `return` works anywhere (including loops); fall-off-end folded into the error
  condition. Parser allows return-in-loop (still rejects `break`/`continue`). Added
  `contains` fixture + `all_positive` example + early-exit tests. Tests: 105 pass,
  12/12 fixtures, 0 unsound.
- **2026-06-25** — **M3 complete: demo gallery + roadmap.** `examples/` gallery of
  5 realistic AI-refactor pairs (`run_gallery.py`, pinned by `test_examples.py`);
  committed `docs/demo.svg` leading the README; deduped precondition notes.
  Rewrote ROADMAP into an ordered plan with "done when" criteria (next: M4 early
  exit). Tests: 95 pass.
- **2026-06-25** — **Sound `xs[i]` + division in proofs.** Each function summary
  is now `(output, error)`; out-of-bounds access and divide-by-zero are modeled
  as path-condition-guarded runtime errors, and equivalence requires matching
  error behavior. Removed the UNKNOWN fallbacks for indexing / non-constant
  divisors. Added `array_first` fixture (guarded indexing proven). Tests: 94 pass,
  11/11 fixtures, 0 unsound.
- **2026-06-25** — **M3 benchmarks landed.** `bench_recall.py` (recall table +
  zero-unsound-verdict gate, 11/11 match) and `bench_scaling.py` (time vs. bound);
  `test_benchmarks.py` guards the invariant. Tests: 87 pass.
- **2026-06-25** — **M2 arrays landed.** `list[int]` inputs as bounded Z3 arrays +
  symbolic length; `len(xs)` and `for x in xs` proven; `xs[i]` reads in difftest
  (declined in proofs for soundness). Env-merge fix so list params skip scalar
  merges. 3 array fixtures. Verdict notes array-length bound. Tests: 85 pass.
- **2026-06-25** — **Input preconditions landed.** Leading `assume(<expr>)` +
  CLI `--assume`; difftest filters and the solver constrains over the precondition
  domain; verdict notes the precondition. Fixed a loop-bound-overflow soundness
  bug in the in-bound condition (now requires a non-wrapping index window).
  `sum_to_n` re-added as an EQUIVALENT fixture. Tests: 66 pass.
- **2026-06-25** — **M2 loops landed.** `for ... in range(...)` in IR/parser;
  symbolic loop unrolling with guarded iterations + in-bound (BMC) assumptions;
  concrete interpreter caps loops at `bound` so difftest stays in the in-bound
  domain. Loop verdicts read "up to bound N"; loop-free stay complete. Fixtures
  `loop_reorder` (EQ) and `loop_off_by_one` (CX) added; `sum_to_n` deferred until
  input preconditions exist. Tests: 57 pass.
- **2026-06-25** — **M1 complete (symbolic core).** `symbolic.py` lowers loop-free
  functions to Z3 bitvector expressions via continuation-passing path merging;
  Python-faithful floor `//`/`%`; `solver.py` solves the equivalence query and
  decodes counterexamples; `check()` escalates difftest → symbolic with a sound
  UNKNOWN fallback for unmodeled constructs. Resolved §8 decision 1 → built our
  own interpreter. z3-solver added. Tests: 43 pass.
- **2026-06-25** — **M0 complete.** IR parser + v1 subset validation; fixed-width
  two's-complement concrete interpreter; differential tester (boundary + random);
  `check()` wired (COUNTEREXAMPLE/UNKNOWN/ERROR); CLI parse→check→report with exit
  codes. Fixtures reworked to 4 pairs that are correct under the fixed-width
  domain. Tests: 35 pass, 2 xfail (M1). Resolved §8 decision 3 → bitvector ints.
- **2026-06-25** — Project scaffolded: repo layout per foundational doc §5, packaging (`pyproject.toml`), README/ROADMAP/PROGRESS, package stubs with interfaces, `Verdict` data model, seed fixtures, and test skeleton.
