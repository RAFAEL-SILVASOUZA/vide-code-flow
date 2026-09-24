---
name: creating-a-flow
title: Creating a Flow
description: Use when the user wants a new vide-code flow, an agent pipeline, or to turn a multi-step task into cooperating agents, and the shape of that pipeline has not been decided yet.
---

# Creating a flow

## Overview

A flow is worth building only when a task splits into steps that a single agent
would do worse in one pass. The hard part is not the YAML, it is deciding what
the steps are and what each one hands to the next.

**REQUIRED BACKGROUND:** `authoring-flow-yaml` has the schema, the file
location, the positioning grid and the connection types, `retry` included.
This skill is the process; that one is the reference.

**Core principle:** design the graph backwards from the deliverable, then write
the prompts. Never the other way round.

## Interview first

Ask these before writing anything. Ask them in one message, not one at a time,
and stop asking as soon as the answers are unambiguous.

1. **What is the final deliverable?** A file, a report, a review, a migration?
   This becomes the last node, and everything else exists to feed it.
2. **What does that final step need in hand before it can start?** Each answer
   is a candidate upstream node.
3. Repeat question 2 on each new node until the answers are things that already
   exist in the repository or in the user's head. That is the top of the graph.
4. **Which of these steps do not depend on each other?** Independent steps run
   in parallel, each branch at its own pace, and cost nothing extra in wall
   time.
5. **For each dependency: does the downstream step need the previous agent's
   TEXT, or only the state it left behind?** Text means `data`. State means
   `order`.
6. **If a step reviews another step's work, what happens when it finds a
   problem?** Fixing it itself (the reviewer rewrites the output) and sending
   it back (the original agent redoes it) are different designs. See below.
7. **What must be true before the flow starts?** Anything the flow cannot do
   itself belongs in the user's hands, not in a node. That includes every
   decision: nodes run unattended and never ask the user anything.

If the user's answer to question 1 collapses into a single step, say so and
suggest a plain agent instead. A one-node flow is a worse way to run one agent.

## Build in this order

The order matters, and it is the order the panel is designed around.

1. **Nodes with ids and positions, prompts empty.** Drafts are legal: a node
   without `name` or `prompt` saves fine. Getting the graph right first is
   cheaper than rewriting prompts after the shape changes.
2. **Edges, with the right type on each.** Now the graph is the contract.
3. **Names.** A `data` parent's name becomes the heading its children read, so
   name a node after what it *produces*: `Estrutura do Projeto`, not `Agente 1`.
4. **Prompts, downstream nodes last.** By the time you write a child's prompt
   you know exactly which sections it will receive and under which headings.
5. **Reasoning, per node.** It is on by default. Turn it off
   (`reasoning: false`) only where the node's job is mechanical.

## Writing a node prompt

Each node runs as its own unattended agent. The node's prompt is its task,
with the handoff sections appended, under a fixed system prompt that says the
final response is the deliverable. So:

- **Say what the node produces**, in what shape, because that text is what the
  next agent reads. "Return a bullet list of every public endpoint" beats "look
  at the endpoints".
- **Name the sections it will receive.** A child of a node named
  `Estrutura do Projeto` should say: use the `## Resultado de Estrutura do
  Projeto` section below as the repository map.
- **Do not tell a node about work another node owns.** Overlap makes two agents
  produce contradictory text and the downstream agent picks arbitrarily.
- **Keep the last node's output the deliverable**, not a summary of the flow.
- **Never ask the agent to check with the user.** It cannot, and it will
  guess. Settle the decision in the interview and write it into the prompt.

## Reasoning on or off

Every card has a Reasoning checkbox, on by default. Off, the node runs with
reasoning disabled even when the model has it enabled: faster and cheaper,
but worse at judgement.

- **Leave it on** for nodes that plan, review, validate, or decide between
  options. The quality of the whole flow rests on those nodes.
- **Turn it off** for nodes that collect or transcribe: listing files,
  running a command and reporting the output, or reformatting a handoff.

Do not ask the user about this node by node. Propose the split with the
graph and let them object.

## Fix it in place, or send it back?

A reviewer node has two honest designs, and mixing them up is easy to do
without noticing:

- **Fix it in place.** The reviewer rewrites the output itself and that
  rewrite is the deliverable. No `retry` edge needed; this is the default,
  and the right choice whenever the reviewer is capable of correcting what it
  finds (the `Revisor` in the worked example below does this).
- **Send it back.** The reviewer cannot, or should not, fix the problem
  itself — only the original specialist can redo the work properly (it needs
  tools, context or a skill the reviewer does not have). This needs a `retry`
  edge from the reviewer back to that node; see `authoring-flow-yaml` for the
  mechanics.

Ask this whenever a step's job is to check another step's work, before
picking either shape. Do not default to `retry` just because a validator
exists in the graph: it costs an extra tool call and, if it fans in from
several nodes, rejects all of them together, not just the broken one. If the
reviewer would only need to reject one specific upstream node at a time, give
that one its own dedicated validator instead of sharing one across several.

## Worked example

"Document this API: I want an overview, an endpoint reference, and someone to
check the result."

Backwards from the deliverable: the review needs both documents; both documents
need the repository structure; the structure needs nothing.

```yaml
version: 2
name: Documentador de API
nodes:
  - id: estrutura
    name: Estrutura do Projeto
    prompt: |-
      # Agente: Estrutura do Projeto
      Mapeie o repositorio e devolva: arquivos de entrada, rotas encontradas e
      onde cada uma esta definida. Uma linha por rota, com caminho de arquivo.
    x: 40
    y: 280
  - id: visao
    name: Visao Geral
    prompt: |-
      # Agente: Visao Geral
      Escreva a introducao da documentacao: o que a API faz e como autenticar.
      Use a secao "## Resultado de Estrutura do Projeto" abaixo como mapa do
      repositorio. Nao liste endpoints, outro agente cuida disso.
    x: 360
    y: 120
  - id: endpoints
    name: Referencia de Endpoints
    prompt: |-
      # Agente: Referencia de Endpoints
      Para cada rota listada na secao "## Resultado de Estrutura do Projeto"
      abaixo, documente metodo, caminho, parametros e resposta. Nada de
      introducao: outro agente escreve a visao geral.
    x: 360
    y: 440
  - id: revisor
    name: Revisor
    prompt: |-
      # Agente: Revisor
      Audite as duas secoes abaixo em busca de contradicao, rota faltando ou
      exemplo que nao bate com o codigo. Devolva a documentacao final ja
      corrigida, nao uma lista de problemas.
    x: 680
    y: 280
edges:
  - from: estrutura
    to: visao
    type: data
  - from: estrutura
    to: endpoints
    type: data
  - from: visao
    to: revisor
    type: data
  - from: endpoints
    to: revisor
    type: data
```

`visao` and `endpoints` share no edge, so both start the moment `estrutura`
finishes, in parallel. Both feed `revisor`, which starts when the slower of
the two finishes and sees two sections. Every node here writes or judges
text, so reasoning stays on everywhere; if `estrutura` only listed files, it
would be the candidate for `reasoning: false`.

## Where to put it

`<workspace>/.flows/<slug>.yaml`, where the slug is the flow name lowercased
with non-alphanumerics collapsed to `-`. The example above goes in
`.flows/documentador-de-api.yaml`. See `authoring-flow-yaml` for the details.

## Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| Writing prompts before the graph | Every shape change rewrites prompts | Nodes and edges first, prompts last |
| One node per file to be written | The flow becomes a loop, not a pipeline | One node per *kind of thinking*, not per artifact |
| `order` between steps that share text | Downstream agent invents the input | `data` whenever the text matters |
| Two nodes with overlapping scope | Contradictory handoffs | Give each node an exclusive job and say so in its prompt |
| Generic node names | Children read `## Resultado de Agente 2` | Name for the product |
| A final node that summarises the flow | The deliverable is lost | Last node outputs the artifact itself |
| Chain with no parallelism | Slow for no reason | Anything independent runs in parallel for free |
| Prompt that says "ask the user" or "confirm before" | The node cannot ask, so it guesses | Decide it in the interview and write the answer into the prompt |
| Reasoning off on a reviewer or validator | Saves time on the one node whose judgement the flow depends on | Keep reasoning on wherever the job is judgement |
| One validator fanned in from several specialists, expecting selective retry | Rejecting sends all of them back together, same feedback | Give each specialist its own validator if rejection must be selective |
| Reviewer meant to send work back, wired as plain `data` | It just writes a report nobody redoes anything from | Add the `retry` edge, and write the prompt to actually reject when needed |
