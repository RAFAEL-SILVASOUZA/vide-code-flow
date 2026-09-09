---
name: authoring-flow-yaml
title: Authoring Flow YAML
description: Use when writing, reading or hand-editing a vide-code flow file in .flows/, when asked where flows live or what a node or edge accepts, or when a flow fails to open, save or run and the YAML itself is suspect.
---

# Authoring flow YAML

## Overview

A vide-code flow is a directed acyclic graph of agents stored as one YAML file.
Each node is an agent with its own prompt; each edge says that one agent runs
before another, and optionally hands its answer over.

**Core principle:** the file is the whole truth. The panel is a view of it, so
anything you can do by dragging cards you can do by editing the YAML, and the
same validation applies to both.

## Where flows live

```
<workspace>/.flows/<slug>.yaml
```

- The folder is `.flows` at the workspace root. No workspace open, no flows.
- The slug comes from the flow name: lowercase, accents stripped, every run of
  non-alphanumeric characters turned into a single `-`, leading and trailing
  dashes removed. `Documentação de API` becomes `documentacao-de-api.yaml`.
- Reading accepts `.yaml`, `.yml` and `.json`, in that order. `.json` is the
  pre-migration format: it still loads, and it is rewritten as `.yaml` on the
  next save. Writing always produces `.yaml`.
- The file name is independent of `name` after creation. Renaming the flow in
  the panel does not rename an existing file.

## The shape

```yaml
version: 2
name: Documentador
nodes:
  - id: n1
    name: Project Structure
    prompt: |-
      # Agent: Project Structure Collector
      Map the repository layout and report the entry points.
    x: 40
    y: 280
  - id: n2
    name: Planner
    prompt: |-
      # Agent: Documentation Planner
      Turn the structure into a documentation plan.
    x: 360
    y: 280
    instruction: um agente que planeja a documentacao a partir da estrutura
edges:
  - from: n1
    to: n2
    type: data
```

## Every property

### Top level

| Field | Required | Notes |
|---|---|---|
| `version` | yes | Write `2`. Overwritten with the current version on load, so it never blocks anything. Files at version 1 are migrated on read. |
| `name` | yes | Non-empty. A flow without a name is rejected on save. Seeds the file slug on first save. |
| `nodes` | yes | May be empty to save, never to run. |
| `edges` | yes | May be empty. A flow with no edges runs every node in one parallel round. |

### Node

| Field | Required | Notes |
|---|---|---|
| `id` | yes | Unique within the flow. Any string. The panel generates `n` + base36 timestamp + 4 random chars; hand-written ids like `planner` are equally valid and far easier to edit. |
| `name` | to run | **Functional, not cosmetic.** It becomes the heading of the handoff this node sends downstream. See Handoff below. |
| `prompt` | to run | The agent's system prompt. Use a YAML block scalar (`\|-`) for anything multi-line. |
| `x`, `y` | yes | Position in pixels. Multiples of 20. See Positioning. |
| `instruction` | no | The natural-language request that was given to the prompt generator. Purely a record of intent, never executed. Omit it rather than writing `instruction: ""`. |

A node missing `name` or `prompt` is a **draft**: it saves fine and shows on the
canvas with a dashed border, but the flow will not run until it is filled in.
This is deliberate, so you can lay out and wire a flow before writing prompts.

### Edge

| Field | Required | Notes |
|---|---|---|
| `from` | yes | Id of the upstream node. |
| `to` | yes | Id of the downstream node. |
| `type` | yes | `data` or `order`. Any other value orders the nodes but hands nothing over, which is almost never what you meant. |

## Connections

There are exactly two kinds, and picking the wrong one is the most common
modelling mistake in a flow.

| Type | Colour in the panel | Meaning |
|---|---|---|
| `data` | green | `to` runs after `from`, **and receives its answer**. |
| `order` | amber | `to` runs after `from`, and receives nothing. |

Use `order` when the downstream agent only needs the world to be in a certain
state (files written, migration applied), not the upstream text. Use `data`
when it needs to read what the previous agent produced.

### Handoff: what the downstream agent actually sees

At run time the effective prompt of a node is its own `prompt`, followed by one
section per `data` parent that produced output:

```
<the node's own prompt>

## Resultado de <parent name>
<everything that parent answered>

## Resultado de <other parent name>
<everything that parent answered>
```

Three consequences worth internalising:

1. **The parent's `name` is the heading.** Rename a node and you change the
   contract its children read. Name nodes after what they produce, not after
   what they are: `Estrutura do Projeto` beats `Agente 1`.
2. **Sections appear in flow node order**, not edge order.
3. **A parent that answers nothing contributes no section.** A child must not
   assume a section exists.

Write the child's prompt so it expects this. A prompt that says "read the
`## Resultado de Estrutura do Projeto` section below" works; one that says
"the user will give you the structure" does not.

## Positioning

Coordinates are pixels on an infinite canvas, snapped to a 20px dot grid.

| Constant | Value |
|---|---|
| Grid step | 20 |
| Card size | 220 wide, 120 tall |
| Auto-layout column | 260 |
| Auto-layout row | 160 |

Rules that keep a hand-written layout readable:

- **Always use multiples of 20.** Anything else is snapped on load, so `x: 47`
  silently becomes `40` and your file no longer matches what you see.
- **Negative coordinates clamp to 0.** There is no space left of or above the
  origin.
- **Flow left to right.** Edges leave the right edge of a card and enter the
  left edge of the next. Placing a target left of its source makes the edge
  loop backwards around both cards.
- **Space columns at least 260 apart** (220 card + 40 gap) and rows at least
  160 apart. Closer than that and cards touch.
- **Keep a linear chain on one `y`.** The ports sit at fixed heights, so
  same-`y` neighbours give the flattest line.
- **Fan out vertically.** One node feeding three others reads best with the
  three sharing an `x` and spaced 160 apart on `y`.

```
x:      40        360       680       1040
y: 120                      Backend
y: 280  Structure  Planner   Database   Revisor
y: 440                      Frontend
```

## What is rejected

Two different gates. Knowing which one you hit saves a lot of guessing.

**Saving** checks structure only:

- flow has a non-empty `name`
- every node has an `id`
- no duplicate ids
- every edge points at ids that exist
- no edge from a node to itself
- no cycles

**Running** checks all of the above, plus:

- every node has a non-empty `name`
- every node has a non-empty `prompt`
- the flow has at least one node

Two nodes may overlap on the canvas. That is layout, not a defect, and nothing
rejects it.

## How a flow executes

Rounds, not a single path:

1. Every `pending` node whose parents are **all** `done` starts, **in parallel**.
2. When a round finishes, the next round is computed the same way.
3. A node that fails pauses the whole run.
4. Resuming re-runs failed nodes from scratch; nodes already `done` are not
   repeated.
5. If nothing is ready but something is still pending, the run pauses rather
   than spinning.

So a flow with no edges is not sequential: it is one parallel round of every
node at once. If you need order, say so with an edge.

## Common mistakes

| Mistake | What happens | Fix |
|---|---|---|
| `order` where `data` was meant | Downstream agent runs with no input and invents one | Use `data` whenever the text matters |
| Coordinates not multiples of 20 | File and canvas disagree after the first load | Round to 20 |
| Renaming a node late | Children's prompts reference a heading that no longer exists | Rename node and children's prompts together |
| Expecting sequence from an edgeless flow | Everything runs at once | Add explicit edges |
| `instruction: ""` | Noise in a file meant to be hand-edited | Omit the key |
| Node id reused after deleting a node | Duplicate id rejects the save | Ids must be unique across the whole file |
| Downstream node placed left of its source | Edge loops backwards, unreadable | Order columns by dependency |

## Minimal valid flows

Smallest saveable flow (will not run, no prompt yet):

```yaml
version: 2
name: Rascunho
nodes:
  - id: a
    name: ''
    prompt: ''
    x: 0
    y: 0
edges: []
```

Smallest runnable flow:

```yaml
version: 2
name: Um agente
nodes:
  - id: a
    name: Escritor
    prompt: Escreva um resumo do repositorio.
    x: 40
    y: 40
edges: []
```
