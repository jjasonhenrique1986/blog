---
title: "Criando e Configurando um usuário IAM"
description: "Tutorial para criar e configurar um usuário IAM com Access Key e Secret Access Key e instalar o AWS CLI."
image: "/images/iam.png"
author: Jason
date: 2024-01-07
tags: [ "AWS", "Segurança" ]
type: post
showTableOfContents: true
---

![Criando e Configurando um usuário IAM](/images/iam.png)

> **Atualização (novembro de 2026):** usuário IAM com access key fixa não
> é mais a forma recomendada de acessar a AWS pela linha de comando, porque
> a chave não expira e pode vazar. Para pessoas, prefira o IAM Identity
> Center, que entrega credenciais temporárias com MFA: veja
> [Acesso à AWS sem access keys com o IAM Identity Center](/posts/2026/2026-10-31-acesso-a-aws-sem-access-keys-com-iam-identity-center/).
> Para pipelines, use OpenID Connect, como em
> [GitHub Actions com OpenID Connect na AWS, sem access keys](/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys/).
> O passo a passo abaixo continua válido para entender o IAM.

O objetivo desse artigo é criar um tutorial demonstrando como criar e
configurar um usuário IAM na AWS para que você possa interagir de forma
programatica usando um **Access Key** e um **Secret Access Key**.

Também será demosntrado a instalação do awscli para o gereciamento dos
seus serviços na AWS.

## Pré Requisitos 

Abaixo os pré requisitos para seguir esse tutorial:

-   Conta AWS
-   Uma máquina Linux

## Criação usuário IAM 

Como primeiro passo você deve acessar sua conta AWS e acessar o serviço
**IAM** e ir na opção **Users** e depois clicar na opção **Create User**
será apresentado a seguinte tela abaixo:

![Tela Specify user details da criação do usuário IAM blog, sem acesso ao console](/images/iam-2.webp)

Não deve ser habiltado a opção **Provide user access to the AWS
Management Console** pois será habilitado acesso a console de
administração da AWS.

Deve ser inserido um nome para nosso usuário IAM e clicar em **Next**.

Na próxima tela vamos inserir as permissões para esse usuário IAM. No
meu caso vou inserir uma policy com permissão total no S3 conforme tela
abaixo:

![Tela Set permissions com a policy AmazonS3FullAccess anexada diretamente ao usuário](/images/iam-3.webp)

Lembrando que você pode customizar essa policy.

Depois disso vamos clicar em Next e será apresentado um Review da
criação do seu usuário conforme abaixo:

![Tela Review and create do usuário blog com a policy AmazonS3FullAccess](/images/iam-4.webp)

Depois disso clicar na opção **Create User** e seu usuário será
criado.Seu usuário deve ser apresentado conforme tela abaixo:

![Mensagem de usuário criado com sucesso e o usuário blog na lista do IAM](/images/iam-5.webp)

## Criando a Acess Key do seu usuário 

Agora vamos criar clicando sobre seu usuário IAM indo na opção Security
credentials e depois clicando em Create access key conforme imagem
abaixo:

![Aba Security credentials do usuário com a seção Access keys ainda vazia](/images/iam-6.webp)

Depois da tela você deve escolher a primeira opção **Command Line
Interface (CLI)** e habilitar a opção **I understand the above
recommendation and want to proceed to create an access key** e clicar em
Next conforme imagem abaixo:

![Tela Access key best practices & alternatives com o caso de uso Command Line Interface (CLI) selecionado](/images/iam-7.webp)

Na próxima tela você pode configurar uma descrição para sua access key e
depois clicar em **Create access key** conforme imagem abaixo:

![Tag de descrição da access key preenchida com Para post no blog](/images/iam-8.webp)

Apos clicar no botão **Create access key** será gerado o **Acces key** e
o **Secret access key** conforme imagem abaixo:

![Tela Retrieve access keys com a access key criada, com o ID censurado](/images/iam-9.webp)

Lembrando que o **Secret access key** só estará disponivel nesse
momento. É recomendavel fazer o download do arquivo cvs com os valores
do **Access key** e do **Secret access key** e depois clicar em **Done**
e retornará para a tela inicial conforme abaixo:

![Access key ativa na aba Security credentials do usuário, com o ID censurado](/images/iam-10.webp)

Nesse
[link](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#securing_access-keys)
você tem boas práticas para o gerenciamento de AWS access keys na AWS.

## Instalação AWS CLI 

Agora vamos fazer a instalação do aws cli para podemos configurar o
**Access key** e o **Secret access key** gerados no passo anterior.
Abaixo os comandos para instalação. Para mais informações consultar esse
[link](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/getting-started-install.html).

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Para verificar se a instalação ocorreu com sucesso excutar o comando
abaixo:

![Saída do comando aws --version mostrando o AWS CLI 2.13.34](/images/iam-11.webp)

## Configurando o seu Access Key 

Agora vamos fazer a configuração do seu Access key e do seu Secret
access key no seu computador. Você deve executar o comando aws
configure conforme abaixo. No meu caso vou inserir a flag --profile para
gerar um novo profile dentro do arquivo .aws/credentials

![Comando aws configure --profile blog com a access key e a secret censuradas e a região sa-east-1](/images/iam-12.webp)

## Testando o seu access Key

Agora vamos testar o acesso via CLI tentando listar os buckets S3 na sua
conta executando o comando abaixo:

```bash
aws s3 ls --profile blog
```

Deverá retornar o seguinte resultado abaixo:

![Saída do aws s3 ls --profile blog listando o bucket teste-blog](/images/iam-13.png)

Lembrando que esse Access key e Secret access key tem acesso somente ao
S3. Caso tente executar o comando **aws ec2 describe-availability-zones
--profile blog** deverá retornar o seguinte erro abaixo por falta de
permissão:

``` bash
An error occurred (UnauthorizedOperation) when calling the DescribeAvailabilityZones operation: You are not authorized to perform this operation. User: arn:aws:iam::id_conta:user/blog is not authorized to perform: ec2:DescribeAvailabilityZones because no identity-based policy allows the ec2:DescribeAvailabilityZones action
```

## Conclusão 

O objetivo desse artigo foi a criação de um tutorial para a criação e
configuração de um usuário IAM usando o Access key e o Secret access key
para interagir via CLI com a sua conta AWS.

Lembrando que o Access key e o Secret access key não estará disponivel
para uso. Você deve gerar os seus próprios valores na sua conta caso
queira seguir com esse artigo.