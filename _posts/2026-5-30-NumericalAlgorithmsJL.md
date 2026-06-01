---
layout: post
title: "numerical-algorithms: a no-sorry Johnson–Lindenstrauss proof in Lean 4, randomized SVD in progress (updated)"
---

Follow-on to the [LU + QR post]({% post_url 2026-4-8-NumericalAlgorithmsWIP %}). That one was about classical linear algebra dual-stacked in Lean 4 and F*. Since then two things landed: a **complete** machine-checked proof of the Johnson–Lindenstrauss lemma, and the start of a randomized SVD. Worth being precise about which is done and which isn't.

## Johnson–Lindenstrauss, end to end

This is the one I'm happy about. `johnson-lindenstrauss/` is a Lean 4 development that compiles with **no `sorry`** and rests only on the three standard foundational axioms — you can check that claim yourself:

```
> #print axioms JL.johnsonLindenstrauss
'JL.johnsonLindenstrauss' depends on axioms: [propext, Classical.choice, Quot.sound]
```

Nothing project-specific is axiomatized. The formal statement (`JL/UnionBound.lean`):

```lean
theorem johnsonLindenstrauss (n d : ℕ) (m : ℕ) (m_pos : 0 < m)
    (x : Fin n → EuclideanSpace ℝ (Fin d)) (hx : Function.Injective x)
    (ε δ : ℝ) (hε_pos : 0 < ε) (hε_lt : ε < 1) (hδ_pos : 0 < δ)
    (hsuff : 2 * (n : ℝ) ^ 2 * rexp (-(m : ℝ) * ε ^ 2 / 12) ≤ δ) :
    (rowLaw m d).real
      {G | ∃ i j, i ≠ j ∧
        ((1 + ε) * ‖x i - x j‖ ^ 2 ≤ (1 / (m : ℝ)) * ∑ k, ⟪x i - x j, G k⟫_ℝ ^ 2
         ∨ (1 / (m : ℝ)) * ∑ k, ⟪x i - x j, G k⟫_ℝ ^ 2 ≤ (1 - ε) * ‖x i - x j‖ ^ 2)}
      ≤ δ
```

In words: project `n` points from `ℝ^d` to `ℝ^m` with a Gaussian random matrix, and the probability that *any* pairwise squared distance is distorted by more than `ε` is at most `δ` once `m` is large enough. Built on mathlib, split into eight modules following the textbook chain:

| Module | Section |
|---|---|
| `JL/LogBounds.lean`   | §1 elementary `log` inequalities |
| `JL/GaussianMGF.lean` | §2 `E[exp(t Z²)] = (1−2t)^{−1/2}` for `Z ∼ N(0,1)` |
| `JL/Basic.lean` + `JL/ChiSq.lean` | §3 the chi-squared law and its MGF `(1−2t)^{−m/2}` |
| `JL/TailBounds.lean`  | §4 Chernoff tail bounds for `χ²ₘ` |
| `JL/Projection.lean`  | §5 single-vector concentration |
| `JL/UnionBound.lean`  | §6 union bound ⇒ the JL lemma |

`WALKTHROUGH.md` explains each section in prose next to the Lean tactics, and `article.tex` is the same as a typeset PDF. mathlib is vendored as prebuilt `.olean`s, so `lake build JL` only compiles the eight files — no mathlib rebuild.

## Randomized SVD — started, not proved

`randomized-svd/` is the opposite end of the maturity scale: a Rust implementation built from scratch, correctness first, against "Pass-efficient randomized SVD with boosted accuracy." It uses a PerSVD scheme with shifted power iteration and an economy inner SVD via `svd_from_eig`, with QR by modified Gram-Schmidt + re-orthogonalization. The plan (in `specs.txt`) is to compare f32 / f64 / 2×f64 (quad) precision with Kahan summation and look at high-accuracy inner SVD (twisted factorization, the Fernando approach).

The Lean side here is **not** the JL story — `RandomizedSVD.lean`, `Eigen.lean`, `QR.lean`, `AccuracyBounds.lean` all still carry `sorry`. It's a formalization skeleton, honestly tagged as such.

## Where LU and QR actually stand

A correction to any impression the earlier post left that these are "verified." Per `STATUS.md`, what's *proved* in Lean 4 is narrower than what's *checked*:

- **Proved (Lean 4):** L lower-triangular + unit diagonal (LU); R upper-triangular (QR).
- **Axiomatized but empirically validated:** `PA = LU`, `A = QR`, `Q` columns orthogonal, `solve` correctness, `det` via LU — each backed by 14 unit tests + 3500 fuzz tests (sizes 2–20 plus Hilbert matrices).
- **Admitted in F\*:** a handful of `let rec` closure and sequence-equality lemmas that F* can't currently reach without refactoring.

Discharging the `PA = LU` / `A = QR` axioms means induction over the full recursive decomposition — ~100s of lines each, still open. So: the structural lemmas are proved, the end-to-end correctness theorems are axioms-with-tests. JL is the only thing here that's complete.

## An honest note on the harness

These proofs and the Rust were written with an AI coding harness — Claude Code, plus a local **Qwen3.6** running on my **RTX PRO 6000** for the cheap, high-volume iteration (grinding mathlib lemma names, tactic permutations, test scaffolding), escalating to a frontier model only for the hard structural steps. The commit log isn't shy about it.

The reason I'll state that plainly: **it changes nothing about the guarantee.** A Lean proof is checked by Lean's kernel, not by its author. `#print axioms JL.johnsonLindenstrauss` reports exactly what the theorem rests on, and a green `lake build JL` with no `sorry` means the same thing whether a human, an LLM, or a coin flip produced the tactic script. Same for F*: Z3 discharges the obligations and the admits are listed explicitly. The machine-checker *is* the trust mechanism — which is the whole point of doing this in a proof assistant rather than writing "verified" in a README.

Where the harness was genuinely weak: global architecture. The model is good at local proof search and bad at deciding how to *decompose* a formalization — the "split into modules" commit was human structuring, after which the LLM filled sections in. And it's no help at all on the axiomatized correctness theorems above, which is partly why they're still open.
