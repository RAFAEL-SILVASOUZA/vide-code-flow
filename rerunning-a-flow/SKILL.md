---
name: rerunning-a-flow
title: Rerunning a Flow
description: Use before running or resuming any vide-code flow that may have run before, when the user asks to run a flow again, when a flow was interrupted or VS Code closed mid-run, or when a flow's outputs already exist in the project.
---

# Rerunning a flow

## Overview

Starting a flow that ran before does one of two very different things, and
nothing in the request "run the flow" says which:

- **Resume.** An unfinished run left a checkpoint. Nodes already `done` are
  skipped, and the rest pick up where they stopped.
- **Start over.** Every node runs again from scratch and every write happens
  again, over whatever the last run produced.

**REQUIRED BACKGROUND:** `authoring-flow-yaml` has the schema and the execution
model this skill depends on.

**Core principle:** a flow that already left artifacts is a destructive action.
Read what the last run left, say which of the two will happen and what it
overwrites, get an explicit decision. In that order, every time.

## Why this is not paranoia

Four facts about the runner, none of them obvious from the request:

1. **Flow nodes get the full tool set**, `createFile`, `applyPatch` and
   `runShell` included. A flow writes real files, and can delete them through
   the shell.
2. **Only an unfinished run resumes.** A run that paused (by hand, because a
   node failed, or because VS Code closed) keeps a checkpoint on disk, and it
   survives restarts. A run that **completed** deletes its checkpoint:
   starting it again redoes every node.
3. **Resuming is the silent default.** The `start_flow` tool resumes an
   interrupted run unless `restart: true` is passed, and reports that it did
   so in `resumedFrom`. The panel asks with a Continue / Start over modal, but
   the tool does not ask anyone.
4. **Starting over does not undo anything.** Discarding the checkpoint throws
   away the progress, not the files. Everything the interrupted run wrote
   stays in the project, and the new run writes over it.

## The gate

**Do not start or resume a flow before completing these three steps.**

### 1. Read what the last run left

Two files, same slug as the flow:

```
<workspace>/.flows/.runs/<slug>.yaml             manifest
<workspace>/.flows/.runs/<slug>.checkpoint.json  checkpoint
```

| Manifest | Checkpoint | Meaning | Starting the flow will... |
|---|---|---|---|
| absent | absent | Never ran, or ran before manifests existed | Run fresh, nothing to overwrite |
| `outcome: completed` | absent | Finished | Redo every node from scratch |
| `outcome: paused` | present | Stopped halfway, files left behind | Resume by default, or start over if asked |
| `outcome: paused` | absent | The checkpoint was discarded, or the flow was renamed since | Redo every node from scratch |

The manifest says what each node touched:

```yaml
version: 1
flow: Documentador
startedAt: 2026-09-09T13:40:00.000Z
finishedAt: 2026-09-09T13:47:12.000Z
outcome: paused
nodes:
  - id: estrutura
    name: Estrutura do Projeto
    status: done
    artifacts:
      - .videCode/docs/overview.md
  - id: revisor
    name: Revisor
    status: failed
    artifacts: []
    error: o modelo excedeu o limite de contexto
```

The checkpoint is JSON the runner writes after every node. You only need
`flow`, `updatedAt` and each entry's `status` from it, to say how far the run
got. Never edit it by hand.

```json
{"version":1,"runId":"…","flow":"Documentador","startedAt":"…","updatedAt":"2026-09-09T13:47:12.000Z",
 "nodes":{"estrutura":{"status":"done","output":"…","artifacts":[".videCode/docs/overview.md"]},
          "revisor":{"status":"failed","error":"…"}}}
```

A checkpoint whose `flow` differs from the flow's current `name` is ignored:
renaming a flow loses its interrupted run.

### 2. Confirm the artifacts still exist

The manifest records what was written, not what is there now. A file the user
already deleted or rewrote by hand changes the decision. Check each path.

### 3. State the consequence and ask

Name the files, say which nodes run and which files they may overwrite, and
wait for an answer. Not "should I proceed?" in the abstract, but the specific
list, and when a checkpoint exists, the choice between resuming and starting
over.

> A execução anterior parou em 09/09 13:47, com 1 de 2 agentes concluídos
> (Revisor falhou: o modelo excedeu o limite de contexto). Ela escreveu
> `.videCode/docs/overview.md`, que ainda existe.
>
> - **Continuar:** só o Revisor roda de novo, do zero. O overview fica como
>   está.
> - **Recomeçar:** os dois agentes rodam de novo e o overview é
>   sobrescrito.
>
> Qual prefere?

Then call `start_flow` with `restart: true` only if they chose to start over.

## What resuming actually does

Worth knowing before promising anything:

- **`done` nodes are not rerun**, and their saved answers still feed their
  children. That holds even if their prompt was edited since, so an edited
  `done` node keeps its old output until a restart.
- **Failed nodes rerun from scratch.**
- **A node interrupted mid-work continues its own conversation**, told that
  files may have changed since. It keeps the task it was started with, so a
  prompt edited during the pause does not reach it.
- **Nodes added since the pause run; nodes removed are dropped.**
- **Retry feedback and attempt counts carry over.**

## When the gate does not apply

Skip it, do not perform it half-heartedly:

- No manifest and no checkpoint, or neither lists artifacts. Nothing to
  overwrite.
- The user already named the artifacts in this conversation and chose
  between resuming and starting over. Do not re-ask; that is friction, not
  safety.
- The run is **resuming a paused flow the user just paused themselves**, in the
  same conversation. They know the state. The panel does not ask in that case
  either: it resumes the live run directly.

## Red flags - STOP and run the gate

- "The user said run, so I run"
- "It is their flow, they know what it does"
- "The manifest is probably stale anyway"
- "I will mention the overwrite afterwards"
- "It only writes documentation, that is harmless"
- "Asking breaks the flow of vibe coding"
- "It will just resume, nothing gets redone"
- About to call `start_flow` without having opened `.flows/.runs/`

**Every one of these means: read the manifest and the checkpoint first.**

## Rationalizations

| Excuse | Reality |
|---|---|
| "Running again is idempotent" | It is not. Every node that runs rewrites, with a fresh model call each time |
| "It will resume where it stopped" | Only an unfinished run resumes. A completed one starts over |
| "It will start fresh, like I asked" | An interrupted run resumes unless `restart: true` is passed |
| "Starting over cleans up the half-written state" | It discards the progress, not the files. They stay and get overwritten |
| "Resuming picks up my prompt edits" | `done` nodes keep their old output; an interrupted node keeps its old task |
| "A failed run left nothing behind" | The nodes before the failure completed and wrote |
| "The user asked, that is authorisation enough" | They asked to run, not to overwrite files they may have forgotten about |
| "Checking is slower than just running" | Reading two files costs a second. Regenerating a document the user hand-edited costs their afternoon |
| "The panel would have warned" | The panel only asks when a checkpoint exists, and shows no file list. The tool never asks |

## After the run

The manifest is rewritten with the new run, replacing the old one, and the
checkpoint is deleted once the run completes. If the user needs the previous
state, that is version control, not the manifest: say so **before** running,
not after.
