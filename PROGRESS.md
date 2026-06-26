# Progress

Running log of where the build is and what's next. Keep this honest — it's the working memory between build sessions.

**Current phase:** M0, M1 complete ✅; M2 loops + preconditions landed — next: arrays

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
| CLI | `src/congruent/cli.py` | ✅ parse → check → report, `--assume`, exit codes 0/1/2 |
| Verdict formatting | `src/congruent/report.py` | ✅ done (all four statuses) |
| Fixtures (eval set) | `tests/fixtures/` | ✅ 7 pairs (3 CX, 4 EQ; incl. loops + a precondition) |
| Tests | `tests/` | ✅ 66 pass |

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

## Next actions (rest of M2)

1. **`list[int]` arrays**: fixed-length symbolic arrays — indexing, `len`,
   `for x in xs`. Decide Z3 arrays vs. fixed-length element vectors.
2. Consider `return` inside loops (needs the CPS merge to thread an
   "already-returned" guard through unrolling).

## Open design decisions (resolve before/while building the symbolic core)

From the foundational doc §8. Recommendations noted; nothing is locked.

1. **Symbolic layer: build-your-own vs. existing tools.** ✅ **Resolved: built-our-own** mini symbolic interpreter (`symbolic.py`) over the IR subset, emitting Z3 directly. Owns the "from scratch" signal.
2. **Python subset grammar.** *(M0: settled for straight-line/branching code.)* Implemented: `def` (annotated positional params), `return`, name/aug assignment, `if/elif/else`, conditional expressions, int/bool arithmetic (`+ - * // %`, unary `-`), comparisons (incl. chained), `and/or/not`. Loops + arrays deferred to M2. Everything else → `UnsupportedConstruct`.
3. **Integer model.** ✅ **Resolved: fixed-width bitvectors** (catches overflow — the killer demo). M0's concrete interpreter wraps to `--int-width` two's-complement; M1's Z3 model must match.
4. **Counterexample decoding + minimization.** Decode Z3 model → concrete inputs for M1; minimization (shrink to smallest failing input) deferred to M4.

## Changelog

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
