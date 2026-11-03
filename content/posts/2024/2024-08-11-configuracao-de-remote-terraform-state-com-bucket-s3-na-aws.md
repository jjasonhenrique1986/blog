---
title: "Configuração de Remote Terraform State com Bucket S3 na AWS"
description: "Como configurar o Terraform state remoto em um bucket S3 na AWS em vez de mantê-lo localmente."
image: "/images/terraform.jpg"
author: Jason
date: 2024-08-11
tags: [ "IaC", "AWS", "Terraform" ]
type: post
showTableOfContents: true
---

![Configuração de Remote Terraform State com Bucket S3 na AWS](/images/terraform.jpg)

O objetivo desse post é fazer a configuração de um remote terraform
state usando um bucket s3 em vez de deixar seu terraform state
localmente. E também criar algum recurso na aws para testar o
funcionamento do terraform.

## Pré-Requisitos 

Para continuar a configuração do seu remote terraform state serão
necessários os seguintes pré-requitios abaixo:

-   Uma conta na AWS
-   Um Usuário IAM
-   AWS CLI instalado.
-   Uma conta no github
-   Um bucket S3
-   CLI do terraform instalado

## Criação Usuário IAM 

Nesse primeiro passo vamos fazer a criação de um usuário IAM para usar
nos arquivos de configurações do terraform.

Para fazer a criação do usuário IAM seguir esse outro
[artigo](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/)
que escrevi há um tempo atrás.

Para a nossa configuração vamos anexar a policy **AdmistratorAccess** no
nosso usuário **blog** conforme imagem abaixo:

![Console do IAM com a policy AdministratorAccess anexada ao usuário blog](/images/terraform-2.jpg)

Foi anexada essa policy pois é apenas um lab para ambientes produtivos
devem ser colocadas as devidas permissões.

## Criação bucket S3 

Nesse próximo passo vamos criar o nosso bucket S3 que terá a função de
armazenar o nosso terraform state. Para isso devemos ir na console de
administração da AWS procurar pelo serviço S3 e clicar em **Create
Bucket** deve ser apresentado a tela abaixo:

![Tela Create bucket no console do S3](/images/terraform-3.jpg)

No meu caso vou colocar o nome do bucket de terraform-blog-jason e outro
detalhe é que esse bucket será criado na região us-east-1.

Devemos deixar habilitado a opção **Block all public access** conforme
imagem abaixo:

![Opção Block all public access habilitada na criação do bucket S3](/images/terraform-4.jpg)

Depois disso vamos clicar no botão **Create bucket** no final da tela.
Com isso o nosso bucket será criado conforme imagem abaixo:

![Bucket terraform-blog-jason criado no console do S3](/images/terraform-5.jpg)

## Criação repositório Github 

Agora vamos criar um repositório no Github para armazenar os nossos
arquivos de configurações do terraform conforme abaixo:

![Criação do repositório no GitHub com o template de .gitignore do Terraform](/images/terraform-6.jpg)

Na opção Add .gitignore adicionar o template do Terraform. Segue o link
do [repositório](https://github.com/jjasonhenrique/terraform-blog-jason)
criado.

Agora vamos fazer um clone desse repositório usando o comando abaixo:

```bash
git clone https://github.com/jjasonhenrique/terraform-blog-jason.git
```

## Criando os arquivos de configurações 

Agora vamos fazer a criação dos arquivos de configurações do terraform.
Como pré-requisito o usuário IAM já deve estar configurado via AWS CLI
conforme o artigo que escrevi anteriormente e que deixei o link no
começo desse artigo.

O primeiro arquivo que vamos criar é o **versions.tf** conforme abaixo:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Em seguida vamos fazer a criação do arquivo **provider.tf** conforme abaixo:

```hcl
provider "aws" {
  shared_config_files      = ["/home/jason/.aws/config"]
  shared_credentials_files = ["/home/jason/.aws/credentials"]
  profile                  = "blog"
}
```

Nesse arquivo passamos o usuário que será utilizado para a conexão com a
AWS.

Agora vamos fazer a criação do arquivo **main.tf** conforme abaixo:

```hcl
terraform {
  backend "s3" {
    bucket  = "terraform-blog-jason"
    key     = "terraform.tfstate"
    region  = "us-east-1"
    profile = "blog"
  }
}
```

No arquivo acima estamos fazendo a configuração de um bucket S3 na AWS
para que seja usado como remote terraform state.

> **Atualização (novembro de 2026):** a configuração acima não tem **state
> locking**, então duas pessoas (ou dois pipelines) rodando `terraform
> apply` ao mesmo tempo podem corromper o state. Veja como habilitar na
> seção [State locking no S3](#state-locking-no-s3).

Agora vamos criar um recurso na AWS criando o arquivo **ecs.tf** conforme
abaixo:

```hcl
resource "aws_ecs_cluster" "teste-blog" {
  name = "teste-blog"

  setting {
    name  = "containerInsights"
    value = "disabled"
  }
}
```

Lembrando que para a criação de um cluster ECS na AWS existem mais
configurações a serem feitas. É apenas um teste para validar o
funcionamento do remote terraform state.

Após a criação dos arquivos você deve commitar eles excutando os
comandos abaixo:

```bash
git add .
git commit -m "Criação Cluster ECS"
git push
```

## Validação 

Agora vamos validar o remote terraform state e a criação de um recurso
na AWS. Primeiramente vamos inicializar o nosso terraform state
executando o comando abaixo:

```bash
terraform init
```

A saída do comando deve ser similar a abaixo:

![Saída do comando terraform init configurando o backend S3](/images/terraform-7.jpg)

Agora vamos executar o comando terraform plan para validar o que será
criado na AWS. A saída deve ser similar a abaixo:

![Saída do comando terraform plan com o cluster ECS que será criado](/images/terraform-8.jpg)

Agora vamos executar o comando terraform apply -auto-approve para a
criação do recurso na AWS.

Para validar vamos no serviço ECS e verificar se o cluster ECS foi
criado. Deve ter criado conforme imagem abaixo:

![Cluster ECS teste-blog criado no console da AWS](/images/terraform-9.jpg)

Também vamos validar se foi criado um arquivo terraform.tfstate no
bucket conforme imagem abaixo:

![Arquivo terraform.tfstate gravado no bucket S3](/images/terraform-10.jpg)

Para destruir o recurso criado na AWS para esse teste executar o comando
abaixo:

```bash
terraform destroy --auto-approve
```

## State locking no S3

O state locking impede que duas execuções alterem o mesmo state ao mesmo
tempo: quem roda primeiro pega um lock, e a segunda execução espera ou
falha. Antigamente isso exigia uma tabela no DynamoDB
(`dynamodb_table`). A partir do Terraform 1.11 o backend S3 faz o lock
direto no bucket (a opção surgiu como experimental no 1.10), criando um
arquivo `.tflock` ao lado do state, e a opção com DynamoDB passou a ser
considerada obsoleta.

Para habilitar, basta adicionar `use_lockfile = true` no backend e exigir
uma versão do Terraform que suporte a opção:

```hcl
terraform {
  required_version = ">= 1.11"

  backend "s3" {
    bucket       = "terraform-blog-jason"
    key          = "terraform.tfstate"
    region       = "us-east-1"
    profile      = "blog"
    use_lockfile = true
  }
}
```

Depois de alterar o backend, rode `terraform init -reconfigure` para o
Terraform aplicar a nova configuração. A identidade que roda o Terraform
precisa de permissão para criar e apagar o objeto
`terraform.tfstate.tflock` no bucket (`s3:PutObject` e `s3:DeleteObject`),
além das permissões que já usava para o state.

Também vale habilitar o **versionamento** no bucket: se um state for
corrompido ou apagado por engano, dá para recuperar uma versão anterior.
Mais detalhes na
[documentação do backend S3](https://developer.hashicorp.com/terraform/language/backend/s3).

## Conclusão 

O Objetivo desse artigo foi demonstar como deve ser feito a configuração
de um remote terraform state usando um bucket S3 na AWS.

Para o próximo passo com Terraform, veja
[como usar a expressão for_each para criar subnets na AWS](/posts/2024/2024-08-18-como-usar-a-expressao-for_each-para-criar-subnets-na-aws-com-terraform/).

