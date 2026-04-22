---
layout: post
title: "numerical-algorithms: LU and QR, dual-stacked in Lean 4 and F* (WIP)"
---

[numerical-algorithms](https://github.com/zeta1999/numerical-algorithms) is a small, explicitly-WIP experiment: take classical linear-algebra algorithms and verify them in two proof ecosystems side-by-side, instead of picking one.

## Scope right now

Two algorithms:

- `nr1` — **LU** with Doolittle + partial pivoting.
- `nr2` — **QR** via Modified Gram-Schmidt.

Each one is implemented **twice**:

- A Lean 4 library with both an exact-ℚ path and an IEEE-754 Float path, side-by-side. Exact ℚ is the ground truth; Float is the path you'd actually run.
- An F* library over fraction-free integer arithmetic (no rounding, no real-number axioms). Z3 discharges the VCs.

Plus a linear solver per algorithm, sensitivity analysis, unit tests (14 per algo: 5 exact-ℚ + 9 float), and a fuzz harness (3500 random matrices of size 2–20, plus Hilbert matrices 2–8).

## Proof state, honestly

This is the part I want to be careful about, because "formally verified" is a phrase the internet stretches past breaking.

What's **machine-proved** in Lean 4:

- `L` is lower triangular with unit diagonal (LU).
- `R` is upper triangular with unit diagonal (QR).

What's **machine-proved** in F*:

- Identity-matrix properties, `iabs` properties, zero vector/matrix lemmas. Structural, not end-to-end.

What's **axiomatized and checked only by fuzzing + sensitivity analysis**:

- `U` upper triangular (LU).
- `PA = LU` (the core LU correctness statement).
- `A = QR` (the core QR correctness statement).
- `Q` columns orthogonal.
- `solve` correctness (both decompositions).
- `det` via LU.

What's **admitted** in F* and waiting on a refactor:

- LU: `find_pivot` bounds, `swap_rows` involution.
- QR: `dot_product` self non-negativity, `dot_product` commutativity. Blocked by F*'s closure-reference limitation; the fix is to lift the inline `let rec`s to top-level.

So the headline is: the triangularity of the output factors is proved; the correctness of the decomposition itself is still an axiom backed by 3500 random matrices and pathological Hilbert cases, not a theorem. That's where the work actually is.

## Why two stacks

Lean 4 and F* answer different questions:

- Lean's strength here is putting exact ℚ and IEEE Float next to each other in the same library. You can prove something over ℚ and then state the Float version as an approximation, and the gap is the thing you measure rather than the thing you hide.
- F* over fraction-free integers removes rounding from the picture entirely — but the integer MGS is unnormalized, so the "R" you get is scaled. You give up a property (normalization) to get a cleaner proof.

An axiom in one stack is often a theorem in the other's model. The interest is less each individual proof and more the shared surface of lemmas.

## Sensitivity

The LU run at n=50 looks like textbook backward stability — forward error tracks κ(A) · ε_machine, backward error pinned at machine precision across six orders of κ:

| Target κ | Actual κ | Max forward err | Backward err |
|---:|---:|---:|---:|
| 1     | 1.00    | 0      | 0 |
| 1e4   | 4.2e4   | 0      | 0 |
| 1e8   | 3.5e8   | 0      | 0 |
| 1e10  | 3.2e10  | ~0     | 0 |
| 1e12  | 3.2e12  | 2.5e-5 | 0 |

Full n × κ tables in `nr1/SENSITIVITY_ANALYSIS.md` and `nr2/SENSITIVITY_ANALYSIS.md`. This isn't a proof — it's the empirical leg that's currently holding up the axiomatised correctness statements.

## What I'd do next, in rough order

1. Promote `PA = LU` from axiom to Lean 4 theorem. This is the one that actually closes the loop; everything else is easier once the recursive step is handled.
2. Lift the F* `let rec` admits (`find_pivot_*`, `dot_product_*`) to top-level recursion so Z3 can see them.
3. Householder or Givens QR for ill-conditioned matrices — MGS loses orthogonality there, which is the one place the sensitivity CSV would be embarrassing if κ went higher.
4. Cholesky next (`nr3`), then SVD. Cholesky fits the existing shape; SVD is where the proof cost goes up noticeably.

Code @ [github.com/zeta1999/numerical-algorithms](https://github.com/zeta1999/numerical-algorithms).
