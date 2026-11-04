---
title: "Terraform: state locking nativo no S3, sem DynamoDB"
description: "Como funciona o state locking nativo do backend S3 do Terraform com use_lockfile, as permissões do IAM necessárias e como migrar do lock com DynamoDB, que está obsoleto."
images: ["/images/capa-terraform-lock-s3.png"]
author: Jason
date: 2026-11-21
tags: [ "Terraform", "IaC", "AWS" ]
type: post
---

No post sobre
[Remote Terraform State com bucket S3](/posts/2024/2024-08-11-configuracao-de-remote-terraform-state-com-bucket-s3-na-aws/)
o state ficava num bucket, mas sem nenhum controle de concorrência. Se duas
pessoas, ou dois pipelines, rodam `terraform apply` ao mesmo tempo no mesmo
state, as duas execuções leem a mesma versão, cada uma grava a sua, e a
última sobrescreve a primeira. É assim que se perde recurso no state.

O **state locking** resolve isso: antes de qualquer operação que possa
alterar o state, o Terraform pega um lock, e uma segunda execução falha (ou
espera) até o lock ser liberado.

Por muito tempo, o backend S3 só fazia lock com uma tabela no **DynamoDB**.
Desde o Terraform 1.10 ele consegue fazer o lock direto no bucket, e no
1.11 essa opção ficou estável e o lock com DynamoDB passou a ser
considerado obsoleto. Neste post vamos ver como o lock nativo funciona, como
habilitar e como migrar uma configuração que ainda usa DynamoDB.

## Como o lock nativo funciona

Com `use_lockfile = true`, o Terraform cria um arquivo com o mesmo nome do
state mais o sufixo `.tflock`. Para um state em `rede/terraform.tfstate`, o
lock fica em `rede/terraform.tfstate.tflock`.

O detalhe importante é **como** esse arquivo é criado. O Terraform faz um
`PutObject` com o cabeçalho `If-None-Match: *`, que é uma escrita
condicional do S3: o objeto só é gravado se ainda não existir. Se outra
execução já tiver criado o `.tflock`, o S3 recusa a escrita e o Terraform
sabe que o state está travado. Como a verificação e a escrita acontecem numa
única operação no S3, não existe a janela de corrida que haveria se o
Terraform primeiro checasse e depois gravasse.

O conteúdo do `.tflock` é um JSON com as informações do lock: um ID, quem
pegou o lock, a operação e a data. Para liberar, o Terraform lê o arquivo,
confere se o ID é o mesmo que ele criou e apaga o objeto.

Dá para ver isso no
[código do backend S3](https://github.com/hashicorp/terraform/blob/main/internal/backend/remote-state/s3/client.go)
do Terraform, nas funções `lockWithFile` e `unlockWithFile`.

## Habilitando o lock

Basta adicionar `use_lockfile = true` no bloco do backend. Como a opção só
é estável a partir do 1.11, vale exigir essa versão:

```hcl
terraform {
  required_version = ">= 1.11"

  backend "s3" {
    bucket       = "terraform-blog-jason"
    key          = "rede/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
  }
}
```

Depois de alterar o backend, rode o `init` de novo para o Terraform aplicar
a nova configuração:

```bash
terraform init -reconfigure
```

A partir daí, todo `plan`, `apply` e `destroy` pega o lock antes de começar
e solta no final.

## Permissões do IAM

Além das permissões que já eram necessárias para ler e gravar o state, a
identidade que roda o Terraform precisa de `s3:GetObject`, `s3:PutObject`
e `s3:DeleteObject` no arquivo `.tflock`. Note que o `DeleteObject` é
necessário só no lock: o Terraform nunca apaga o state.

Uma policy de menor privilégio para o state do exemplo fica assim:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::terraform-blog-jason"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::terraform-blog-jason/rede/terraform.tfstate"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::terraform-blog-jason/rede/terraform.tfstate.tflock"
    }
  ]
}
```

Se o pipeline roda no GitHub Actions, essa policy vai na role que o
workflow assume via OIDC, como mostrei em
[GitHub Actions com OpenID Connect na AWS, sem access keys](/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys/).

## Migrando do DynamoDB

Se o seu backend ainda tem `dynamodb_table`, a migração pode ser feita em
duas etapas, sem janela em que o state fique sem lock.

**Etapa 1: ligar os dois locks ao mesmo tempo.** O backend aceita
`use_lockfile` e `dynamodb_table` juntos. Nesse modo o Terraform pega os
dois locks, e a operação só segue se conseguir os dois:

```hcl
terraform {
  required_version = ">= 1.11"

  backend "s3" {
    bucket         = "terraform-blog-jason"
    key            = "rede/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock"
    use_lockfile   = true
  }
}
```

No `terraform init`, o Terraform já avisa que o parâmetro está com os dias
contados:

```text
Warning: Deprecated Parameter

  on main.tf line 8, in terraform:
   8:     dynamodb_table = "terraform-lock"

The parameter "dynamodb_table" is deprecated. Use parameter "use_lockfile"
instead.
```

Isso é útil quando várias pessoas ou pipelines usam o mesmo state e nem
todos foram atualizados ainda: quem está com a configuração antiga continua
respeitando o lock do DynamoDB, e quem está com a nova respeita os dois.

**Etapa 2: remover o DynamoDB.** Quando todo mundo estiver usando a
configuração nova, tire o `dynamodb_table`, rode `terraform init
-reconfigure` e, depois de confirmar que nada mais usa a tabela, apague-a.

## Quando o state está travado

Se outra execução estiver com o lock, o Terraform para com o erro
`Error acquiring the state lock`, seguido das informações do lock (ID,
caminho, operação, quem e quando) e desta explicação:

```text
Terraform acquires a state lock to protect the state from being written
by multiple users at the same time. Please resolve the issue above and try
again. For most commands, you can disable locking with the "-lock=false"
flag, but this is not recommended.
```

Algumas opções nessa situação:

- **Esperar o lock ser liberado.** Com `-lock-timeout`, o Terraform tenta
  de novo durante o tempo informado em vez de falhar na hora, o que ajuda
  em pipelines que rodam em sequência:

  ```bash
  terraform apply -lock-timeout=5m
  ```

- **Liberar um lock esquecido.** Se uma execução foi interrompida no meio
  (o runner caiu, alguém fechou o terminal), o `.tflock` pode ficar no
  bucket. Antes de liberar, confirme que ninguém está mesmo rodando o
  Terraform, e use o ID que aparece no erro:

  ```bash
  terraform force-unlock ID-DO-LOCK
  ```

- **Não usar `-lock=false`** no dia a dia. Ele desliga a proteção e volta
  exatamente ao problema do começo do post.

## Versionamento do bucket

O lock evita escritas concorrentes, mas não protege contra um state apagado
ou corrompido por engano. Para isso, a documentação do backend recomenda
habilitar o **versionamento** no bucket: com ele, dá para voltar a uma
versão anterior do state.

## Conclusão

Com `use_lockfile = true`, o lock do state passa a depender só do bucket
S3 que já guardava o state: uma tabela a menos para criar, pagar e dar
permissão. Se você ainda usa DynamoDB, a migração com os dois locks ligados
ao mesmo tempo permite trocar sem pressa.

## Referências

- [Backend S3 na documentação do Terraform](https://developer.hashicorp.com/terraform/language/backend/s3)
- [Comando terraform force-unlock](https://developer.hashicorp.com/terraform/cli/commands/force-unlock)
- [Conditional writes no Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html)
