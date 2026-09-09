# vide-code-flow

Pacote de skills para projetar, escrever e auditar **flows** do
[vide-code](https://github.com/RAFAEL-SILVASOUZA): pipelines de agentes que
rodam em rodadas paralelas e passam resultado de um para o outro.

## As skills

| Skill | Quando ela entra |
|---|---|
| [`authoring-flow-yaml`](authoring-flow-yaml/SKILL.md) | Escrever ou editar o YAML na mão, ou entender o que cada propriedade aceita |
| [`creating-a-flow`](creating-a-flow/SKILL.md) | Transformar uma tarefa de várias etapas num pipeline, decidindo os nós por entrevista |
| [`reviewing-flow-integrity`](reviewing-flow-integrity/SKILL.md) | Auditar um flow antes de confiar trabalho real a ele, ou quando ele recusa salvar, rodar ou entrega handoff vazio |

`authoring-flow-yaml` é a referência; as outras duas a citam em vez de repetir o
schema.

## Instalação

Cada pasta é uma skill. Copie as que quiser para um dos diretórios que o
vide-code lê:

| Escopo | Diretório canônico | Também aceito |
|---|---|---|
| Projeto | `<workspace>/.agents/skills/` | `<workspace>/.claude/skills/` |
| Global | `~/.agents/skills/` | `~/.claude/skills/` |

```bash
git clone https://github.com/RAFAEL-SILVASOUZA/vide-code-flow.git
mkdir -p ~/.agents/skills
cp -r vide-code-flow/authoring-flow-yaml ~/.agents/skills/
cp -r vide-code-flow/creating-a-flow ~/.agents/skills/
cp -r vide-code-flow/reviewing-flow-integrity ~/.agents/skills/
```

O escopo de projeto tem precedência sobre o global, então uma cópia local
sobrepõe a instalada em `~`.

## O que é um flow, em cinco linhas

Um arquivo YAML em `<workspace>/.flows/<slug>.yaml`. Cada nó é um agente com o
próprio prompt; cada conexão diz que um roda antes do outro e, quando é do tipo
`data`, entrega a resposta dele. A execução acontece em rodadas: todo nó cujos
predecessores terminaram roda em paralelo. Nós sem nome ou sem prompt são
rascunhos legítimos, salvam normalmente e só impedem a execução.

```yaml
version: 2
name: Documentador
nodes:
  - id: estrutura
    name: Estrutura do Projeto
    prompt: Mapeie o repositorio e liste os pontos de entrada.
    x: 40
    y: 280
  - id: revisor
    name: Revisor
    prompt: |-
      Audite a secao "## Resultado de Estrutura do Projeto" abaixo e devolva a
      documentacao final ja corrigida.
    x: 360
    y: 280
edges:
  - from: estrutura
    to: revisor
    type: data
```

## Sobre a exatidão do conteúdo

O schema, as regras de validação, o passo da grade de posicionamento, o formato
do handoff e a semântica de execução descritos nestas skills foram extraídos do
código do vide-code (`FlowDomain`, `FlowStorageService`, `FlowRunnerService`),
não de memória. Todos os exemplos em YAML deste repositório foram validados
contra `FlowDomain.validateStructure`, `FlowDomain.validate` e
`FlowDomain.fromStorageNodes`, incluindo a checagem de que as coordenadas
sobrevivem ao encaixe na grade sem mudar de valor.

Vale contra o vide-code na versão 1.10.24, formato de flow versão 2.
