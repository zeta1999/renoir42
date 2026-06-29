---
layout: post
title: "Le Harnais: a world model in the loop, then distilled to run local"
---

"Le Harnais" is a harness for building tools with local models — the agent loop, the CLI (`lh`), and the plumbing that lets a small model do useful work. Two pieces landed recently that are worth writing down, because they go together.

## A world model in the ReAct loop

The standard ReAct loop is: think, act, observe, repeat. The agent commits to an action and only finds out what it did after the environment answers. **AgentWorld** is a learned world model that lets the agent look *before* it leaps: simulate the likely outcome of a candidate action, and use that to decide whether to take it, escalate, or try something else.

It came up in stages. Phase 1 was a live world-model simulator end-to-end — `lh harness sim` — a model of the environment you can step by hand. Phase 2 wired that simulator into the ReAct loop as **lookahead**: at each step the agent can roll the world model forward instead of acting blind.

The obvious question is whether lookahead actually helps or just burns tokens. So there's an A/B harness — `lh harness lookahead-ab` — that measures lookahead's effect on escalation and on step count directly, instead of leaving it to intuition. Whether it earns its token cost is a measured question, and the honest answer is "it depends on the task" — which is the whole reason the A/B switch exists.

## Distilling the teacher down to a local student

A 35B world model is not what you want running on a laptop next to the agent. Phase 3 is distillation: take **AgentWorld-35B** as the teacher and train smaller students against it — a **1B** student first, then **3B** and **8B** for the scaling picture. Each is benchmarked against the teacher so the trade-off is explicit rather than assumed.

The point of the whole exercise is local-first: a harness where the loop, the lookahead, and increasingly the model itself can run on hardware you control. That's the thread connecting Le Harnais to the rest of what I build — [leanlift](https://github.com/zeta1999/leanlift) for proof automation, the security toolkit for review, and the [simple tools](https://github.com/zeta1999/simple-tools) libraries for everything the agent reaches for.

Public release is coming. The design bet is local-first; whether the world model and its lookahead are worth their cost is left to the measurements rather than asserted here.
