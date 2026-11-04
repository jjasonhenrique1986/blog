---
title: "Criando e Configurando uma role IAM"
description: "Tutorial para criar e configurar uma role IAM na AWS, uma forma mais segura de acesso do que usar um usuário IAM."
images: ["/images/capa-criando-e-configurando-uma-role-iam.png"]
author: Jason
date: 2024-03-30
tags: [ "AWS", "Segurança" ]
type: post
showTableOfContents: true
---

O objetivo desse artigo é criar um tutorial demonstrando como criar e
configurar uma role IAM na AWS para que você possa interagir de forma
programática com a AWS. Usar uma role IAM é mais seguro do que usar
diretamente um usuário IAM, como mostrei em [Criando e configurando um
usuário
IAM](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/).

## Pré-requisitos 

Abaixo os pré-requisitos para seguir esse tutorial:

-   Conta AWS
-   Uma máquina Linux
-   Um usuário IAM sem nenhuma permissão
-   AWS CLI instalado

## Criação usuário IAM 

Como primeiro passo você deve acessar sua conta AWS e acessar o serviço
**IAM** e ir na opção **Users** e depois clicar na opção **Create
User**. Será apresentada a tela abaixo:

![Tela Specify user details da criação do usuário IAM blog](/images/iam-14.webp)

Não habilite a opção **Provide user access to the AWS Management
Console**, pois ela libera o acesso à console de administração da AWS.

Insira um nome para o usuário IAM e clicar em **Next**.

Na próxima tela, não adicione nenhuma permissão a esse usuário IAM: a
permissão será concedida pela role IAM, por meio de uma política IAM.

Depois disso clicar na opção **Create User** e seu usuário será criado.

Para saber como gerar o **Access Key** e o **Secret Access Key** e como
configurar veja o [post sobre o usuário
IAM](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/).

## Criando uma Role IAM

Para demonstrar a utilização de uma Role iremos criar duas Roles IAM
conforme abaixo:

-   Uma Role IAM com permissão de listar buckets no S3
-   Uma Role IAM com permissão para parar, iniciar e reiniciar uma instância EC2

Antes de criar as roles vamos criar duas políticas no IAM, uma para cada role.

Para isso, acesse a console de administração da AWS, vá no serviço
**IAM** e depois em **Políticas** e então clicar no
botão **Criar política**. Após clicar no botão **Criar política**, será
apresentada a tela abaixo:

![Editor de políticas do IAM para criar uma policy](/images/iam-15.webp)

Do lado direito da tela acima clicar na opção JSON e substituir o
conteúdo conforme abaixo:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        }
    ]
}
```

Clique no botão Próximo e dê um nome para a política. No meu caso, o
nome será **s3-list** e depois clicar no botão **Criar
política**.

Deverá ser criada outra política com o conteúdo abaixo com o nome de
**ec2-reboot**.

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ec2:RebootInstances",
                "ec2:StartInstances",
                "ec2:StopInstances"
            ],
            "Resource": "*"
        }
    ]
}
```

Agora vamos criar duas roles, uma para cada política criada. Para isso
você deve ir no serviço **IAM** e depois na opção **Funções** e então
clicar no botão **Criar perfil**. Deverá aparecer a tela abaixo:

![Criação da role com o tipo de entidade confiável Conta da AWS, usando esta conta](/images/iam-16.webp)

Você deve selecionar a opção **Conta da AWS** e clicar em Próximo. Após
isso selecionar a policy **s3-list** criada anteriormente e clicar em
Próximo. Será apresentada a tela abaixo:

![Tela Nomear, revisar e criar da role s3 com a política de confiança para sts:AssumeRole](/images/iam-17.webp)

Insira o nome do perfil como **s3** e clicar no botão **Criar
perfil**. Agora vamos precisar editar o conteúdo da política de
confiança do perfil criado na tela abaixo:

![Resumo da role s3 com a política de confiança que permite ao usuário blog assumir a role](/images/iam-18.webp)

Agora vamos substituir o conteúdo da política de confiança conforme
abaixo:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::id_conta:user/blog"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

Onde id_conta é o número da sua conta AWS. Com isso finalizamos a
configuração da role **s3**. Você deve fazer o mesmo procedimento para
role **ec2** e inserir a política **ec2-reboot** nessa role.

## Configurando as roles na CLI 

Como pré-requisito o usuário criado deve estar configurado na sua
máquina usando o aws cli. Lembrando que esse usuário não pode ter nenhum tipo de acesso na AWS.

Agora vamos configurar o acesso desse usuário nas roles **s3** e
**ec2**. Devem ser adicionados ao arquivo .aws/config os seguintes profiles:

``` bash
[profile s3]
role_arn = arn:aws:iam::id_conta:role/s3
source_profile = blog

[profile ec2]
role_arn = arn:aws:iam::id_conta:role/ec2
source_profile = blog
```

Onde id_conta é o número da sua conta da AWS.

## Testando as roles 

Agora vamos testar as roles criadas. Para esses testes, criei um bucket
S3 e uma instância EC2. Primeiramente, vamos verificar se o usuário
criado tem alguma permissão. No meu caso, o usuário IAM se chama blog.

![Erros de acesso negado ao tentar parar uma instância EC2 e listar buckets S3 com o usuário blog, com o ID da conta censurado](/images/iam-19.webp)

Devem ser apresentados os erros acima para o usuário blog. Agora vamos
testar a role s3 usando o comando abaixo:

``` bash
aws s3 ls --profile s3
```

Deve ser apresentada a tela abaixo, listando um bucket S3:

![Saída do aws s3 ls --profile s3 listando o bucket teste-blog com a role assumida](/images/iam-20.webp)

Agora vamos para o segundo teste onde vamos parar uma instância ec2
usando a role ec2. Para isso vamos executar o comando abaixo:

``` bash
aws ec2 stop-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile ec2
```

Executando esse comando acima aparecerá uma tela de mudança de estado da
instância ec2 de running para stopping conforme imagem abaixo:

![Saída do aws ec2 stop-instances com a instância passando de running para stopping](/images/iam-21.webp)

Assim que a instância parar você pode executar o comando abaixo para
inicializar a mesma instância ec2.

``` bash
aws ec2 start-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile ec2
```

Você também pode testar o desligamento da instância ec2 executando o
comando abaixo. Será apresentado um erro, pois essa role não tem essa permissão.

``` bash
aws ec2 terminate-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile ec2
```

Por fim, para garantir que a role s3 não consegue iniciar e parar uma
instância ec2 executar um dos comandos abaixo:

``` bash
aws ec2 stop-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile s3

aws ec2 start-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile s3
```

## Conclusão 

O objetivo desse artigo foi demonstrar a criação de uma role e efetuar
testes usando as roles em vez de utilizar o usuário IAM. Nesse artigo
demonstrei a utilização de uma role usando apenas uma conta AWS porém é
possível utilizar múltiplas contas, conforme o [tutorial de acesso entre
contas com
roles](https://docs.aws.amazon.com/pt_br/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)
da AWS.

Para ir além das roles assumidas a partir de um usuário IAM, veja como
acessar a AWS com credenciais temporárias e MFA em
[Acesso à AWS sem access keys com o IAM Identity Center](/posts/2026/2026-10-31-acesso-a-aws-sem-access-keys-com-iam-identity-center/),
ou como acessar instâncias EC2 sem chave SSH em
[Acessando instâncias EC2 com o Session Manager, sem SSH](/posts/2025/2025-04-12-acessando-instancias-ec2-com-session-manager-sem-ssh/).