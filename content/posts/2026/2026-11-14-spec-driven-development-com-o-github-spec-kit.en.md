---
title: "Spec-Driven Development in practice with GitHub Spec Kit"
slug: "2026-11-14-spec-driven-development-with-github-spec-kit"
description: "How I use GitHub Spec Kit with an AI agent: project constitution, spec, plan, tasks and implementation, with real examples from the infrastructure PR reviewer."
images: ["/images/capa-spec-kit.png"]
author: Jason
date: 2026-11-14
tags: [ "AI", "DevOps" ]
type: post
---

When I started using AI agents to write code, the workflow was to
chat until the result looked right. It works for small tasks, but
in a bigger change the decisions end up scattered across the conversation, and the following
week nobody (neither me nor the agent) remembers why something was done one
way and not another.

**Spec-Driven Development** (SDD) flips this around: before the code, you
write **what** you want and **why**; then **how** to do it; and only then
break it into tasks and implement. **GitHub Spec Kit** is an open source kit
that organizes this workflow into commands the AI agent runs, generating
Markdown files that are versioned alongside the code.

In this post I'll show how I use Spec Kit, with real examples from the project in the
series
[Infrastructure PR review with AI]({{< relref "/posts/2026/2026-11-01-revisao-de-pr-de-infraestrutura-com-ia-parte-1" >}}).
The project's specs are written in Portuguese, so the excerpts below are
too; I explain each one in the text.

## The workflow

Spec Kit's minimal workflow has five steps, each with its own command:

1. **Constitution** (`/speckit-constitution`): the project's principles,
   written once and valid for every feature.
2. **Spec** (`/speckit-specify`): what the feature does and why, from the
   user's point of view. No technology.
3. **Plan** (`/speckit-plan`): how to implement it, with the technical choices,
   checked against the constitution.
4. **Tasks** (`/speckit-tasks`): the plan broken into small, ordered
   tasks.
5. **Implementation** (`/speckit-implement`): the agent executes the tasks.

There are also supporting commands: `/speckit-clarify` to raise questions about the
spec, `/speckit-checklist` to check the quality of the requirements,
`/speckit-analyze` to look for inconsistencies between spec, plan and
tasks, and `/speckit-converge` to check whether the implementation meets
what was specified.

The rule that makes the biggest difference: **review each file before moving on to
the next step**. The spec is the cheapest place to fix a misunderstanding.

## Installation

Spec Kit needs Python 3.11 or newer. The official way to install the
`specify` CLI is with `uv`:

```bash
uv tool install specify-cli
```

I don't have `uv` on my machine, so I install the package in a virtual
environment, with the version pinned:

```bash
python3 -m venv ~/.venvs/speckit
~/.venvs/speckit/bin/pip install specify-cli==1.2.0
```

To start a new project, `specify init project-name`. To adopt it
in an existing repository, run it inside the repository with `--here`, passing the
integration for the agent you use. In my case, Claude Code:

```bash
cd my-repo
~/.venvs/speckit/bin/specify init --here --integration claude
```

`init` creates the `.specify/` folder, with the templates, the scripts and the
constitution, and installs the Spec Kit commands as agent skills. With
Claude Code, they go in `.claude/skills/speckit-*`.

## The constitution

The constitution is the `.specify/memory/constitution.md` file. It lists the
principles that no feature can violate, each with a
rationale. In the AI PR reviewer, for example, there are six
principles: no long-lived credentials, least privilege, the AI explains but doesn't
decide, validated output, testable without AWS and documentation in Portuguese.

One of them, as an example of the format (principle III, "The AI explains, it doesn't decide":
the review must only be a PR comment, never approve, block or change the PR, and model
failures must become a clear message, never an invented result):

```markdown
### III. A IA explica, não decide

A revisão MUST ser apenas um comentário no Pull Request. Ela NEVER aprova, bloqueia ou altera o PR,
e o job MUST NOT falhar por causa do conteúdo da revisão. Falhas do modelo (recusa, resposta
cortada, cota, acesso negado) MUST virar uma mensagem clara no comentário ou no log, nunca um
resultado inventado.

Racional: a decisão de merge é humana; a IA é apoio.
```

The words `MUST`, `MUST NOT` and `MAY` make it clear what is mandatory and
what is optional, and the "Racional" (rationale) explains the reason, which helps when someone
(or the agent) has to decide a case the text didn't foresee.

The constitution also has a version, with semver: MAJOR when a principle is
removed or redefined, MINOR when a principle is added, PATCH for
wording tweaks. This is useful, as we'll see further down.

## The spec

Each feature gets a numbered folder in `specs/`. The first
feature I specified in the reviewer was letting users choose the AI
model. The initial request was one sentence:

> Let the user choose the model they want to use or create a
> list with the available models.

`/speckit-specify` turned that into `specs/001-selecao-de-modelo/spec.md`,
with prioritized user stories, acceptance scenarios and numbered
requirements. An excerpt (user story 1, "choose the review model": with no model configured,
the review uses the default; with a model outside the supported list, the model isn't called and the
comment lists the accepted ones):

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

And the requirements (choose the model through the repository configuration, without code changes;
reject an unsupported model before any call):

```markdown
- **FR-001**: O revisor MUST permitir escolher o modelo pela configuração do repositório, sem mudança de código.
- **FR-004**: O revisor MUST recusar, antes de qualquer chamada, um modelo fora da lista de suportados,
  informando no comentário os modelos aceitos.
```

The spec also lists the edge cases, such as "the model exists on Bedrock but
isn't in the supported list" or "the GitHub role doesn't have permission for
the chosen model". It was while reviewing that list that I noticed error scenarios
I wouldn't have thought of before writing the code.

Along with the spec, Spec Kit generates a quality checklist in
`checklists/requirements.md`, with items like "no implementation
details" and "measurable success criteria".

## The plan

`/speckit-plan` generates `plan.md`, now with the technical decisions:
language, versioned dependencies, how to test, file structure. It can
also generate `research.md` (alternatives researched), `data-model.md`,
`contracts/` and `quickstart.md`.

The part I like most is the **Constitution Check**: a table that shows
how the plan meets each principle of the constitution, before any
line of code. In the model selection plan:

```markdown
| Princípio | Como o plano atende | Status |
|---|---|---|
| I. Sem credenciais fixas | Converse e Mantle usam a cadeia padrão de credenciais (OIDC/SSO) | ✅ |
| II. Menor privilégio | Lista fechada; IAM libera só os 2 ARNs gpt-oss | ✅ |
| V. Testável sem AWS | Stubber do botocore para o Converse; servidor local para o Mantle | ✅ |
```

(No long-lived credentials: Converse and Mantle use the standard credential chain. Least
privilege: a closed list, and IAM only allows the two gpt-oss ARNs. Testable without AWS:
botocore's Stubber for Converse and a local server for Mantle.)

If the plan violates a principle, it shows up here, not in code review.

## Tasks and implementation

`/speckit-tasks` breaks the plan into `tasks.md`, organized in phases: setup,
common foundation and one phase per user story. Each task has an ID, the
story it belongs to and the file it will change. `[P]` marks the
tasks that can run in parallel:

```markdown
## Phase 3: User Story 1 - Escolher o modelo da revisão (P1) 🎯 MVP

- [x] T004 [P] [US1] Testes: `resolve_model` (vazio, válido, inválido) em `tests/test_review.py`
- [x] T006 [US1] Implementar `review_schema()` (schema expandido, sem `$ref`) em `reviewer/review.py`
- [x] T008 [US1] Em `main()`, escolher Mantle ou Converse pelo `api` do modelo; modelo não suportado vira
  mensagem com a lista de aceitos, sem chamar o Bedrock, em `reviewer/review.py`
```

With `/speckit-implement`, the agent follows the list and checks off each completed
task. Since the constitution requires tests without AWS, the test tasks
come before the implementation ones.

## When reality disagrees with the constitution

In [part 2 of the series]({{< relref "/posts/2026/2026-11-02-revisao-de-pr-de-infraestrutura-com-ia-parte-2" >}})
I told how gpt-oss sometimes returned truncated findings. The original constitution said any response outside
the schema was a failure, and the whole review was thrown away. With bigger diffs,
that happened in most runs.

Instead of "hacking around it" in the code, the change was made in the constitution,
with an amendment that changed Principle IV to allow dropping malformed
findings **one by one**, as long as the comment says how many were
dropped. Since the amendment redefines a principle, the version went from 1.0.0 to
2.0.0, and the reason was recorded in the file itself (with bigger diffs, gpt-oss returns
truncated findings and the whole review was lost in most runs):

```markdown
- Version change: 1.0.0 → 2.0.0 (MAJOR: redefine o Princípio IV)
- Motivo: com diffs maiores, o gpt-oss devolve achados cortados (aspas sem escape) e a revisão inteira
  era perdida na maioria das execuções (PR #4)
```

Six months from now, whoever opens the code and wonders why some findings are
dropped will find the decision and the reason in the repository, not lost
in a conversation.

## What changed in my day to day

- **Less rework**: misunderstandings show up in the spec, which is text,
  not in the code.
- **Recorded decisions**: spec, plan and constitution are versioned in the
  repository and go into the pull request along with the code.
- **A more predictable agent**: with the constitution as a fixed rule, the agent
  doesn't "forget" requirements like "never use access keys" in the middle of a
  long conversation.

The cost is more text to read before seeing code run. For a one-line
fix, it isn't worth it. For any change in behavior, it has been.

The Spec Kit files for the current project, the reviewer's reusable actions,
are in the
[jjasonhenrique1986/github-actions](https://github.com/jjasonhenrique1986/github-actions) repository,
in the `.specify/` and `specs/` folders.

## References

- [GitHub Spec Kit](https://github.com/github/spec-kit)
- [Spec Kit documentation](https://github.github.io/spec-kit/)
- [Spec Kit in existing projects](https://github.github.io/spec-kit/guides/existing-projects.html)
