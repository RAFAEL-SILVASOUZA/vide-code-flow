---
name: reviewing-flow-integrity
title: Reviewing Flow Integrity
description: Use when a vide-code flow refuses to save or run, pauses mid-execution, produces empty or contradictory handoffs, or when asked to audit a flow before trusting it with real work.
---

# Reviewing flow integrity

## Overview

A flow can be perfectly valid and still be useless. The tool rejects broken
graphs; it says nothing about a pipeline whose agents talk past each other.

**REQUIRED BACKGROUND:** `authoring-flow-yaml` has the schema and the two
validation gates this skill checks against.

**Core principle:** report both tiers. A review that only runs the structural
checks tells the user what the tool would have told them anyway.

## Two tiers, always both

### Tier 1: what the tool rejects

Blocking. The flow will not save or will not run.

**Save is blocked by:**

- [ ] `name` missing or empty
- [ ] a node without an `id`
- [ ] two nodes sharing an `id`
- [ ] an edge whose `from` or `to` is not an existing node id
- [ ] an edge from a node to itself
- [ ] a cycle anywhere in the graph

**Run is additionally blocked by:**

- [ ] a node with an empty `name`
- [ ] a node with an empty `prompt`
- [ ] a flow with no nodes

A node missing `name` or `prompt` is a legal draft, so this is a warning while
the user is still building and a blocker the moment they press run. Say which.

### Tier 2: what the tool accepts and shouldn't

Non-blocking, and this is where unproductive pipelines actually live.

- [ ] **`order` where `data` was meant.** The single most common defect. For
      every `order` edge, ask whether the downstream prompt refers to anything
      the upstream agent produced. If it does, the type is wrong and the agent
      will invent its input.
- [ ] **`data` edge nobody reads.** The child's prompt never mentions the
      parent's section. The text is appended and ignored, burning context.
- [ ] **Handoff heading mismatch.** A child references
      `## Resultado de <name>` with a name no parent actually has, usually
      because a node was renamed. Compare against the parents' current `name`.
- [ ] **Orphan node.** No incoming and no outgoing edge. It runs in the first
      round and its output goes nowhere. Either wire it or drop it.
- [ ] **Unreachable deliverable.** More than one node has no outgoing edge. A
      flow normally converges; several terminals usually means someone forgot
      an edge.
- [ ] **Overlapping scope.** Two nodes instructed to produce the same artifact.
      Downstream gets contradictions.
- [ ] **Serialised independence.** An edge between nodes that share no data and
      no state dependency. It costs a whole round for nothing.
- [ ] **Positions off the grid.** Any `x` or `y` that is not a multiple of 20,
      or is negative. The file and the canvas will disagree after the next load.
- [ ] **Cards on top of each other.** Two nodes closer than 220 horizontally
      and 120 vertically overlap on screen. Legal, unreadable.
- [ ] **Backwards edge.** `to.x` is less than `from.x`. The connection loops
      around both cards.
- [ ] **`instruction: ""`.** Noise in a file meant to be hand-edited.

## How to check the graph properties

Cycles, orphans and terminals are easy to get wrong by eye on anything past
five nodes. Reduce it to two lists first:

```
parents[n]  = every edge with to == n
children[n] = every edge with from == n

orphan      : parents[n] empty AND children[n] empty
terminal    : children[n] empty
source      : parents[n] empty
```

Then read the flow as rounds, which is how it actually executes:

```
round 1 = every source
round k = every node whose parents are all in rounds < k
```

If some node never lands in a round, the graph has a cycle. If a round is wide,
that is parallelism and it is free; if every round has one node, check whether
the chain is real.

## Reporting

Order findings by whether they stop the flow, not by where they appear in the
file. For each finding give the node or edge, what happens at run time, and the
fix. If tier 1 is clean, say so explicitly before moving to tier 2, so the user
knows the flow at least runs.

State plainly when a flow is fine. An audit that manufactures findings to look
thorough is worse than no audit.

## Worked example

```yaml
version: 2
name: Revisao de PR
nodes:
  - id: diff
    name: Coletor de Diff
    prompt: Colete o diff do branch atual e liste os arquivos alterados.
    x: 40
    y: 160
  - id: testes
    name: Rodador de Testes
    prompt: Rode a suite de testes e relate as falhas.
    x: 360
    y: 320
  - id: revisor
    name: Revisor
    prompt: |-
      Revise as mudancas usando a secao "## Resultado de Coletor de Diff".
      Aponte defeitos de correcao.
    x: 360
    y: 0
edges:
  - from: diff
    to: revisor
    type: data
  - from: diff
    to: testes
    type: order
```

Tier 1: clean. It saves and runs.

Tier 2, three findings:

1. **`testes` is a terminal that nobody reads.** The suite runs and the report
   dies there. If the review should account for failing tests, add
   `testes -> revisor` as `data` and tell `revisor` to read that section.
2. **`diff -> testes` is `order` and probably should not exist at all.**
   Running the test suite does not need the diff text or any state the collector
   leaves behind. Dropping the edge puts both nodes in round 1, in parallel, and
   the flow gets faster for free.
3. **Two terminals (`testes` and `revisor`).** Symptom of finding 1, not a
   separate defect. Worth naming so the user sees the shape.

Positions are all multiples of 20, no card overlaps, and no edge runs backwards.

## Common mistakes when reviewing

| Mistake | Fix |
|---|---|
| Stopping after tier 1 | The tool already does tier 1. The value is tier 2 |
| Calling a draft node an error | It is legal until the user presses run. Say which gate it hits |
| Flagging overlapping cards as invalid | It is a layout nit, never a blocker |
| Assuming a chain is intentional | Ask whether the dependency is real; parallel rounds are free |
| Reporting findings in file order | Blockers first, always |
| Inventing findings on a healthy flow | Say it is healthy |
