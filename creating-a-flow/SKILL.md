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
location, the positioning grid and the two connection types. This skill is the
process; that one is the reference.

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
   in the same parallel round and cost nothing extra in wall time.
5. **For each dependency: does the downstream step need the previous agent's
   TEXT, or only the state it left behind?** Text means `data`. State means
   `order`.
6. **What must be true before the flow starts?** Anything the flow cannot do
   itself belongs in the user's hands, not in a node.

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

## Writing a node prompt

Each node runs as its own agent with the node's prompt as its system prompt,
plus the handoff sections appended. So:

- **Say what the node produces**, in what shape, because that text is what the
  next agent reads. "Return a bullet list of every public endpoint" beats "look
  at the endpoints".
- **Name the sections it will receive.** A child of a node named
  `Estrutura do Projeto` should say: use the `## Resultado de Estrutura do
  Projeto` section below as the repository map.
- **Do not tell a node about work another node owns.** Overlap makes two agents
  produce contradictory text and the downstream agent picks arbitrarily.
- **Keep the last node's output the deliverable**, not a summary of the flow.

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

`visao` and `endpoints` share no edge, so they run in the same round, in
parallel. Both feed `revisor`, which therefore sees two sections.

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
| Chain with no parallelism | Slow for no reason | Anything independent shares a round for free |
