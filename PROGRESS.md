# Progress

Running log of where the build is and what's next. Keep this honest — it's the working memory between build sessions.

**Current phase:** M0–M7 complete ✅ **plus the LLM closed-loop stretch** (`refine.py`) — the v1 product is complete **and validated live** (real model caught proposing the midpoint overflow, realistic-scale rewrite proven — `docs/live_run.md`). Incl. CVC5 cross-check, bounded strings, C front end; stress-tested + polished (a fuzzer + two independent oracles, seven adversarial-audit rounds, ruff + mypy clean, CI). See [ROADMAP.md](ROADMAP.md).

## State of the tree

| Component | File | Status |
| --- | --- | --- |
| Verdict / Counterexample data model | `src/congruent/equiv.py` | ✅ done |
| Orchestration / escalation | `src/congruent/equiv.py` | ✅ difftest → symbolic, sound UNKNOWN fallback |
| AST → typed IR + subset validation | `src/congruent/ir.py` | ✅ done (loud `UnsupportedConstruct`) |
| Fixed-width concrete interpreter | `src/congruent/difftest.py` | ✅ done (two's-complement, catches overflow) |
| Differential tester | `src/congruent/difftest.py` | ✅ done (boundary + random generation) |
| Symbolic interpreter → Z3 | `src/congruent/symbolic.py` | ✅ path-merge, bitvectors, floor //, **loop unrolling** |
| Z3 query + model decode | `src/congruent/solver.py` | ✅ UNSAT/SAT/unknown → Verdict; in-bound assumptions; `timeout_ms` so a hard query (nonlinear bitvector multiply) is UNKNOWN, never a hang |
| Bounded loops (`for range`) | `ir.py` / `difftest.py` / `symbolic.py` | ✅ parse + capped concrete eval + symbolic unroll |
| Input preconditions (`assume`) | `ir.py` / `difftest.py` / `symbolic.py` | ✅ filters difftest + constrains solver; CLI `--assume` |
| `list[int]` arrays | `ir.py` / `difftest.py` / `symbolic.py` | ✅ bounded arrays, `len`, `for x in xs`, `xs[i]` (proven) |
| Runtime-error modeling | `src/congruent/symbolic.py` | ✅ OOB access + divide-by-zero as guarded errors (path-condition aware) |
| CLI | `src/congruent/cli.py` | ✅ parse → check → report, `--assume`, `--timeout` (no hang on intractable queries), exit codes 0/1/2 |
| Verdict formatting | `src/congruent/report.py` | ✅ done (all four statuses) |
| Fixtures (eval set) | `tests/fixtures/` | ✅ 16 pairs (ints, loops, preconditions, arrays, indexing, early-exit, list outputs, strings) |
| Benchmarks | `benchmarks/` | ✅ recall (zero-unsound gate) + timing-vs-bound |
| Early exit (`return`/`break`/`continue`) | `src/congruent/symbolic.py` | ✅ state-threading pass; `broken`/`continued` per loop |
| Counterexample minimization | `src/congruent/solver.py` | ✅ incremental solver shrink (length, then scalars→0); `--no-minimize` |
| List outputs (build/return `list[int]`) | `ir.py` / `difftest.py` / `symbolic.py` / `solver.py` | ✅ literals + concat; map/filter verify; output length bounded |
| CVC5 cross-check backend | `src/congruent/backends.py` | ✅ SMT-LIB2 bridge; `--cross-check`; disagreement → UNKNOWN |
| Bounded `str` | `ir.py` / `difftest.py` / `symbolic.py` / `solver.py` | ✅ `SymList` `kind="char"`; literals, `len`, `==`, `+`, index, iteration |
| C front end | `src/congruent/cfront.py` | ✅ pycparser → IR; truncating `/`/`%`; CLI dispatches `.c` |
| Self-validating fuzzer | `benchmarks/fuzz.py` | ✅ random pairs (ints, loops, lists, strings) re-checked vs. concrete interp; CI guard |
| Adversarial audit | (multi-agent) | ✅ seven rounds + real-Python oracle → **36 bugs** the fuzzer missed, all fixed + pinned in `test_regressions.py` |
| Lint / types / CI | `pyproject.toml`, `.github/workflows/ci.yml` | ✅ ruff + mypy clean; GitHub Actions runs lint/types/tests/recall |
| Demo gallery | `examples/` + `docs/demo.svg` | ✅ 11 Python pairs (incl. a ~50-line realistic-scale entry + a real-codebase `numpy.polyval` entry, both live-model rewrites) + a C example; per-example bound/width + capped; runner pinned by tests |
| Real-Python oracle | `benchmarks/realpy_fuzz.py` | ✅ unparses IR→Python, diffs vs interpreter — catches bugs both stages share (found negative-indexing); runs wide to isolate *semantics* |
| Fixed-width wrapping oracle | `benchmarks/numpy_oracle.py` | ✅ independent numpy-C two's-complement scalars vs interpreter at small widths — validates the *wrapping* (exhaustive at 8-bit; agrees across 8/16/32/64) |
| LLM closed loop *(stretch)* | `src/congruent/refine.py` | ✅ AI proposes → Congruent verifies → counterexample feeds back until *proven* equivalent; pluggable rewriter (`AnthropicRewriter` / `ScriptedRewriter`), demo + tests offline; **validated live** (`docs/live_run.md`) + generic driver `examples/live_rewrite.py` |
| Tests | `tests/` | ✅ 229 pass (cvc5 / pycparser / numpy / anthropic tests skip or stub if absent) |

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
  EQUIVALENT / false COUNTEREXAMPLE). Currently 16/16 match, 0 unsound.
- `benchmarks/bench_scaling.py` — solver time vs. `--bound` on the loop/array
  fixtures (sub-100ms through bound 32).
- `tests/test_benchmarks.py` locks the "no unsound verdicts / fully decided"
  invariant into the suite.

## Next actions

**None outstanding for v1 — the backlog is empty.** Everything previously listed
here shipped: the README demo image + curated gallery (M3), `return`/`break`/
`continue` in loops (M4/M7), list outputs (M6), bounded strings (M7),
counterexample minimization (M5), the C front end and the CVC5 backend (M7), and
the LLM closed loop (stretch) — since validated against a live model
(`docs/live_run.md`).

Anything further is the deliberate **out-of-scope** list in
[ROADMAP.md](ROADMAP.md#out-of-scope-for-v1--future-work) (recursion, floats,
side effects, concurrency, heap aliasing, full Python semantics). Those are not a
backlog: refusing them is what makes a `EQUIVALENT` verdict here worth believing.
The one honest *internal* limit worth revisiting is the nonlinear-multiply wall
(symbolic-coefficient polynomials return UNKNOWN — see `benchmarks/README.md`);
that is a solver-capability ceiling, not unfinished work.

## Open design decisions — all resolved

From the foundational doc §8; kept as a record of what was decided and why.

1. **Symbolic layer: build-from-scratch vs. existing tools.** ✅ **Resolved: built from scratch** — a mini symbolic interpreter (`symbolic.py`) over the IR subset, emitting Z3 directly. Owns the "from scratch" signal.
2. **Python subset grammar.** *(M0: settled for straight-line/branching code.)* Implemented: `def` (annotated positional params), `return`, name/aug assignment, `if/elif/else`, conditional expressions, int/bool arithmetic (`+ - * // %`, unary `-`), comparisons (incl. chained), `and/or/not`. Loops + arrays deferred to M2. Everything else → `UnsupportedConstruct`.
3. **Integer model.** ✅ **Resolved: fixed-width bitvectors** (catches overflow — the killer demo). M0's concrete interpreter wraps to `--int-width` two's-complement; M1's Z3 model must match.
4. **Counterexample decoding + minimization.** ✅ **Resolved: both shipped** — Z3 model → concrete inputs in M1; minimization (originally slated for M4) landed in **M5** as an incremental solver shrink in `solver.py` (list length, then scalars→0), with `--no-minimize` to skip it.

## Changelog

- **2026-07-14** — **Final polish pass (v1 backlog empty).** M0–M7 + the stretch
  all shipped and the external critique fully addressed, so: a consistency sweep
  over the finished repo, plus a 4-lens adversarial audit (43 agents; 36 findings
  confirmed, 3 rejected). **Four real bugs, all user-facing:**
  1. **The CLI could hang forever.** `cli.py` never passed `timeout_ms`, so
     `congruent` on an intractable query (32-bit polyval) spun indefinitely — the
     failure the library fix had closed everywhere *except* the primary
     interface. Added `--timeout SECONDS` (default 300, `0` = no limit): now
     `UNKNOWN … solver returned unknown: timeout` in 5s, exit 2.
  2. **The CLI crashed on a legacy Windows console.** A string counterexample
     decodes to arbitrary code points (`solver._decode_seq` → `chr(e)`), which
     cp1252 cannot encode → `UnicodeEncodeError`. Same class as the demo crash
     the critique caught, still shipping. `main()` now reconfigures stdout/stderr
     to UTF-8 (verified: `cp1252 → utf-8`, a CJK verdict prints).
  3. **The C front end accepted `break`/`continue` outside a loop** — the Python
     front end always rejected them; C silently produced an IR the two stages
     model inconsistently. `cfront` now calls `ir._check_loop_control`; pinned by
     three tests.
  4. **The real-Python oracle was partly blind.** It still tolerated "real
     returns `None`, Congruent raises" as a *documented design choice* — a design
     that changed. The clause was dead (verified: 0 mismatches at 3 seeds after
     removal) and silently disabled detection of the exact bug class the oracle
     exists for. Removed, with its two inverted docstrings.

  **Honesty fix:** an EQUIVALENT verdict always printed "equivalent up to bound
  N", even when the pair is loop-free and decided over the *whole* input space —
  underselling a complete proof and contradicting its own note two lines below.
  `Verdict.complete` now carries the fact (derived in `_scope_note`, not
  string-sniffed) and complete results say "equivalent (complete — no bound
  needed)".

  **Coverage:** the CLI had *no tests at all* despite its exit codes being a
  documented interface — added `tests/test_cli.py` (exit 0/1/2, the timeout
  anti-hang, the cp1252 crash, `--assume` reporting, missing-file).
  **Dead code:** removed `backends.available()` (zero callers), a dead
  `boundary_params` in `numpy_oracle`, and unused params in `solver._differ` /
  the decode chain / `symbolic._append`/`_concat`; `run_gallery.main` no longer
  duplicates `evaluate`'s loop. **Docs:** fixed contradictions (ROADMAP M4 said
  `break`/`continue` were "still rejected", M7 shipped them; fixtures/README said
  preconditions were inexpressible and `sum_to_n` absent while listing it 10
  lines later; `equiv.check`'s docstring still claimed the symbolic stage didn't
  exist), corrected stale counts (154→229 tests, 213→229, 11/11 and 10/10→16/16,
  "~4,900 pairs"→a re-measured figure), routed optional deps through the declared
  extras, documented the `oracle` extra, and added `UNKNOWN` to the README
  headline (it framed a strict binary). Tests: 229 pass, ruff + mypy clean.
- **2026-07-14** — **Independent wrapping oracle, a real-codebase entry, and a
  solver-timeout fix the entry surfaced.** Closing the last two gaps from the
  external critique. (1) **numpy wrapping oracle** (`benchmarks/numpy_oracle.py`,
  `tests/test_numpy_oracle.py`): the two's-complement wrapping — previously
  validated only by construction — is now checked against numpy's C fixed-width
  scalars, an engine that shares no code with our masking. Exhaustive over every
  8-bit operand pair (incl. INT_MIN//-1, overflow edges), and agrees across
  8/16/32/64-bit on boundary-biased fuzzing (~9% of cases actually wrap). Added
  the `oracle` extra (numpy). (2) **Real-codebase gallery entry**
  (`examples/polyval.py`): the scalar-int reduction of `numpy.polyval`'s Horner
  loop; a live model seed-optimized it (correctly adding the empty-list guard)
  and Congruent proved it EQUIVALENT — transcript in `docs/live_run.md`. (3) That
  entry exposed a real bug: `check()` had **no solver timeout**, so polyval's
  symbolic-coefficient polynomial (nonlinear bitvector multiply) made Z3 spin
  *forever* at 32-bit. Added `timeout_ms` threaded through `check` → `solver`
  (and `refine`/`live_rewrite`/gallery), so a hard query degrades to honest
  UNKNOWN, never a hang — pinned by `test_hard_query_times_out_to_unknown_not_a_hang`.
  The gallery now honors per-example `BOUND`/`INT_WIDTH` (polyval proves at
  8-bit/bound-2 in 0.8s; higher is intractable — documented in
  `benchmarks/README.md`). Also added `AnthropicRewriter` `timeout`/`max_retries`
  (the SDK's 10-min default hung the demo) and a `live_rewrite` extraction test.
  Tests: 220 pass, ruff + mypy clean.
- **2026-07-13** — **Reality contact: live runs, realistic scale, the measured
  scaling edge.** Acting on an external critique of the portfolio: (1) Ran the
  closed loop **live** — a real model (`claude-opus-4-8`) proposed the classic
  `(a+b)//2` midpoint overflow, was handed the exact failing input, and returned
  the proven-safe form; unedited transcripts in `docs/live_run.md`. Live-mode
  narration no longer reuses the scripted story (it narrated attempts a live
  model doesn't make); `DEFAULT_MODEL` and a public `AnthropicRewriter.model`
  added. (2) New `examples/live_rewrite.py` points the live loop at any
  `file.py:function`; used it on a new realistic-scale gallery entry,
  `examples/water_bill.py` — a ~50-line tiered-billing routine whose committed
  candidate is the live model's own rewrite, accepted only after proof (2
  rounds, 8.2s wall). The first live session exposed a real product gap: on
  UNKNOWN the loop said only "inconclusive" while the verdict knew the reason,
  so the model re-proposed the same unprovable form for 4 rounds (correctly
  never accepted). `_feedback` now relays the verifier's reason — pinned by
  `test_unknown_feedback_relays_the_verifiers_reason`. (3) Fixed the Windows
  cp1252 console crash (`UnicodeEncodeError` on `→`/`✓`) in the demo and gallery
  runners via stdout/stderr UTF-8 reconfigure. (4) Measured the scaling edge to
  bound 1024 and documented the honest envelope in `benchmarks/README.md`:
  interactive ≤32, sub-second ≤128, ~6×/doubling past 256, 102s worst case at
  1024. Tests: 214 pass.
- **2026-06-25** — **Stretch item shipped: the LLM closed loop (`refine.py`).**
  An LLM proposes a rewrite, Congruent verifies it, and each counterexample is fed
  back into the prompt until the rewrite is *proven* equivalent within the bound —
  the loop never accepts an unverified rewrite. The rewriter is pluggable via a
  `Rewriter` protocol: `AnthropicRewriter` drives the real API (optional
  `congruent-eq[llm]` extra, reads `ANTHROPIC_API_KEY`), and `ScriptedRewriter` keeps
  the loop fully offline for the demo and tests. `examples/closed_loop_demo.py`
  shows Congruent catching a plausible-but-wrong refactor (off-by-one closed form;
  the `(a+b)//2` midpoint overflow) and guiding the fix to a proven-equivalent
  rewrite. 9 new tests (loop convergence, counterexample feedback, never-falsely-
  verified, parse-error recovery, a fake-client API path). Tests: 213 pass. **The
  v1 product — M0–M7 plus the stretch — is complete.**
- **2026-06-25** — **Seventh round: audit of the round-6 fixes + 4 more fixes.**
  Another real-Python-grounded pass returned **8 confirmed findings, 0 false
  positives**, in 4 root causes: (33) the concrete interpreter `int()`-coerced a
  non-int where an int is required (`xs["0"]`, `-("5"+s)`, `range("1")`) —
  `int("0")` succeeds and fabricated a divergence, but Python raises TypeError;
  it now requires an int (raises otherwise). (34) `len()` was not fixed-width
  wrapped, so `len(xs + xs)` (128) diverged from `len(xs) + len(xs)` (wraps to
  −128 at width 8) — `len` now wraps. (35) a *candidate* `assume()` whose argument
  raises (`assume(xs[0] > 0)` on `[]`) was ignored — the argument evaluation is
  real behavior, now folded into the candidate's error. (36) **loop-bound scope,
  redesigned properly**: a range loop is in scope iff its *real* (un-wrapped) trip
  count ≤ bound AND every loop-variable value is representable — so `range(126,128)`
  and `range(a, a+1)` at `a=127` run their real 1–2 iterations (superseding round
  6's coarser decline), while `range(n+1)` at `n=imax` and `range(0, x//-1)` at
  `x=imin` are correctly out of scope (huge trip count). Pinned all four; added
  `and`/`or`-value + boundary coverage to both fuzzers. 204 tests, both fuzzers
  clean. **36 bugs found & fixed total.**
- **2026-06-25** — **Sixth round: audit of the round-5 fixes + 3 more fixes.** A
  fresh real-Python-grounded audit of the just-changed code returned **12 confirmed
  findings, 0 false positives**, in 3 root causes: (30) `and`/`or` were lowered to a
  **boolean** in both stages, but Python returns an **operand value** (`x or 5` is
  `x` or `5`, `5 and 3` is `3`) → false EQUIVALENT and false COUNTEREXAMPLE; both
  stages now return the operand (truthiness in a boolean context is unchanged).
  (31) a **mixed-type ternary** (`x if x>0 else "a"`, `xs if c else 5`) produced the
  round-5 `_UNDEFINED` sentinel, which reached the solver and **crashed** z3;
  `_as_bv`/`_as_bool`/`_coerce_return` now decline on it. (32) an **out-of-width
  loop-bound literal** (`range(126, 128)` at width 8) was silently wrapped by the
  symbolic stage, mis-counting the trips → wrong verdict (round 7 later refined the
  whole loop-bound scope so such a loop runs its real 2 iterations). Pinned all three, added
  `and`/`or`-value coverage to both fuzzers. 199 tests; ~28k fresh fuzz + oracle
  evaluations clean. **32 bugs found & fixed total.**
- **2026-06-25** — **Fifth round: real-Python-grounded audit + 5 more fixes.** A
  6-lens audit re-grounded in *real Python* (not the interpreter, whose shared
  blind spots gave round-3 false positives) returned **10 confirmed findings, 0
  false positives**, collapsing to 5 root causes — all fixed: (25) a variable that
  is a sequence on one branch and a scalar on the other (`if xs: y=xs else: y=0`),
  or a str on one and a list on the other, **crashed `_merge`** (or mis-merged its
  kind → false COUNTEREXAMPLE); now such a variable is undefined (declines).
  (26) a sequence used where a scalar is required (`-xs`, `xs[xs]`) **crashed
  `_as_bv`** in z3; now declines. (27) an *empty* `range(n)` (n < 0, 0 iterations)
  was wrongly excluded from scope → **false EQUIVALENT**; the loop-bound in-scope
  check now allows empty ranges. (28) a near-`imax` bound (`range(126, 127)`)
  overflowed `start + bound` and was wrongly excluded → **false EQUIVALENT**; the
  trip-count/guard arithmetic is now widened, with explicit bound-expression
  overflow detection distinguishing an empty range (in scope) from a wrapped one
  (out of scope) — keeping the Gauss-vs-loop fixture provable. (29) **falling off
  the end** (Python returns `None`) was conflated with a raised exception in both
  stages → **false EQUIVALENT**; `None` is now a distinct outcome (≠ exception,
  ≠ value; == another `None`). Pinned all five; 195 tests, both fuzzers clean.
  **29 bugs found & fixed total.**
- **2026-06-25** — **Fourth round: multi-agent adversarial audit + 3 more fixes.**
  A 6-lens Workflow audit (type/kind confusion, sequence capacity, the `_UNDEFINED`
  sentinel, errors, integer-width, freeform), each finding independently verified,
  surfaced **3 more confirmed cardinal-sin bugs** (re-verified here against *real
  Python*, since the audit's own oracle was the interpreter): (18) mixing a `str`
  and a `list[int]` under `+` (`"" + [1]`, `xs + "a"`) was modeled as element
  concatenation instead of the Python `TypeError` → **false COUNTEREXAMPLE**;
  (19) a `-> bool` return coerced its value to a truthiness bool, so `return x + y`
  made 88 and 89 both `True` → **false EQUIVALENT** (Python does not enforce the
  annotation); (20) an inner loop variable *shadowing a parameter* was excluded
  from the outer loop's merged names, so `return p` reverted to the stale param →
  **false COUNTEREXAMPLE** (now declines to UNKNOWN). The audit also flagged
  computed-sequence iteration (`for x in xs+xs`) as unsound, but that was a **false
  positive** — the interpreter's `oob` marker misled it; the tool's counterexample
  is real vs Python (one audit agent correctly rejected it). Pinned all three plus
  a guard for the false positive. Probing those fixes then surfaced **2 crash bugs**
  in the same type-confusion vein: (21) a `-> int` function that returns a *list*
  crashed merging a `SymList` with a scalar default (now declines to UNKNOWN);
  (22) `if xs:` / `not xs` crashed `_as_bool` on a sequence (now modeled as
  non-empty truthiness, `len(xs) > 0`). Two more from continued probing:
  (23) `s * 2` (sequence repetition) — valid Python but unmodeled — was raised as
  an error by the interpreter, so `s*2` vs `s+s` (equal!) was a **false
  COUNTEREXAMPLE**; now skipped as out-of-model → UNKNOWN. (24) a nested list
  literal `[xs]` (`list[list[int]]`, outside the model) both crashed the symbolic
  stage and was fabricated as an error by the interpreter → **false
  COUNTEREXAMPLE**; both stages now decline. Added truthiness/indexing coverage to
  both fuzzers. 187 tests; ~40k fresh fuzz + oracle evaluations clean.
  **24 bugs found & fixed total** (false verdicts + crashes), all pinned.
- **2026-06-25** — **Real-Python differential oracle + negative-indexing fix
  (bug #17).** Every prior check (fuzzer, audits, regression harness) used the
  concrete interpreter as ground truth — so a flaw *shared* by both stages was
  invisible. Added an independent oracle: unparse each generated IR function to
  real Python source, `exec` it, and diff against the interpreter (small values
  at width 64 so nothing overflows, isolating semantics from the intended
  wrapping). It immediately caught that **both stages modeled `xs[-1]` as
  out-of-range**, so `return xs[-1]` was proven EQUIVALENT to a function that
  always crashes — a **false EQUIVALENT vs real Python** (`f([5])=5`, not an
  error). Implemented Python negative indexing (`-n ≤ i < n`, offset `i+n`) in
  *both* stages. The main fuzzer never generated `xs[i]` at all — added an
  indexing family (incl. negative indices); 12k pairs + 280k oracle evaluations
  now clean. Kept the oracle as a permanent benchmark (`benchmarks/realpy_fuzz.py`
  + `test_realpy_oracle.py`). Documented that fall-off-the-end is deliberately
  modeled as an error (the only remaining interpreter-vs-Python divergence, and a
  defensible one). Tests: 176 pass. **17 soundness bugs found & fixed total.**
- **2026-06-25** — **Third audit round (auditing the fixes' fixes) + 2 more
  soundness fixes.** Reasoning about the round-2 changes surfaced **2 more
  confirmed cardinal-sin bugs**: (1) a concat *chain* (`xs+xs+xs`, length 3·bound)
  still injected the fixed `length ≤ 2·bound` cap, excluding a length-`bound`
  input from the query → a **false EQUIVALENT** (`check` certified `f`≡`g` though
  they diverge at `xs=[42,42]`); (2) sequence `==` compared a `str` and a
  `list[int]` by contents, so `s == xs` was satisfiable → a **false
  COUNTEREXAMPLE** (Python: a str never equals a list). Root-caused the cap bug
  properly: each `SymList` now carries its own *static max length* (`cap`, summed
  compositionally), so `length ≤ cap` holds by construction and no in-scope input
  is ever assumed away; pathological growth (doubling a list in a loop) *declines*
  to UNKNOWN via a slot ceiling instead of blowing up. Pinned both (+ companions)
  in `test_regressions.py`; ran ruff + mypy clean and 12k fuzz pairs (0 unsound).
  Tests: 173 pass. **16 soundness bugs found & fixed total.**
- **2026-06-25** — **Re-audit of the fixes + 6 more soundness fixes.** A second
  adversarial pass — this time targeting the *first audit's fixes* (a fix can
  introduce a new bug) — found **6 more confirmed soundness bugs**, each verified
  by a repro: (1) a loop variable nested in an `if` and shadowing a param stayed
  in scope after the loop → `return i` reverted to the stale param → false
  EQUIVALENT; (2) a precondition that itself raises (`assume(100//x < 0)`) wasn't
  excluded from the domain; (3) `str` silently wrapped code points for
  `int_width < 22` → the symbolic stage now *declines* (UNKNOWN) instead of
  certifying; (4) sequence `==` compared only `bound` slots, missing a computed
  tail; (5) the output-length cap (`bound`) dropped in-scope inputs whose *output*
  exceeded it → cap raised to `2*bound`; (6) the C loop-counter-escape check was
  flow-insensitive (rejected a counter merely *read before* its loop). Introduced
  an `_UNDEFINED` sentinel so out-of-scope reads fail closed. Fixed all, pinned
  each in `test_regressions.py`, re-ran ruff + mypy clean and a 5000-trial fuzz
  batch (0 unsound). Tests: 169 pass. **14 soundness bugs found & fixed total.**
- **2026-06-25** — **Adversarial audit + soundness fixes.** A multi-agent audit
  (7 dimensions, each finding empirically verified by a repro) found **8 confirmed
  soundness bugs the ~14.5k-pair fuzzer had missed** — all in corners the fuzzer
  didn't generate: (1) `str` inputs constrained to ASCII → false EQUIVALENT on
  non-ASCII; (2) all runtime errors collapsed to one bool → difftest/symbolic
  disagreed when both raised different exception types; (3) `for x in xs+[1]`
  (computed sequence > bound) → symbolic truncated → false COUNTEREXAMPLE;
  (4) loop variable popped after the loop → fabricated NameError; (5) candidate-
  side `assume()` narrowed the domain → hid a divergence; (6) C loop-counter
  escape value mis-modeled; plus C octal-literal crash and `int f(void)` rejected.
  Fixed all, pinned each in `test_regressions.py`, extended `fuzz.py` to cover the
  missed areas (5000-trial post-fix batch clean), and corrected stale docs. Tests:
  163 pass.
- **2026-06-25** — **Stress test + polish.** Added a self-validating fuzzer
  (`benchmarks/fuzz.py`): random expression/loop pairs whose verdicts are
  independently re-checked against the concrete interpreter. Ran ~4,900 pairs +
  400 Z3/CVC5 cross-checks with **zero unsound verdicts and zero backend
  disagreements**; wired a fast deterministic batch into `test_fuzz.py`. Made
  ruff and mypy clean (pragmatic mypy config; the dynamic concrete interpreter is
  exempt), and added a GitHub Actions CI workflow. Tests: 154 pass.
- **2026-06-25** — **M7 stretch: C front end.** `cfront.py` lowers a C function
  (via `pycparser`) to the same IR, so the whole engine is reused. Added C
  truncating `/`/`%` (IR ops `c/`/`c%`) distinct from Python floor `//`/`%`; the
  CLI dispatches on the `.c` extension; comments/directives are stripped before
  parsing. `pycparser` added as an optional extra; `examples/midpoint.c` added.
  Tests: 153 pass.
- **2026-06-25** — **M7: bounded strings.** `str` modeled as a bounded sequence of
  code points (a character is a length-1 string), reusing `SymList` via a `kind`
  tag ("int" vs "char") so list and string code share nearly everything. Literals,
  `len`, `==`/`!=`, `+`, indexing, iteration; string elements constrained to ASCII
  so counterexamples decode cleanly. cvc5 cross-check handles the array-backed
  queries with `arrays-exp`. Added string fixtures + `string_greeting` example.
  Tests: 143 pass, 16/16 fixtures, 0 unsound.
- **2026-06-25** — **M7: CVC5 cross-check backend.** New `backends.py`: Z3 stays
  primary (encoding + models), CVC5 independently re-decides the same query via an
  SMT-LIB2 round-trip under `--cross-check`; agreement is noted, disagreement
  downgrades to UNKNOWN. `cvc5` added as an optional extra. Tests skip when cvc5
  is absent. Tests: 130 pass.
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
