# Progress

Running log of where the build is and what's next. Keep this honest — it's the working memory between build sessions.

**Current phase:** M0 complete ✅ — next up is M1 (symbolic core)

## State of the tree

| Component | File | Status |
| --- | --- | --- |
| Verdict / Counterexample data model | `src/congruent/equiv.py` | ✅ done |
| Orchestration / escalation | `src/congruent/equiv.py` | ✅ `check()` runs Stage 1; ERROR on signature mismatch |
| AST → typed IR + subset validation | `src/congruent/ir.py` | ✅ done (loud `UnsupportedConstruct`) |
| Fixed-width concrete interpreter | `src/congruent/difftest.py` | ✅ done (two's-complement, catches overflow) |
| Differential tester | `src/congruent/difftest.py` | ✅ done (boundary + random generation) |
| CLI | `src/congruent/cli.py` | ✅ parse → check → report, exit codes 0/1/2 |
| Verdict formatting | `src/congruent/report.py` | ✅ done (EQUIVALENT/COUNTEREXAMPLE/UNKNOWN/ERROR) |
| Symbolic interpreter → Z3 | `src/congruent/symbolic.py` | ⬜ stub (M1) |
| Z3 abstraction + model decode | `src/congruent/solver.py` | ⬜ stub (M1) |
| Fixtures (eval set) | `tests/fixtures/` | ✅ 4 pairs (2 CX, 2 EQ) |
| Tests | `tests/` | ✅ 35 pass, 2 xfail (M1 equivalence proofs) |

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

## Next actions (M1 — symbolic core, the credibility milestone)

1. `symbolic.summarize` — interpret the IR over fresh Z3 symbols (bitvectors for
   ints), forking on branches, collecting (path condition, output) per path.
   Mirror the concrete interpreter's semantics exactly so M0 and M1 agree.
2. `solver.prove_equivalence` — assert *inputs equal ∧ outputs differ*; `UNSAT`
   → EQUIVALENT, `SAT` → decode model → COUNTEREXAMPLE, `unknown` → UNKNOWN.
3. `equiv.check` — escalate difftest → symbolic; flip the `test_equivalent_
   fixtures_are_proven` xfail to passing.

## Open design decisions (resolve before/while building the symbolic core)

From the foundational doc §8. Recommendations noted; nothing is locked.

1. **Symbolic layer: build-your-own vs. existing tools.** → **Recommended: build-your-own** mini symbolic interpreter over the Python AST subset (owns the "from scratch" signal). Fall back to a `crosshair`/`klee`-style tool only if time-boxed.
2. **Python subset grammar.** *(M0: settled for straight-line/branching code.)* Implemented: `def` (annotated positional params), `return`, name/aug assignment, `if/elif/else`, conditional expressions, int/bool arithmetic (`+ - * // %`, unary `-`), comparisons (incl. chained), `and/or/not`. Loops + arrays deferred to M2. Everything else → `UnsupportedConstruct`.
3. **Integer model.** ✅ **Resolved: fixed-width bitvectors** (catches overflow — the killer demo). M0's concrete interpreter wraps to `--int-width` two's-complement; M1's Z3 model must match.
4. **Counterexample decoding + minimization.** Decode Z3 model → concrete inputs for M1; minimization (shrink to smallest failing input) deferred to M4.

## Changelog

- **2026-06-25** — **M0 complete.** IR parser + v1 subset validation; fixed-width
  two's-complement concrete interpreter; differential tester (boundary + random);
  `check()` wired (COUNTEREXAMPLE/UNKNOWN/ERROR); CLI parse→check→report with exit
  codes. Fixtures reworked to 4 pairs that are correct under the fixed-width
  domain. Tests: 35 pass, 2 xfail (M1). Resolved §8 decision 3 → bitvector ints.
- **2026-06-25** — Project scaffolded: repo layout per foundational doc §5, packaging (`pyproject.toml`), README/ROADMAP/PROGRESS, package stubs with interfaces, `Verdict` data model, seed fixtures, and test skeleton.
