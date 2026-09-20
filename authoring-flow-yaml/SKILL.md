---
name: authoring-flow-yaml
title: Authoring Flow YAML
description: Use when writing, reading or hand-editing a vide-code flow file in .flows/, when asked where flows live or what a node or edge accepts, or when a flow fails to open, save or run and the YAML itself is suspect.
---

# Authoring flow YAML

## Overview

A vide-code flow is a graph of agents stored as one YAML file. Each node is an
agent with its own prompt; each edge says that one agent runs before another,
and optionally hands its answer over. For scheduling purposes the graph must
be acyclic — with one deliberate exception, the `retry` edge, which closes a
loop on purpose. See Retry below.

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
| `type` | yes | `data`, `order` or `retry`. Any other value orders the nodes but hands nothing over, which is almost never what you meant. |

## Connections

Three kinds. Picking the wrong one between `data` and `order` is the most
common modelling mistake in a flow; `retry` is a different animal entirely,
covered in its own section below.

| Type | Colour in the panel | Meaning |
|---|---|---|
| `data` | green | `to` runs after `from`, **and receives its answer**. |
| `order` | amber | `to` runs after `from`, and receives nothing. |
| `retry` | red | `to` (a node upstream of `from`) reruns when `from` rejects it. See Retry. |

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

## Retry: sending work back instead of fixing it

A `retry` edge points from a validator node back to a node it validates. It is
the exception to "no cycles": it is drawn in the direction opposite to normal
flow on purpose, closing a loop that the two other edge types would never be
allowed to close.

```yaml
edges:
  - from: escritor
    to: revisor
    type: data
  - from: revisor
    to: escritor
    type: retry
```

**Requires the forward edge first.** A `retry` edge `from: V, to: T` is only
accepted at save time if a `data` or `order` edge `from: T, to: V` already
exists. Draw the normal connection before the red one; the panel rejects the
red one otherwise (a flash that disappears, no error text). This is also why
`retry` is exempt from the "no cycles" check and from the "flow left to right"
positioning rule: it always runs backwards, by design.

**Having an outgoing `retry` edge makes a node a validator**, and changes how
it runs:

- It is given one extra tool, `flow_validate`, which it must call before its
  final response: `approved: true`, or `approved: false` with a concrete
  `feedback` string.
- **Approved:** nothing special happens, the flow continues as if the edge
  were not there.
- **Rejected:** every node the validator has a `retry` edge to goes back to
  `pending`, and so does the validator itself. The `feedback` is appended to
  the target's prompt as `## Feedback da validação` the next time it runs,
  accumulating across attempts if it is rejected more than once.
- **No attempt cap.** A validator that never approves loops forever; the user
  has to pause the run by hand. Do not promise a retry limit, there is none.
- **One verdict, not one per target.** A validator with `retry` edges to
  several nodes rejects **all of them together**, with the same feedback.
  There is no way to reject just one. If you need that, give each node its
  own validator instead of fanning several nodes into one.

Write the validator's prompt so it actually uses the contract: say what makes
the work rejectable, and that feedback should name what to fix, not just that
something is wrong. A prompt that only asks for an audit report, with no
mention of approving or rejecting, still gets the `flow_validate` tool forced
on it (the runner adds that instruction on top) but tends to always approve
or to reject with vague feedback, because the prompt never told it what
"reject" is for.

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
- no cycles among `data`/`order` edges (a `retry` edge does not count as a cycle)
- a `retry` edge needs a `data`/`order` edge already going the other way

**Running** checks all of the above, plus:

- every node has a non-empty `name`
- every node has a non-empty `prompt`
- the flow has at least one node

Two nodes may overlap on the canvas. That is layout, not a defect, and nothing
rejects it.

## How a flow executes

Rounds, not a single path:

1. Every `pending` node whose `data`/`order` parents are **all** `done` starts,
   **in parallel**. `retry` edges play no part in this: they never gate
   readiness, only react afterwards.
2. When a round finishes, the next round is computed the same way.
3. A node that fails pauses the whole run.
4. A node with an outgoing `retry` edge that rejects sends its target(s), and
   itself, back to `pending` right after it finishes — the same round loop
   then picks the target up again with the feedback attached, no user action
   needed.
5. Resuming re-runs failed nodes from scratch; nodes already `done` are not
   repeated.
6. If nothing is ready but something is still pending, the run pauses rather
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
| Downstream node placed left of its source | Edge loops backwards, unreadable | Order columns by dependency (does not apply to `retry`, which is meant to loop backwards) |
| `retry` edge drawn before the forward `data`/`order` edge exists | Save rejects it | Connect the two nodes forward first, then add `retry` |
| Expecting a validator to reject just one of several targets | It rejects every node it has a `retry` edge to, same feedback for all | Give each node its own validator if rejection must be selective |
| Validator prompt never mentions approving or rejecting | It still gets forced to call `flow_validate`, but tends to always approve or reject with vague feedback | Say explicitly what makes the work rejectable and what feedback should contain |

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
