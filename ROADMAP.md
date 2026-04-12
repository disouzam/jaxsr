# JAXSR Roadmap: discopt Integration

> **Status:** Deferred until discopt/ripopt APIs stabilize.
> **Last reviewed:** 2026-04-11

## Motivation

jaxsr's `exhaustive_search()` enumerates all C(n, k) basis-function subsets — globally
optimal but capped at 100,000 combinations. For 30 basis functions with `max_terms=5`,
that's 142,506 combinations (already over the limit). The greedy forward strategy scales
but is a local heuristic that can miss the true best subset.

Best-subset regression is a textbook Mixed-Integer Quadratic Program (MIQP). discopt
can solve it to global optimality via Branch-and-Bound, pruning the search tree so only
a fraction of subsets are evaluated. This unlocks 50-100+ basis function problems that
are currently out of reach.

---

## Tier 1 — MIQP Best Subset Selection

**Impact: Very high | Feasibility: High**

### Problem

```
minimize  beta^T (Phi^T Phi) beta  -  2 (Phi^T y)^T beta
subject to:
  |beta_i| <= M * z_i        (big-M linking)
  sum(z_i) <= k               (sparsity)
  z_i in {0, 1}               (binary selection)
```

jaxsr already precomputes the Gram matrix `Phi^T Phi` and `Phi^T y` in `_precompute_gram`
(`selection.py:198`) — exactly the data needed for the MIQP objective.

### Implementation steps

1. **Optional dependency.** Add to `pyproject.toml`:
   ```toml
   [project.optional-dependencies]
   miqp = ["discopt>=0.3"]   # pin to stable release when available
   ```

2. **New strategy function.** Add `miqp_subset_selection()` in `selection.py` with the
   same signature as `exhaustive_search()`:
   - Build a `dm.Model` with `beta` (continuous), `z` (binary), big-M linking constraints,
     sparsity constraint, and quadratic objective from the precomputed Gram matrix.
   - Solve for each k = 1..max_terms, evaluate IC on each, return `SelectionPath`.
   - Big-M heuristic: run OLS on the full model, set M = 10 * max(|beta_OLS|).

3. **Register strategy.** Add `"miqp": miqp_subset_selection` to the `strategies` dict
   in `select_features()` (line ~1529).

4. **Guard the import.** Use try/except around `import discopt.modeling`; raise a helpful
   `ImportError("Install discopt for MIQP selection: pip install jaxsr[miqp]")` if missing.

### Key files to modify

| File | Change |
|------|--------|
| `src/jaxsr/selection.py` | Add `miqp_subset_selection()`, register in dispatcher |
| `src/jaxsr/regressor.py` | Wire `strategy="miqp"` through to regressor API |
| `pyproject.toml` | Add `miqp` optional dependency group |
| `tests/test_selection.py` | Tests comparing MIQP to exhaustive on small problems |

### Verification

- MIQP and exhaustive must agree on small problems (n_basis <= 15, max_terms <= 4).
- Benchmark on medium problems (30-60 basis functions) where exhaustive hits the cap.
- `import jaxsr` must succeed without discopt installed; `strategy="miqp"` must raise
  a clear error without discopt, and work correctly with it.

---

## Tier 2 — Combined Selection + Constraint Enforcement

**Impact: Medium | Feasibility: Medium | Prerequisite: Tier 1**

### Problem

jaxsr currently selects a subset first, then refits with constraints (penalty/scipy/CVXPY).
This two-phase approach can degrade the IC score — the selected subset may not be optimal
under the constraints.

### Solution

Physical constraints are already linearized in `_precompute_constraint_data`
(`constraints.py:1083`). Add them directly to the MIQP:

```
(same MIQP as Tier 1)
+ (Phi_plus - Phi_minus) @ beta >= 0   (monotonicity)
+ sign_i * beta_i >= 0                  (sign constraints)
+ A_lin @ beta <= b_lin                  (linear constraints)
+ beta_j = fixed_val                     (fixed constraints)
```

Still a MIQP (quadratic objective, all constraints linear in beta) — discopt handles
it natively.

### Implementation steps

1. Extend `miqp_subset_selection()` to accept an optional `constraints` list.
2. Translate each jaxsr `Constraint` type to `dm.subject_to()` calls using the same
   linearization already done in `_precompute_constraint_data`.
3. Add `constraint_enforcement="miqp"` option to `SymbolicRegressor` that performs
   selection and constraint enforcement in one shot, skipping the separate
   `_apply_constraints` step.

---

## Evaluated and deferred

These were analyzed but are not worth the dependency complexity at this time.

| Opportunity | Why deferred |
|---|---|
| **Robust symbolic regression** (discopt `RobustCounterpart`) | Changes fitting criterion away from OLS/IC; need already met by AICc, CV, bootstrap, conformal prediction. |
| **FIM-based DOE** (discopt `doe/`) | Solves parameter estimation DOE, not model discovery DOE. jaxsr's `ActiveLearner` already has `DOptimal`/`AOptimal` that compute FIM analytically for linear-in-parameters models. |
| **Rust backend for selection loops** | Bottleneck is combinatorial, not per-evaluation speed. MIQP (Tier 1) solves the real problem. |
