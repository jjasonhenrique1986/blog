---
title: "Terraform mock_provider: testando módulos sem credenciais"
description: "Como usar o mock_provider do Terraform 1.7 para rodar terraform test sem credenciais e sem criar recursos: valores gerados, mock_resource, mock_data, override_during, blocos override e arquivos .tfmock.hcl."
images: ["/images/capa-terraform-mock-provider.png"]
author: Jason
date: 2026-11-12
tags: [ "Terraform", "IaC", "AWS", "CI/CD" ]
type: post
---

No post
[Um módulo Terraform para a role OIDC do GitHub Actions](/posts/2026/2026-11-11-modulo-terraform-para-role-oidc-do-github-actions/)
usei o `mock_provider` para testar o módulo sem acesso à AWS, mas só
com um bloco vazio. Neste post vou mostrar o recurso com mais detalhe: o
que o provider simulado devolve, onde ele quebra e como controlar os
valores com `mock_resource`, `mock_data`, `override_during` e os blocos
`override_*`.

Todos os exemplos foram executados com o Terraform 1.17.1 e o provider da
AWS 6.73.0, sem nenhuma credencial configurada.

## O problema

O `terraform test`, disponível desde o Terraform 1.6, executa o módulo de
verdade. Com o provider real, um `run` com `command = apply` cria os
recursos na conta e os destrói no final, e mesmo um `command = plan`
precisa de credenciais para o provider consultar a AWS. Isso deixa os testes
lentos, custa dinheiro e obriga o CI do repositório de módulos a ter acesso
a uma conta.

O Terraform 1.7 trouxe o `mock_provider`. Ele substitui o provider por uma
versão simulada, que conhece o mesmo schema do provider real, mas não chama
nenhuma API. Os recursos passam a existir só no state em memória do teste.

## O módulo de exemplo

Vou usar um módulo pequeno, que cria um bucket de logs com o ID da conta no
nome e uma policy que permite escrever nele:

```hcl
data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "logs" {
  bucket = "${var.name}-${data.aws_caller_identity.current.account_id}"
}

data "aws_iam_policy_document" "write_logs" {
  statement {
    actions   = ["s3:PutObject"]
    resources = ["${aws_s3_bucket.logs.arn}/*"]
  }
}

resource "aws_iam_policy" "write_logs" {
  name   = "${var.name}-write-logs"
  policy = data.aws_iam_policy_document.write_logs.json
}
```

As saídas são o nome e o ARN do bucket e o ARN da policy:

```hcl
output "bucket_name" {
  value = aws_s3_bucket.logs.bucket
}

output "bucket_arn" {
  value = aws_s3_bucket.logs.arn
}

output "policy_arn" {
  value = aws_iam_policy.write_logs.arn
}
```

O módulo tem dois data sources e dois atributos calculados pela AWS (os
ARNs), e são esses pontos que o provider simulado precisa preencher.

## O primeiro teste

Os testes ficam em arquivos `.tftest.hcl` no diretório `tests/`. O mais
simples tem um `mock_provider` vazio:

```hcl
mock_provider "aws" {}

variables {
  name = "app"
}

run "plan_basico" {
  command = plan

  assert {
    condition     = aws_iam_policy.write_logs.name == "app-write-logs"
    error_message = "Nome da policy incorreto."
  }
}
```

Para rodar:

```bash
terraform init
terraform test
```

```text
tests/basico.tftest.hcl... in progress
  run "plan_basico"... pass
tests/basico.tftest.hcl... tearing down
tests/basico.tftest.hcl... pass

Success! 1 passed, 0 failed.
```

O `terraform init` ainda baixa o provider da AWS, porque o Terraform
precisa do schema dele. O que o teste dispensa são as credenciais.

## O que o provider simulado devolve

Os atributos que você define na configuração continuam valendo. Os
atributos calculados, que o provider real preencheria depois de criar o
recurso, recebem valores gerados conforme o tipo:

| Tipo | Valor gerado |
|------|--------------|
| String | 8 caracteres alfanuméricos aleatórios |
| Número | `0` |
| Booleano | `false` |
| Lista, set e map | vazio |

Rodando um `apply` com `terraform test -verbose`, dá para ver o state do
teste. Este é um trecho da saída:

```text
# data.aws_caller_identity.current:
data "aws_caller_identity" "current" {
    account_id = "o0a5b2kv"
    arn        = "3idg4vkf"
    id         = "79iqltu5"
    user_id    = "wz7ryi77"
}

# aws_s3_bucket.logs:
resource "aws_s3_bucket" "logs" {
    arn    = "htm7dneh"
    bucket = "app-o0a5b2kv"
    id     = "4agb9ujq"
    ...
}

Outputs:

bucket_arn = "htm7dneh"
bucket_name = "app-o0a5b2kv"
policy_arn = "5kkbdrot"
```

O ID da conta virou `o0a5b2kv` e o ARN do bucket virou `htm7dneh`. O
provider simulado não sabe o formato de um ARN nem de um ID de conta. Ele só
sabe que o atributo é uma string.

## Onde o mock vazio quebra

Esse comportamento causa dois problemas no módulo de exemplo.

O primeiro aparece no `apply`. O data source `aws_iam_policy_document` não
é executado, então o atributo `json` também recebe uma string aleatória. O
`aws_iam_policy` valida o conteúdo de `policy` e recusa:

```text
  run "apply_sem_defaults"... fail

Error: "policy" contains an invalid JSON policy: not a JSON object

  with aws_iam_policy.write_logs,
  on main.tf line 16, in resource "aws_iam_policy" "write_logs":
  16:   policy = data.aws_iam_policy_document.write_logs.json
```

Ou seja, as validações do provider continuam rodando, mesmo com ele
simulado.

O segundo aparece no `plan`. Por padrão, os valores são gerados só no
`apply`, então no `plan` um atributo calculado ainda é desconhecido. Um
`assert` sobre o ARN do bucket falha com:

```text
Error: Unknown condition value

  on tests/plan_unknown.tftest.hcl line 11, in run "arn_no_plan":
  11:     condition     = startswith(aws_s3_bucket.logs.arn, "arn:aws:s3:::")
    ├────────────────
    │ aws_s3_bucket.logs.arn is a string

Condition expression could not be evaluated at this time.
```

As próximas seções resolvem os dois.

## Valores padrão com mock_resource e mock_data

Dentro do `mock_provider`, os blocos `mock_resource` e `mock_data` definem
valores padrão para um tipo de recurso ou de data source. Os atributos que
você não informar continuam sendo gerados:

```hcl
mock_provider "aws" {
  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "111122223333"
    }
  }

  mock_data "aws_iam_policy_document" {
    defaults = {
      json = "{\"Version\":\"2012-10-17\",\"Statement\":[]}"
    }
  }

  mock_resource "aws_s3_bucket" {
    defaults = {
      arn = "arn:aws:s3:::app-111122223333"
    }
  }
}

variables {
  name = "app"
}

run "nome_e_arn_do_bucket" {
  command = apply

  assert {
    condition     = output.bucket_name == "app-111122223333"
    error_message = "O nome do bucket deveria terminar com o ID da conta."
  }

  assert {
    condition     = output.bucket_arn == "arn:aws:s3:::app-111122223333"
    error_message = "ARN do bucket incorreto."
  }

  assert {
    condition     = data.aws_iam_policy_document.write_logs.statement[0].resources == toset(["arn:aws:s3:::app-111122223333/*"])
    error_message = "A policy deveria liberar só os objetos do bucket de logs."
  }
}
```

Agora o `apply` passa, porque o `json` é uma policy válida, e os dois
primeiros `assert` conferem o nome e o ARN.

O terceiro `assert` merece atenção. Como o `aws_iam_policy_document` não é
executado, o `json` do teste é o valor fixo do `defaults`, e não a policy
que o módulo montaria. Testar o `json` não diz nada sobre o módulo. O que
dá para testar são os argumentos do data source, como a lista `resources` do
`statement`, que vêm da configuração.

## Valores no plan com override_during

Para usar os valores simulados já no `plan`, existe o atributo
`override_during`, disponível a partir do Terraform 1.11. O padrão é
`apply`:

```hcl
mock_provider "aws" {
  override_during = plan

  mock_resource "aws_s3_bucket" {
    defaults = {
      arn = "arn:aws:s3:::app-111122223333"
    }
  }
}

variables {
  name = "app"
}

run "arn_no_plan" {
  command = plan

  assert {
    condition     = startswith(aws_s3_bucket.logs.arn, "arn:aws:s3:::")
    error_message = "ARN do bucket fora do formato."
  }
}
```

Com isso, o `assert` que falhava com `Unknown condition value` passa. O
`override_during` também pode ser definido em um `mock_resource` ou
`mock_data` específico, e nesse caso vale o do bloco mais interno.

Se você usar esse atributo, o `required_version` do módulo precisa ser
`>= 1.11`, e não `>= 1.7`.

## Sobrescrevendo um recurso específico

O `mock_resource` e o `mock_data` valem para **todos** os recursos de um
tipo. Para definir o valor de um recurso específico, existem os blocos
`override_resource`, `override_data` e `override_module`. Eles recebem um
`target` com o endereço do recurso e podem ficar na raiz do arquivo, valendo
para todos os `run`, ou dentro de um `run`:

```hcl
mock_provider "aws" {}

override_data {
  target = data.aws_iam_policy_document.write_logs
  values = {
    json = "{\"Version\":\"2012-10-17\",\"Statement\":[]}"
  }
}

variables {
  name = "app"
}

run "conta_de_producao" {
  command = apply

  override_data {
    target = data.aws_caller_identity.current
    values = {
      account_id = "999988887777"
    }
  }

  assert {
    condition     = output.bucket_name == "app-999988887777"
    error_message = "O nome do bucket deveria usar a conta de produção."
  }
}

run "conta_de_teste" {
  command = apply

  override_data {
    target = data.aws_caller_identity.current
    values = {
      account_id = "111122223333"
    }
  }

  assert {
    condition     = output.bucket_name == "app-111122223333"
    error_message = "O nome do bucket deveria usar a conta de teste."
  }
}
```

Assim cada `run` simula uma conta diferente. Quando o mesmo `target`
aparece na raiz do arquivo e dentro de um `run`, vale o do `run`.

O `override_module` faz o mesmo para um módulo filho inteiro: em vez de
`values`, ele recebe `outputs` com as saídas que o módulo devolveria. Os
blocos `override_*` também funcionam com o provider real, o que permite
simular só uma parte da configuração.

## Reaproveitando os mocks em arquivos

Quando vários arquivos de teste precisam dos mesmos valores, os blocos
`mock_resource` e `mock_data` podem ficar em um arquivo `.tfmock.hcl`:

```hcl
# tests/mocks/aws.tfmock.hcl
mock_data "aws_caller_identity" {
  defaults = {
    account_id = "111122223333"
  }
}

mock_data "aws_iam_policy_document" {
  defaults = {
    json = "{\"Version\":\"2012-10-17\",\"Statement\":[]}"
  }
}

mock_resource "aws_s3_bucket" {
  defaults = {
    arn = "arn:aws:s3:::app-111122223333"
  }
}
```

O `mock_provider` carrega o diretório com o atributo `source`:

```hcl
mock_provider "aws" {
  source = "./tests/mocks"
}

variables {
  name = "app"
}

run "usa_os_mocks_do_arquivo" {
  command = apply

  assert {
    condition     = output.bucket_name == "app-111122223333"
    error_message = "O nome do bucket deveria terminar com o ID da conta."
  }
}
```

O caminho é relativo ao diretório em que o `terraform test` é executado,
que aqui é a raiz do módulo. Se o mesmo tipo aparecer no arquivo e dentro
do `mock_provider`, vale o bloco do `mock_provider`.

Rodando todos os arquivos de teste deste post:

```text
tests/arquivo.tftest.hcl... in progress
  run "usa_os_mocks_do_arquivo"... pass
tests/arquivo.tftest.hcl... tearing down
tests/arquivo.tftest.hcl... pass
tests/defaults.tftest.hcl... in progress
  run "nome_e_arn_do_bucket"... pass
tests/defaults.tftest.hcl... tearing down
tests/defaults.tftest.hcl... pass
tests/override.tftest.hcl... in progress
  run "conta_de_producao"... pass
  run "conta_de_teste"... pass
tests/override.tftest.hcl... tearing down
tests/override.tftest.hcl... pass
tests/override_during.tftest.hcl... in progress
  run "arn_no_plan"... pass
tests/override_during.tftest.hcl... tearing down
tests/override_during.tftest.hcl... pass

Success! 5 passed, 0 failed.
```

## Misturando provider real e simulado

Um provider simulado pode conviver com o real no mesmo arquivo de teste,
desde que tenha um `alias`, como `mock_provider "aws" { alias = "fake" }`.
Cada `run` escolhe qual usar com o atributo `providers`.

Um detalhe que vi ao testar: basta existir um bloco `provider "aws"` real
no arquivo para o Terraform configurá-lo, mesmo que todos os `run` usem o
simulado. Sem credenciais, o teste falha com `No valid credential sources
found`. Se a ideia é rodar sem acesso à AWS, o arquivo de teste só pode ter
o `mock_provider`.

## Limitações

- O provider simulado não conhece o formato dos valores. ARNs, IDs e
  endpoints viram strings aleatórias, a menos que você defina um padrão.
- Data sources não são executados. O `aws_iam_policy_document` não monta o
  JSON e o `aws_caller_identity` não consulta a conta.
- O teste verifica a **lógica do módulo**: nomes, condições, validações de
  variáveis, o que é criado ou não. Ele não verifica se a AWS aceita a
  configuração. Um nome de bucket inválido ou uma permissão que falta só
  aparecem com o provider real.
- Os mocks só existem dentro do `terraform test`. Não dá para usá-los em um
  `terraform plan` comum.

Por isso os dois tipos de teste se complementam: os com `mock_provider`
podem rodar em todo pull request, e um teste com o provider real fica para
quando a mudança justifica criar recursos de verdade.

## Conclusão

O `mock_provider` deixa o `terraform test` rodar em segundos, sem
credenciais e sem custo, o que permite colocar os testes no CI de qualquer
repositório de módulos. O bloco vazio resolve os casos simples. Quando o
módulo depende de valores calculados, `mock_resource`, `mock_data` e os
blocos `override_*` definem esses valores, e o `override_during = plan`
os deixa disponíveis já no `plan`.

O workflow de CI que roda esses testes sem credenciais está no post
[Um módulo Terraform para a role OIDC do GitHub Actions](/posts/2026/2026-11-11-modulo-terraform-para-role-oidc-do-github-actions/).

## Referências

- [Mocks no terraform test](https://developer.hashicorp.com/terraform/language/tests/mocking)
- [Testes no Terraform](https://developer.hashicorp.com/terraform/language/tests)
- [Changelog do Terraform 1.7](https://github.com/hashicorp/terraform/blob/v1.7/CHANGELOG.md)
- [Changelog do Terraform 1.11](https://github.com/hashicorp/terraform/blob/v1.11/CHANGELOG.md)
