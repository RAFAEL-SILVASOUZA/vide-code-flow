---
name: adjusting-a-flow
title: Adjusting a Flow
description: Use when changing a vide-code flow that has already run, when adding, removing or renaming nodes or edges in an existing flow, or when the user is unhappy with what a flow produced and wants it different.
---

# Adjusting a flow

## Overview

Editing a flow that already ran is not the same as writing a new one. Files
from the last run are still in the project, and the change you are about to
make decides whether they stay correct, go stale, or have to be regenerated.

**REQUIRED BACKGROUND:** `authoring-flow-yaml` for the schema,
`rerunning-a-flow` for the gate before executing anything.

**Core principle:** the edit and the artifacts are one decision, not two.
Deciding what to change without deciding what happens to what already exists
leaves the project half in one world and half in the other.

## Before proposing the edit

Read `<workspace>/.flows/.runs/<slug>.yaml`. If it is missing, this is just a
flow edit: use `creating-a-flow` and stop reading here.

If it is there, the flow produced things. Work out which of the recorded
artifacts the change invalidates, then run the conversation below.

Also check for `<workspace>/.flows/.runs/<slug>.checkpoint.json`. If it
exists, the run is **unfinished**, and the edit lands on a run that will
resume, not restart. See Editing an unfinished run below.

## The conversation

Ask about the change and the consequence together. One message.

1. **What is wrong with what came out?** The answer usually points at one node,
   which is a smaller edit than rebuilding the flow.
2. **Which of these files should survive the change?** Name them from the
   manifest. Some are hand-edited since, some are disposable.
3. **Should the flow rerun after the edit, or only the affected part?** There
   is no partial rerun of a *completed* run: pressing play again redoes every
   node. If only one node's output is wrong, deleting its artifact and
   rerunning still redoes all of it. An *unfinished* run is the opposite: it
   resumes and skips every `done` node, so it may redo less than the edit
   needs. (A `retry` edge is different again: it partially reruns a node
   automatically, but only reactively, inside a run that is still going. It
   is not a substitute for this gate.)

Do not ask what they want the flow to do from scratch. They already have a
flow; the question is what to change about it.

## What each kind of edit does to existing artifacts

| Edit | Effect on what was produced |
|---|---|
| Change a node's `prompt` | Its artifacts are now produced by different instructions. Stale until rerun |
| **Rename a node** | Breaks every child that reads `## Resultado de <old name>`. Update those prompts in the same edit |
| Add a node upstream | Downstream nodes gain a section they were not written to expect. Their prompts need updating too |
| Add a node downstream | Existing artifacts stay valid. The cheapest kind of change |
| Delete a node | Its artifacts are orphaned: nothing regenerates them, and nothing removes them either. Decide explicitly |
| Change `data` to `order` | The downstream agent loses an input it was written to read. Its prompt must change too |
| Change `order` to `data` | The downstream agent gains a section it ignores until its prompt mentions it |
| Add a `retry` edge from an existing node to one of its `data`/`order` parents | No effect on past artifacts, but changes future runs: that parent can now be sent back to redo. Needs the forward edge already in place, and the new validator's prompt updated to actually reject when appropriate |
| Toggle `reasoning` on a node | Nothing already produced changes. Future runs of that node think more or less |
| Move a card (`x`, `y`) | Nothing. Layout only |
| Rename the flow | The file keeps its old slug, and so does the manifest. An unfinished run's checkpoint stops matching and is ignored: the next play starts over instead of resuming |

The node rename row is the one that bites. A node's `name` is the heading of
the handoff it sends, so renaming silently breaks a contract that no
validation checks.

## Editing an unfinished run

When a checkpoint exists, the next play resumes, and resuming applies the edit
only partially:

| Edit | What the resume does with it |
|---|---|
| Change the prompt of a `done` node | Ignored. The node is skipped and its old answer keeps feeding its children |
| Change the prompt of a node interrupted mid-work | Ignored. It continues its own saved conversation, with the task it started with |
| Change the prompt of a `pending` or failed node | Applied. It runs from scratch with the new prompt |
| Add a node | Runs, once its parents are `done` |
| Delete a node | Dropped from the run |

So if the edit touches a `done` or interrupted node, the honest options are
**start over** (`restart: true`, or Start over in the panel's modal), or
accept that the old output stays. Say which, and let the user pick. Starting
over does not remove the files the unfinished run already wrote.

## Then

State the plan as: the edit, the artifacts that go stale, and whether a rerun
follows. Apply the edit. If a rerun follows, `rerunning-a-flow` governs it -
the manifest gate still applies, even though it was you who just changed the
flow.

## Common mistakes

| Mistake | Fix |
|---|---|
| Editing the YAML and stopping there | Say what happens to the files the flow already wrote |
| Renaming a node without touching its children | Grep the other prompts for the old name in the same edit |
| Rebuilding the whole flow for one bad node | Change that node's prompt |
| Promising a partial rerun of a completed run | Pressing play again redoes every node. Only a live `retry` edge reruns partially, and only reactively |
| Editing a `done` node of an unfinished run and resuming | The resume skips it, and the edit has no effect. Start over, or say the old output stays |
| Deleting a node and leaving its output | Ask whether the file should be removed too |
| Running straight after the edit | The rerun gate applies to your own edits as well |
