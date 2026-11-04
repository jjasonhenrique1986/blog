---
title: "Um módulo Terraform para a role OIDC do GitHub Actions"
description: "Como transformar a role IAM do GitHub Actions com OIDC em um módulo Terraform reutilizável, com validações que bloqueiam curingas e AdministratorAccess, testes com terraform test e versionamento por tag."
images: ["/images/capa-modulo-terraform-oidc.png"]
author: Jason
date: 2026-11-28
tags: [ "Terraform", "IaC", "AWS", "CI/CD", "Segurança" ]
type: post
---

No post
[GitHub Actions com OpenID Connect na AWS, sem access keys](/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys/)
criamos uma role IAM que o GitHub Actions assume com OIDC, sem guardar
nenhuma credencial fixa. Funciona bem para um repositório. O problema
aparece quando cada projeto ganha a sua própria role, que é o que eu
recomendo: um repositório comprometido só consegue fazer o que a role
**dele** permite.

Com várias roles, a trust policy passa a ser copiada e colada, e é aí que os
erros aparecem: um `sub` com curinga que aceita qualquer branch, uma role de
CI com `AdministratorAccess` "só para testar". Neste post vou mostrar o
módulo Terraform que uso para criar essas roles, com validações que
recusam esses erros antes do `apply`, testes com `terraform test` e
versionamento por tag.

## O que o módulo faz

Para cada projeto, o módulo cria:

- Uma role com trust policy presa ao provedor OIDC do GitHub, com `aud`
  igual a `sts.amazonaws.com` e uma lista **explícita** de valores de `sub`.
- Opcionalmente, uma policy inline com as permissões do projeto.
- Opcionalmente, managed policies anexadas, desde que não seja
  `AdministratorAccess`.

O provedor OIDC `token.actions.githubusercontent.com` fica **fora** do
módulo, porque ele é criado uma vez por conta, e não uma vez por projeto.

A estrutura é a de um repositório de módulos, com um diretório por módulo:

```text
terraform-modules/
└── modules/
    └── github-oidc-role/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        ├── versions.tf
        ├── README.md
        └── tests/
            └── github_oidc_role.tftest.hcl
```

## O formato do sub

Antes do código, um detalhe que mudou em 2026. O GitHub passou a usar um
formato **imutável** no claim `sub` dos tokens OIDC: repositórios criados a
partir de 15 de julho de 2026 incluem os IDs numéricos do dono e do
repositório. Em vez de:

```text
repo:meu-usuario/meu-repo:ref:refs/heads/main
```

o `sub` passa a ser:

```text
repo:meu-usuario@123456/meu-repo@456789:ref:refs/heads/main
```

Isso evita que alguém que crie uma conta ou repositório com um nome que você
apagou herde o acesso às suas roles. A consequência prática é que você
precisa saber qual formato o seu repositório usa antes de escrever a trust
policy. O GitHub mostra o prefixo pela API:

```bash
gh api repos/OWNER/REPO/actions/oidc/customization/sub --jq .sub_claim_prefix
```

Por isso o módulo recebe o `sub` em duas partes: o **prefixo** do
repositório (`github_sub_prefix`), que você copia da saída desse comando, e
os **sufixos** dos eventos que podem assumir a role (`subjects`), como
`pull_request` ou `ref:refs/heads/main`.

## As variáveis e as validações

A proteção do módulo está nos blocos `validation` das variáveis. O prefixo
precisa começar com `repo:` e não pode terminar com `:`:

```hcl
variable "github_sub_prefix" {
  type        = string
  description = "Prefixo do claim sub do repositório, por exemplo repo:meu-usuario@123456/meu-repo@456789"

  validation {
    condition     = startswith(var.github_sub_prefix, "repo:") && !endswith(var.github_sub_prefix, ":")
    error_message = "Use o prefixo no formato repo:OWNER/REPO (ou repo:OWNER@ID/REPO@ID), sem ':' no final."
  }
}
```

Os sufixos precisam existir e não podem ter curinga:

```hcl
variable "subjects" {
  type        = list(string)
  description = "Sufixos do sub aceitos, por exemplo [\"pull_request\"] ou [\"ref:refs/heads/main\"]"

  validation {
    condition     = length(var.subjects) > 0 && alltrue([for s in var.subjects : !strcontains(s, "*")])
    error_message = "Informe ao menos um subject e não use curingas (*)."
  }
}
```

E a lista de managed policies recusa `AdministratorAccess`:

```hcl
variable "managed_policy_arns" {
  type        = list(string)
  default     = []
  description = "ARNs de managed policies para anexar. Evite AdministratorAccess em roles de CI."

  validation {
    condition     = !anytrue([for arn in var.managed_policy_arns : endswith(arn, ":policy/AdministratorAccess")])
    error_message = "AdministratorAccess não é permitido em role de CI."
  }
}
```

As outras variáveis são o nome da role, o ARN do provedor OIDC, a policy
inline (`inline_policy_json`), a duração máxima da sessão, a descrição e as
tags.

## A trust policy

No `main.tf`, o módulo monta a lista completa de `sub` juntando o prefixo
com cada sufixo, e usa `StringEquals` em vez de `StringLike`. Com
`StringEquals`, um curinga que escapasse da validação seria tratado como o
caractere `*` literal, e não como "qualquer coisa":

```hcl
locals {
  oidc_host = "token.actions.githubusercontent.com"

  # Lista explícita de subs aceitos, sem curingas
  allowed_subs = [for s in var.subjects : "${var.github_sub_prefix}:${s}"]
}

resource "aws_iam_role" "this" {
  name                 = var.name
  description          = var.description
  max_session_duration = var.max_session_duration
  tags                 = var.tags

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = var.oidc_provider_arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${local.oidc_host}:aud" = "sts.amazonaws.com"
          "${local.oidc_host}:sub" = local.allowed_subs
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "this" {
  count  = var.inline_policy_json == null ? 0 : 1
  name   = "${var.name}-inline"
  role   = aws_iam_role.this.id
  policy = var.inline_policy_json
}

resource "aws_iam_role_policy_attachment" "this" {
  for_each   = toset(var.managed_policy_arns)
  role       = aws_iam_role.this.name
  policy_arn = each.value
}
```

As saídas são o `role_arn`, o `role_name` e o `allowed_subs`, que ajuda a
conferir no `plan` exatamente quais valores de `sub` a role vai aceitar.

## Testes sem acesso à AWS

O `terraform test`, disponível desde o Terraform 1.6, roda o módulo e
verifica asserções. Com `mock_provider`, que chegou no 1.7, o provider da
AWS é simulado: os testes rodam sem credenciais e sem criar nada. É por isso
que o `versions.tf` do módulo exige `required_version = ">= 1.7"`.

O arquivo de testes define valores padrão para as variáveis e quatro
cenários:

```hcl
mock_provider "aws" {}

variables {
  name              = "gh-teste"
  oidc_provider_arn = "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com"
  github_sub_prefix = "repo:meu-usuario@123456/meu-repo@7891011"
  subjects          = ["pull_request", "ref:refs/heads/main"]
}

run "trust_policy_restrita_ao_repo" {
  command = plan

  assert {
    condition = jsondecode(aws_iam_role.this.assume_role_policy).Statement[0].Condition.StringEquals["token.actions.githubusercontent.com:sub"] == [
      "repo:meu-usuario@123456/meu-repo@7891011:pull_request",
      "repo:meu-usuario@123456/meu-repo@7891011:ref:refs/heads/main",
    ]
    error_message = "O sub da trust policy não corresponde ao repositório e eventos informados."
  }

  assert {
    condition     = jsondecode(aws_iam_role.this.assume_role_policy).Statement[0].Condition.StringEquals["token.actions.githubusercontent.com:aud"] == "sts.amazonaws.com"
    error_message = "O aud precisa ser sts.amazonaws.com."
  }

  assert {
    condition     = length(aws_iam_role_policy.this) == 0
    error_message = "Sem inline_policy_json não deve haver política inline."
  }
}

run "politica_inline" {
  command = plan

  variables {
    inline_policy_json = "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"s3:ListBucket\",\"Resource\":\"*\"}]}"
  }

  assert {
    condition     = length(aws_iam_role_policy.this) == 1
    error_message = "A política inline deveria ser criada."
  }
}

run "bloqueia_curinga_no_subject" {
  command = plan

  variables {
    subjects = ["*"]
  }

  expect_failures = [var.subjects]
}

run "bloqueia_admin" {
  command = plan

  variables {
    managed_policy_arns = ["arn:aws:iam::aws:policy/AdministratorAccess"]
  }

  expect_failures = [var.managed_policy_arns]
}
```

Os dois últimos cenários usam `expect_failures`: o teste **passa** quando a
validação da variável **falha**. É assim que garanto que as proteções
continuam funcionando se alguém mexer no módulo.

Para rodar:

```bash
cd modules/github-oidc-role
terraform init
terraform test
```

A saída:

```text
tests/github_oidc_role.tftest.hcl... in progress
  run "trust_policy_restrita_ao_repo"... pass
  run "politica_inline"... pass
  run "bloqueia_curinga_no_subject"... pass
  run "bloqueia_admin"... pass
tests/github_oidc_role.tftest.hcl... tearing down
tests/github_oidc_role.tftest.hcl... pass

Success! 4 passed, 0 failed.
```

## CI do repositório de módulos

No GitHub Actions, um workflow roda `fmt`, `validate` e `test` em cada pull
request, com uma matriz que tem um item por módulo. Para adicionar um módulo
novo, basta incluí-lo na lista:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: [github-oidc-role]
    defaults:
      run:
        working-directory: modules/${{ matrix.module }}
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
      - run: terraform fmt -check -recursive
      - run: terraform init -backend=false
      - run: terraform validate
      - run: terraform test
```

Como os testes usam `mock_provider`, o workflow não precisa de nenhuma
credencial da AWS.

## Versionando com tags

Cada versão do módulo é uma tag `vX.Y.Z` no repositório, e quem usa o
módulo fixa a versão no `source` com `?ref=`. Assim, uma mudança no módulo
só chega a um projeto quando alguém atualiza a tag de propósito, num pull
request que mostra o `plan` da mudança.

## Usando o módulo

O provedor OIDC é criado uma vez na conta. Não é mais preciso informar o
thumbprint do certificado: a AWS valida o certificado do GitHub por conta
própria.

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
}
```

E cada projeto vira um bloco `module`. No exemplo, o repositório
`meu-repo` pode assumir a role só em pull requests, e só para ler um bucket:

```hcl
data "aws_iam_policy_document" "meu_repo" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["arn:aws:s3:::meu-bucket/*"]
  }
}

module "gh_meu_repo" {
  source = "git::https://github.com/meu-usuario/terraform-modules.git//modules/github-oidc-role?ref=v0.1.0"

  name               = "gh-meu-repo"
  description        = "GitHub Actions do meu-usuario/meu-repo (OIDC)"
  oidc_provider_arn  = aws_iam_openid_connect_provider.github.arn
  github_sub_prefix  = "repo:meu-usuario@123456/meu-repo@456789"
  subjects           = ["pull_request"]
  inline_policy_json = data.aws_iam_policy_document.meu_repo.json
}

output "gh_meu_repo_role_arn" {
  value = module.gh_meu_repo.role_arn
}
```

Depois do `apply`, guardo o ARN da role como o secret `AWS_ROLE_ARN` do
repositório. No workflow, a action da AWS assume a role e, com
`mask-aws-account-id`, esconde o ID da conta nos logs:

```yaml
- uses: aws-actions/configure-aws-credentials@v6
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: us-east-1
    mask-aws-account-id: true
```

Se você tentar passar `subjects = ["*"]` ou anexar `AdministratorAccess`, o
`plan` para na validação, antes de qualquer mudança na conta.

## Conclusão

Transformar a role em módulo tirou da trust policy o trabalho de copiar e
colar, que é onde os erros de segurança costumam entrar. As validações
recusam curingas e `AdministratorAccess`, os testes garantem que essas
proteções continuam lá, e a tag no `source` deixa cada projeto atualizar o
módulo no próprio ritmo.

O state desses recursos fica num bucket S3 com lock nativo, como mostrei em
[Terraform: state locking nativo no S3, sem DynamoDB](/posts/2026/2026-11-21-terraform-state-locking-nativo-no-s3-sem-dynamodb/).

## Referências

- [OpenID Connect reference: immutable subject claims](https://docs.github.com/en/actions/reference/security/oidc#immutable-subject-claims)
- [Testes no Terraform](https://developer.hashicorp.com/terraform/language/tests)
- [Mocks no terraform test](https://developer.hashicorp.com/terraform/language/tests/mocking)
- [Validação de variáveis no Terraform](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions#input-variable-validation)
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
