---
layout: post
title: "leanlift: the fancy-update layer lands sorry-free"
---

[leanlift](https://github.com/zeta1999/leanlift) is an effort to mechanize Iris-style concurrent separation logic in Lean 4 — the program logic you reach for when you want to actually prove things about concurrent, resourceful code rather than argue about them. This week a milestone closed: **Phase A2, the fancy-update layer, is complete and sorry-free.**

## Why `fupd` is the hard part

The fancy update modality (`fupd`) is the joint in Iris where invariants get opened, used, and closed again. Almost every interesting proof rule passes through it, and it's exactly where the bookkeeping bites: you're manipulating *world satisfaction* — the global ledger of which invariants exist and who knows about them — while staying sound about ownership and about what holds "later".

Five pieces had to land, and all five are now proved with no `sorry`:

- **invariant-knowledge agreement + later-transport** — the base lemmas that say two parties' knowledge of an invariant agrees, and that you can move a proof under a `▷`.
- **`toMap_mem`** — an allocated invariant name really is in the world-satisfaction map.
- **`inv_acc`** — invariant *access*: open an invariant, use its contents, close it. Built on `ownI_open` / `ownI_close`.
- **`inv_alloc`** — invariant *allocation*: mint a fresh name and extend world satisfaction. This was the last one to fall, and it needed a from-scratch pigeonhole argument to show a fresh name always exists.

## Why do this in Lean

Iris lives in Coq. Redoing the core in Lean 4 isn't translation for its own sake — it's so the same proof machinery sits in the toolchain I already use for everything else, and so proof automation (the "lift" in leanlift) can be built on a foundation I control. `inv_alloc` being the holdout is fitting: allocation is where you can't borrow an existing fact and have to build the argument from nothing.

The weakest-precondition calculus is the layer that sits on top of fancy updates, so that's the direction this is headed. Either way the plumbing underneath is the part you only build once — and now it's done, machine-checked.
