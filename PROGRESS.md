# Progress

Running log of where the build is and what's next. Keep this honest — it's the working memory between build sessions.

**Current phase:** M0 + M1 complete ✅ — next up is M2 (bounded loops + arrays)

## State of the tree

| Component | File | Status |
| --- | --- | --- |
| Verdict / Counterexample data model | `src/congruent/equiv.py` | ✅ done |
| Orchestration / escalation | `src/congruent/equiv.py` | ✅ difftest → symbolic, sound UNKNOWN fallback |
| AST → typed IR + subset validation | `src/congruent/ir.py` | ✅ done (loud `UnsupportedConstruct`) |
| Fixed-width concrete interpreter | `src/congruent/difftest.py` | ✅ done (two's-complement, catches overflow) |
| Differential tester | `src/congruent/difftest.py` | ✅ done (boundary + random generation) |
| Symbolic interpreter → Z3 | `src/congruent/symbolic.py` | ✅ done (path-merge, bitvectors, floor //) |
| Z3 query + model decode | `src/congruent/solver.py` | ✅ done (UNSAT/SAT/unknown → Verdict) |
| CLI | `src/congruent/cli.py` | ✅ parse → check → report, exit codes 0/1/2 |
| Verdict formatting | `src/congruent/report.py` | ✅ done (all four statuses) |
| Fixtures (eval set) | `tests/fixtures/` | ✅ 4 pairs (2 CX, 2 EQ) |
| Tests | `tests/` | ✅ 43 pass |

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

## Next actions (M2 — bounded loops + arrays)

1. IR + both interpreters: support `for ... in range(...)`; unroll to depth
   `bound` symbolically; cap concrete loop iterations.
2. Introduce **bounded input domains** so loop-vs-closed-form pairs (e.g.
   `sum_to_n`) can be asked over `0 <= n <= bound`; report the bound honestly.
3. `list[int]` as fixed-length symbolic arrays (Z3 arrays or element vectors).
4. Re-add the `sum_to_n` fixture under the bounded-domain semantics.

## Open design decisions (resolve before/while building the symbolic core)

From the foundational doc §8. Recommendations noted; nothing is locked.

1. **Symbolic layer: build-your-own vs. existing tools.** ✅ **Resolved: built-our-own** mini symbolic interpreter (`symbolic.py`) over the IR subset, emitting Z3 directly. Owns the "from scratch" signal.
2. **Python subset grammar.** *(M0: settled for straight-line/branching code.)* Implemented: `def` (annotated positional params), `return`, name/aug assignment, `if/elif/else`, conditional expressions, int/bool arithmetic (`+ - * // %`, unary `-`), comparisons (incl. chained), `and/or/not`. Loops + arrays deferred to M2. Everything else → `UnsupportedConstruct`.
3. **Integer model.** ✅ **Resolved: fixed-width bitvectors** (catches overflow — the killer demo). M0's concrete interpreter wraps to `--int-width` two's-complement; M1's Z3 model must match.
4. **Counterexample decoding + minimization.** Decode Z3 model → concrete inputs for M1; minimization (shrink to smallest failing input) deferred to M4.

## Changelog

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
