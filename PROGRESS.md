# Progress

Running log of where the build is and what's next. Keep this honest — it's the working memory between build sessions.

**Current phase:** M0 — Walking skeleton (scaffolding complete; engine stages stubbed)

## State of the tree

| Component | File | Status |
| --- | --- | --- |
| Verdict / Counterexample data model | `src/congruent/equiv.py` | ✅ defined |
| CLI arg parsing & spec parsing | `src/congruent/cli.py` | ✅ parses; engine call is WIP-guarded |
| Verdict formatting | `src/congruent/report.py` | ✅ basic formatter |
| AST → typed IR | `src/congruent/ir.py` | ⬜ stub (interfaces + `UnsupportedConstruct`) |
| Differential tester | `src/congruent/difftest.py` | ⬜ stub |
| Symbolic interpreter → Z3 | `src/congruent/symbolic.py` | ⬜ stub |
| Z3 abstraction + model decode | `src/congruent/solver.py` | ⬜ stub |
| Orchestration / escalation | `src/congruent/equiv.py` | ⬜ `check()` raises `NotImplementedError` |
| Fixtures (eval set) | `tests/fixtures/` | ✅ seeded with 2 examples |
| Tests | `tests/test_equiv.py` | ✅ import + fixture-loader skeleton |

## Next actions (toward M0 → M1)

1. Implement `ir.parse_function` for the v1 subset; raise `UnsupportedConstruct` on anything outside it.
2. Implement `difftest.find_counterexample` (random + boundary inputs from the typed signature).
3. Wire `equiv.check` to run difftest, return a `Verdict`; unblock the CLI happy path.
4. Begin `symbolic` + `solver` (M1) — the credibility milestone.

## Open design decisions (resolve before/while building the symbolic core)

From the foundational doc §8. Recommendations noted; nothing is locked.

1. **Symbolic layer: build-your-own vs. existing tools.** → **Recommended: build-your-own** mini symbolic interpreter over the Python AST subset (owns the "from scratch" signal). Fall back to a `crosshair`/`klee`-style tool only if time-boxed.
2. **Python subset grammar.** Which statements/operators are allowed in v1. Start: `def` (annotated positional params), `return`, `if/else`, int/bool arithmetic + comparisons + logical ops, bounded `for range`. Everything else → `UnsupportedConstruct`.
3. **Integer model.** → **Recommended: fixed-width bitvectors** (catches overflow — the killer demo) over unbounded ints.
4. **Counterexample decoding + minimization.** Decode Z3 model → concrete inputs for M1; minimization (shrink to smallest failing input) deferred to M4.

## Changelog

- **2026-06-25** — Project scaffolded: repo layout per foundational doc §5, packaging (`pyproject.toml`), README/ROADMAP/PROGRESS, package stubs with interfaces, `Verdict` data model, seed fixtures, and test skeleton.
