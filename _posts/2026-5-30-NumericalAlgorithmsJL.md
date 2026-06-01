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

## Why this is load-bearing in LLM serving, not just a textbook lemma

If you only ever hear "Johnson–Lindenstrauss" in a proofs class it sounds like a curiosity: you can jam `n` points from a huge dimension `d` down to `m ≈ log n / ε²` dimensions with a random matrix and barely move any pairwise distance. Fine. Why would anyone shipping LLMs care?

Because an LLM is not made of words — it's made of vectors. Every token is an embedding in `ℝ^d` (think `d` = thousands), and attention is *literally* a pile of inner products between those vectors. The model is geometry. Which means the two things that actually blow up your memory budget in production are both "store a lot of high-dimensional vectors and keep their inner products intact":

- **The KV-cache.** During generation a transformer keeps the key/value vector of every past token so it doesn't recompute them. That cache grows with context length × attention heads × concurrent requests — it, not the weights, is the memory wall in LLM serving. People who say "just use a bigger context window" usually have no idea this is what they're spending.
- **Vector databases / RAG.** Retrieval stuffs millions of document embeddings into an index and answers a query by nearest-inner-product search. Same problem: too many vectors, too many bits each.

The obvious move is to store each coordinate in fewer bits. The obvious risk is that naive rounding wrecks exactly the inner products attention and retrieval depend on. "Compress these vectors with a cheap, randomized map that *provably* preserves inner products up to a bounded error" is not a hack you reach for — it's the JL lemma, restated as an engineering requirement.

That's precisely what the current state-of-the-art does. **[TurboQuant](https://arxiv.org/abs/2504.19874)** (Zandieh et al., Google Research, 2026) quantizes a vector in three JL-flavored steps: randomly **rotate** it with an orthogonal transform (a JL-style random map) so its coordinates land in a stable, well-behaved distribution; apply an MSE-optimal scalar quantizer; then encode the leftover residual with a **1-bit Quantized Johnson–Lindenstrauss (QJL)** sketch. JL is the part that guarantees the rotation and the 1-bit sketch keep inner products *unbiased* — that's the whole reason the scheme works at 1–4 bits per coordinate while staying within ≈2.7× of the Shannon lower bound on distortion. It's data-oblivious and online, which is why it drops straight into KV-cache compression and vector indexes.

**TurboVec** — a Rust vector index with Python bindings, built directly on TurboQuant for local RAG — is the same lemma cashed out as a product: a 31 GB float32 corpus of ~10M embeddings fits in about 4 GB, and you still search it on inner products. It inherits its accuracy guarantee from JL through TurboQuant; there is nothing else holding the error down.

So the chain is short and worth stating plainly for anyone who throws "LLM" around without knowing what a DNN actually computes: **JL lemma → random rotation + 1-bit QJL → TurboQuant → TurboVec / KV-cache / RAG.** The probabilistic concentration bound formalized above — `m` large enough and the chance of *any* distorted inner product is below `δ` — is the exact promise these systems bet their output quality on, usually on faith and a benchmark table. The point of doing it with no `sorry` is that "the random projection preserves the geometry" stops being a vibe and becomes a kernel-checked theorem.

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
