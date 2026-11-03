---
title: "Spec-Driven Development na prática com o GitHub Spec Kit"
description: "Como uso o GitHub Spec Kit com um agente de IA: constituição do projeto, spec, plano, tarefas e implementação, com exemplos reais do revisor de PR de infraestrutura."
image: "/images/capa-spec-kit.png"
author: Jason
date: 2026-12-05
tags: [ "IA", "DevOps" ]
type: post
showTableOfContents: true
---

Quando comecei a usar agentes de IA para escrever código, o fluxo era
conversar até o resultado parecer certo. Funciona para tarefas pequenas, mas
numa mudança maior as decisões ficam espalhadas pela conversa, e na semana
seguinte ninguém (nem eu, nem o agente) lembra por que algo foi feito de um
jeito e não de outro.

O **Spec-Driven Development** (SDD) inverte isso: antes do código, você
escreve **o que** quer e **por quê**; depois, **como** fazer; e só então
quebra em tarefas e implementa. O **GitHub Spec Kit** é um kit open source
que organiza esse fluxo em comandos que o agente de IA executa, gerando
arquivos Markdown que ficam versionados junto com o código.

Neste post vou mostrar como uso o Spec Kit, com exemplos reais do projeto da
série
[Revisão de PR de infraestrutura com IA](/posts/2026/2026-11-01-revisao-de-pr-de-infraestrutura-com-ia-parte-1/).

## O fluxo

O fluxo mínimo do Spec Kit tem cinco passos, cada um com o seu comando:

1. **Constituição** (`/speckit-constitution`): os princípios do projeto,
   escritos uma vez e válidos para todas as funcionalidades.
2. **Spec** (`/speckit-specify`): o que a funcionalidade faz e por quê, do
   ponto de vista de quem usa. Sem tecnologia.
3. **Plano** (`/speckit-plan`): como implementar, com as escolhas técnicas,
   conferido contra a constituição.
4. **Tarefas** (`/speckit-tasks`): o plano quebrado em tarefas pequenas e
   ordenadas.
5. **Implementação** (`/speckit-implement`): o agente executa as tarefas.

Há também comandos de apoio: `/speckit-clarify` para levantar dúvidas da
spec, `/speckit-checklist` para checar a qualidade dos requisitos,
`/speckit-analyze` para procurar inconsistências entre spec, plano e
tarefas, e `/speckit-converge` para conferir se a implementação atende ao
que foi especificado.

A regra que mais faz diferença: **revisar cada arquivo antes de seguir para
o próximo passo**. A spec é o lugar mais barato para corrigir um mal-entendido.

## Instalação

O Spec Kit precisa de Python 3.11 ou mais novo. A forma oficial de instalar o
CLI `specify` é com o `uv`:

```bash
uv tool install specify-cli
```

Na minha máquina não tenho o `uv`, então instalo o pacote num ambiente
virtual, com a versão fixada:

```bash
python3 -m venv ~/.venvs/speckit
~/.venvs/speckit/bin/pip install specify-cli==1.2.0
```

Para começar um projeto novo, `specify init nome-do-projeto`. Para adotar
num repositório que já existe, rode dentro dele com `--here`, informando a
integração do agente que você usa. No meu caso, o Claude Code:

```bash
cd meu-repo
~/.venvs/speckit/bin/specify init --here --integration claude
```

O `init` cria a pasta `.specify/`, com os templates, os scripts e a
constituição, e instala os comandos do Spec Kit como skills do agente. Com
o Claude Code, eles ficam em `.claude/skills/speckit-*`.

## A constituição

A constituição é o arquivo `.specify/memory/constitution.md`. Ela lista os
princípios que nenhuma funcionalidade pode violar, cada um com uma
justificativa. No revisor de PR com IA, por exemplo, ficaram seis
princípios: sem credenciais fixas, menor privilégio, a IA explica mas não
decide, saída validada, testável sem AWS e documentação em português.

Um deles, como exemplo do formato:

```markdown
### III. A IA explica, não decide

A revisão MUST ser apenas um comentário no Pull Request. Ela NEVER aprova, bloqueia ou altera o PR,
e o job MUST NOT falhar por causa do conteúdo da revisão. Falhas do modelo (recusa, resposta
cortada, cota, acesso negado) MUST virar uma mensagem clara no comentário ou no log, nunca um
resultado inventado.

Racional: a decisão de merge é humana; a IA é apoio.
```

As palavras `MUST`, `MUST NOT` e `MAY` deixam claro o que é obrigatório e o
que é opcional, e o "Racional" explica o motivo, o que ajuda quando alguém
(ou o agente) precisa decidir um caso que o texto não previu.

A constituição também tem versão, com semver: MAJOR quando um princípio é
removido ou redefinido, MINOR quando um princípio é adicionado, PATCH para
ajustes de texto. Isso é útil, como veremos mais abaixo.

## A spec

Cada funcionalidade ganha uma pasta numerada em `specs/`. A primeira
funcionalidade que especifiquei no revisor foi permitir escolher o modelo de
IA. O pedido inicial foi uma frase:

> Permita que o usuário escolha o modelo que ele quer utilizar ou crie uma
> lista com os modelos disponíveis.

O `/speckit-specify` transformou isso em `specs/001-selecao-de-modelo/spec.md`,
com histórias de usuário priorizadas, cenários de aceite e requisitos
numerados. Um trecho:

```markdown
### User Story 1 - Escolher o modelo da revisão (Priority: P1)

Quem mantém o repositório escolhe, pela configuração do repositório, qual modelo de IA faz a revisão
dos Pull Requests, entre os modelos suportados pelo revisor. A troca não exige mudar código.

**Acceptance Scenarios**:

1. **Given** nenhum modelo configurado, **When** um PR é aberto, **Then** a revisão usa o modelo padrão.
3. **Given** o repositório configurado com um modelo fora da lista de suportados, **When** um PR é
   aberto, **Then** o modelo não é chamado e o comentário explica que o modelo não é suportado e lista
   os modelos aceitos.
```

E os requisitos:

```markdown
- **FR-001**: O revisor MUST permitir escolher o modelo pela configuração do repositório, sem mudança de código.
- **FR-004**: O revisor MUST recusar, antes de qualquer chamada, um modelo fora da lista de suportados,
  informando no comentário os modelos aceitos.
```

A spec também lista os casos de borda, como "o modelo existe no Bedrock mas
não está na lista de suportados" ou "a role do GitHub não tem permissão para
o modelo escolhido". Foi revisando essa lista que percebi cenários de erro
que eu não teria pensado antes de escrever o código.

Junto com a spec, o Spec Kit gera um checklist de qualidade em
`checklists/requirements.md`, com itens como "sem detalhes de
implementação" e "critérios de sucesso mensuráveis".

## O plano

O `/speckit-plan` gera o `plan.md`, agora sim com as decisões técnicas:
linguagem, dependências com versão, como testar, estrutura de arquivos. Pode
gerar também `research.md` (alternativas pesquisadas), `data-model.md`,
`contracts/` e `quickstart.md`.

A parte que eu mais gosto é o **Constitution Check**: uma tabela que mostra
como o plano atende a cada princípio da constituição, antes de qualquer
linha de código. No plano da escolha de modelo:

```markdown
| Princípio | Como o plano atende | Status |
|---|---|---|
| I. Sem credenciais fixas | Converse e Mantle usam a cadeia padrão de credenciais (OIDC/SSO) | ✅ |
| II. Menor privilégio | Lista fechada; IAM libera só os 2 ARNs gpt-oss | ✅ |
| V. Testável sem AWS | Stubber do botocore para o Converse; servidor local para o Mantle | ✅ |
```

Se o plano violar um princípio, isso aparece aqui, e não no code review.

## As tarefas e a implementação

O `/speckit-tasks` quebra o plano em `tasks.md`, organizado em fases: setup,
base comum e uma fase por história de usuário. Cada tarefa tem um ID, a
história a que pertence e o arquivo que vai alterar. O `[P]` marca as
tarefas que podem rodar em paralelo:

```markdown
## Phase 3: User Story 1 - Escolher o modelo da revisão (P1) 🎯 MVP

- [x] T004 [P] [US1] Testes: `resolve_model` (vazio, válido, inválido) em `tests/test_review.py`
- [x] T006 [US1] Implementar `review_schema()` (schema expandido, sem `$ref`) em `reviewer/review.py`
- [x] T008 [US1] Em `main()`, escolher Mantle ou Converse pelo `api` do modelo; modelo não suportado vira
  mensagem com a lista de aceitos, sem chamar o Bedrock, em `reviewer/review.py`
```

No `/speckit-implement`, o agente segue a lista e marca cada tarefa
concluída. Como a constituição exige testes sem AWS, as tarefas de teste
aparecem antes das de implementação.

## Quando a realidade discorda da constituição

Na [parte 2 da série](/posts/2026/2026-11-02-revisao-de-pr-de-infraestrutura-com-ia-parte-2/)
contei que o gpt-oss às vezes devolvia achados cortados. A constituição original dizia que qualquer resposta fora
do schema era falha, e a revisão inteira era descartada. Com diffs maiores,
isso acontecia na maioria das execuções.

Em vez de "dar um jeitinho" no código, a mudança foi feita na constituição,
com uma emenda que passou o Princípio IV a permitir descartar achados
malformados **um a um**, desde que o comentário diga quantos foram
descartados. Como a emenda redefine um princípio, a versão foi de 1.0.0 para
2.0.0, e o motivo ficou registrado no próprio arquivo:

```markdown
- Version change: 1.0.0 → 2.0.0 (MAJOR: redefine o Princípio IV)
- Motivo: com diffs maiores, o gpt-oss devolve achados cortados (aspas sem escape) e a revisão inteira
  era perdida na maioria das execuções (PR #4)
```

Daqui a seis meses, quem abrir o código e estranhar que alguns achados são
descartados encontra a decisão e o motivo no repositório, e não perdidos
numa conversa.

## O que mudou no meu dia a dia

- **Menos retrabalho**: os mal-entendidos aparecem na spec, que é texto, e
  não no código.
- **Decisões registradas**: spec, plano e constituição ficam versionados no
  repositório e entram no pull request junto com o código.
- **Agente mais previsível**: com a constituição como regra fixa, o agente
  não "esquece" requisitos como "nunca usar access keys" no meio de uma
  conversa longa.

O custo é mais texto para ler antes de ver código rodando. Para uma correção
de uma linha, não vale. Para qualquer mudança de comportamento, tem valido.

Os arquivos do Spec Kit do projeto atual, as actions reutilizáveis do
revisor, estão no repositório
[jjasonhenrique1986/github-actions](https://github.com/jjasonhenrique1986/github-actions),
nas pastas `.specify/` e `specs/`.

## Referências

- [GitHub Spec Kit](https://github.com/github/spec-kit)
- [Documentação do Spec Kit](https://github.github.io/spec-kit/)
- [Spec Kit em projetos existentes](https://github.github.io/spec-kit/guides/existing-projects.html)
