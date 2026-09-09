---
name: rerunning-a-flow
title: Rerunning a Flow
description: Use before running or resuming any vide-code flow that may have run before, when the user asks to run a flow again, or when a flow's outputs already exist in the project.
---

# Rerunning a flow

## Overview

Running a flow a second time is not idempotent and is not a resume. Every node
runs again from scratch and every write happens again, over whatever the last
run produced.

**REQUIRED BACKGROUND:** `authoring-flow-yaml` has the schema and the execution
model this skill depends on.

**Core principle:** a flow that already left artifacts is a destructive action.
Check the manifest, show what gets redone, get an explicit decision. In that
order, every time.

## Why this is not paranoia

Three facts about the runner, none of them visible in the panel:

1. **Flow nodes get the full tool set**, `createFile`, `applyPatch`,
   `deleteFile` and `runShell` included. A flow writes real files.
2. **A finished run is not resumable.** Only a *paused* run resumes. Pressing
   play on a `completed` flow starts fresh: every node back to `pending`, every
   previous output discarded.
3. **The previous outputs are gone from memory.** Nothing in the running
   process remembers what the last run said. Only the manifest survives.

So the second run does not skip the work already done, does not diff against
it, and does not warn. It just does it again.

## The gate

**Do not start or resume a flow before completing these three steps.**

### 1. Read the manifest

```
<workspace>/.flows/.runs/<slug>.yaml
```

Same slug as the flow file. Absent means the flow never ran, or ran before
manifests existed. Present means it ran, and `artifacts` says what it touched.

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

Only the last run is kept. `outcome: paused` with a `failed` node means the run
stopped halfway and **still left files behind** - the most dangerous case,
because the project is in a half-written state that looks unfinished.

### 2. Confirm the artifacts still exist

The manifest records what was written, not what is there now. A file the user
already deleted or rewrote by hand changes the decision. Check each path.

### 3. State the consequence and ask

Name the files, say they get rewritten, and wait for an answer. Not "should I
proceed?" in the abstract: the specific list.

> A execução anterior terminou em 09/09 13:47, pausada no nó Revisor. Ela
> escreveu `.videCode/docs/overview.md` e `.videCode/docs/api.md`, e os dois
> ainda existem. Re-executar refaz todos os nós, inclusive os que já estavam
> concluídos, e sobrescreve esses dois arquivos. Confirma?

## When the gate does not apply

Skip it, do not perform it half-heartedly:

- No manifest, or manifest with no artifacts anywhere. Nothing to overwrite.
- The user already named the artifacts in this conversation and said to
  proceed. Do not re-ask; that is friction, not safety.
- The run is **resuming a paused flow the user just paused themselves**, in the
  same conversation. They know the state.

## Red flags - STOP and run the gate

- "The user said run, so I run"
- "It is their flow, they know what it does"
- "The manifest is probably stale anyway"
- "I will mention the overwrite afterwards"
- "It only writes documentation, that is harmless"
- "Asking breaks the flow of vibe coding"
- About to call the run command without having opened `.flows/.runs/`

**Every one of these means: read the manifest first.**

## Rationalizations

| Excuse | Reality |
|---|---|
| "Running again is idempotent" | It is not. Every node reruns and rewrites, with a fresh model call each time |
| "It will resume where it stopped" | Only a paused run resumes. A completed one starts over |
| "The previous output is still there for comparison" | It is discarded the moment the new run starts |
| "A failed run left nothing behind" | The nodes before the failure completed and wrote |
| "The user asked, that is authorisation enough" | They asked to run, not to overwrite files they may have forgotten about |
| "Checking is slower than just running" | Reading one YAML costs a second. Regenerating a document the user hand-edited costs their afternoon |
| "The panel would have warned" | The panel shows no history at all |

## After the run

The manifest is rewritten with the new run, replacing the old one. If the user
needs the previous state, that is version control, not the manifest: say so
**before** running, not after.
