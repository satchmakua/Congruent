# Congruent — Foundational Document

**Working codename:** Congruent (rename freely)
**One line:** Given an original function and an AI-rewritten one, *prove* behavioral equivalence within bounds, or return the concrete input that breaks it.
**Status:** Pre-implementation. This doc is the seed for future build sessions — read it fully before coding.

---

## 1. Thesis (why this project)
Coding agents constantly refactor/migrate/optimize code. Nobody can rigorously answer *"did this preserve behavior?"* — they rely on tests and vibes. Equivalence checking (the author's EDA/Synopsys background) gives a real answer. This sits on a rare intersection: formal methods + frontier-AI trust. That makes it un-clonable by generalists and high-signal for hiring.

**What it proves about the builder:** systems/CS depth (symbolic execution, bounded model checking, SMT) *and* AI fluency (exists to make AI codegen trustworthy) — both at once.

---

## 2. Scope discipline (the make-or-break)
Verifying arbitrary code = research career, not a repo. **v1 stays deliberately narrow and honest.**

**In scope (v1):**
- Pure functions (no I/O, no global mutation, deterministic).
- Bounded input domains: machine ints, bools, fixed-length arrays/lists, optionally bounded strings.
- Two outputs only: **`EQUIVALENT up to bound N`** or **`COUNTEREXAMPLE: <concrete input>`** (with both functions' differing outputs).
- One source language to start: **Python** (subset). C as a stretch.

**Out of scope (v1, state explicitly in README):** unbounded loops/recursion without a bound, floating-point exactness, side effects, concurrency, heap aliasing, full language semantics. These are roadmap, not failure.

**The honesty IS the sophistication.** "Bounded, proof-or-counterexample, no magic" is the credibility signal. Never oversell soundness.

---

## 3. Approach (layered, escalating cost)
Run cheap checks first, escalate to expensive proof only when needed:

1. **Differential testing (fast prefilter).** Property-based random + boundary inputs (hypothesis-style). Kills obvious non-equivalence in milliseconds. If a counterexample is found here, done.
2. **Symbolic execution → SMT (the core).** Translate both functions' bounded behavior into logical constraints; assert outputs differ; ask the solver. `UNSAT` = equivalent within bound. `SAT` = the model *is* the counterexample (decode it back to concrete inputs).
3. **Bounded model checking for loops.** Unroll loops/recursion to depth k; verify equivalence up to that bound; report the bound honestly.

**Backend:** Z3 (via `z3-solver` Python bindings) for v1. Keep the solver interface abstracted so CVC5/others can slot in later.

**Two viable implementation paths for the symbolic layer — pick in first build session:**
- **(A) Build-your-own** mini symbolic interpreter over a Python AST subset → emits Z3 constraints. *More work, far higher signal, fully owns the "from scratch" story.* **Recommended.**
- **(B) Stand on `crosshair`/`klee`-style existing symbolic tools** and focus value on the equivalence harness + counterexample UX. Faster, lower depth signal. Fallback if time-boxed.

---

## 4. Architecture
```
input: fn_original, fn_candidate (source) + bound config
  │
  ├─ parse → normalized IR (typed AST subset, both fns)
  │
  ├─ Stage 1: differential tester ──► counterexample? ─► REPORT
  │
  ├─ Stage 2: symbolic engine
  │     • symbolically execute each fn over fresh symbolic inputs
  │     • collect path constraints + output expressions
  │     • build: (inputs equal) ∧ (outputs differ)
  │     └─ Z3 solve
  │           • UNSAT → EQUIVALENT up to bound N
  │           • SAT   → decode model → COUNTEREXAMPLE
  │
  └─ REPORT (verdict, bound, counterexample, solver stats, caveats)
```
**Core data structures:** typed IR node; symbolic value (Z3 expr + type); path condition; verdict object (`status`, `bound`, `counterexample`, `solver_time`, `assumptions`).

---

## 5. Repo layout
```
congruent/
  src/congruent/
    __init__.py
    ir.py            # AST → normalized typed IR
    symbolic.py      # symbolic interpreter → Z3 exprs
    difftest.py      # property-based prefilter
    solver.py        # Z3 abstraction + model decoding
    equiv.py         # orchestration + escalation logic
    report.py        # verdict formatting
    cli.py           # `congruent a.py:f b.py:g --bound 8`
  tests/
    fixtures/        # equivalent + non-equivalent fn pairs (the eval set)
    test_equiv.py
  benchmarks/        # timing vs bound size; recall on known pairs
  README.md          # lead with scope honesty + a killer demo gif
  ROADMAP.md         # the out-of-scope list as future milestones
  pyproject.toml     # packaging, semver, CI from day one
```

---

## 6. Milestones
- **M0 — Walking skeleton:** CLI parses two functions; Stage-1 diff tester only; verdict object + report. Ship this.
- **M1 — Symbolic core:** integer/bool arithmetic + branches → Z3; UNSAT/SAT → verdict; decode counterexamples. *This is the credibility milestone.*
- **M2 — Bounded loops + arrays:** unroll to depth k; fixed-length arrays; report bounds.
- **M3 — Demo + benchmarks:** curated gallery of real AI-refactor pairs (some truly equivalent, some subtly broken — e.g., off-by-one, overflow, reordered short-circuit); README demo showing a caught bug; timing-vs-bound charts.
- **M4 (stretch):** C subset; LLM-suggested refactors auto-checked; "minimize counterexample" (shrink to smallest failing input).

**Ship publicly at M1–M3.** Don't wait for completeness; the honest bounded tool is the product.

---

## 7. The demo that lands with reviewers
A README example where an LLM "optimizes" a function and Congruent emits the exact integer input where the optimization silently diverges (classic: integer overflow, or `<=` → `<`). One screenshot of *proof-or-counterexample* on a real AI refactor communicates the whole value instantly.

---

## 8. Key decisions for the next session (resolve first)
1. **Path A vs B** for the symbolic layer (recommend A — build the mini symbolic interpreter).
2. Exact **Python subset** grammar for v1 (which statements/operators allowed).
3. Integer model: fixed-width bitvectors (catches overflow — *preferred*, more impressive) vs unbounded ints (simpler, misses overflow bugs).
4. Counterexample decoding + minimization strategy.

---

## 9. Positioning notes (don't lose these)
- Frame as **trust infrastructure for AI-generated code**, not a generic verifier.
- Lead every doc with scope honesty — it reads as senior, not limited.
- This compounds with the author's private formal-methods-flavored work **without exposing any of it.**
- Alt high-value sibling project if adoption > differentiation is the goal: the **agent time-travel debugger** (bigger built-in audience). Congruent is the *differentiation* play and uses the rare EDA edge.

---
*Next session: start at M0, resolve §8 decision 1, then build the §4 symbolic core toward M1.*
