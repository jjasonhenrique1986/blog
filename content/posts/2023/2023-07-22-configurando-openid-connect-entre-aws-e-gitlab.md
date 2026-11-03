---
title: "Configurando OpenID Connect entre AWS e GitLab"
description: "Como configurar OpenID Connect entre a AWS e o GitLab para executar pipelines usando uma role IAM, sem chaves de acesso fixas."
image: "/images/gitlab.png"
author: Jason
date: 2023-07-22
tags: [ "AWS", "CI/CD" ]
type: post
showTableOfContents: true
---

![Configurando OpenID Connect entre AWS e GitLab](/images/gitlab.png)

O objetivo desse post é demonstrar como configurar o OpenID Connect
entre a sua conta AWS e sua conta do GitLab para executar uma pipeline
usando uma role da sua conta AWS.

Primeiramente, o OpenID Connect (OIDC) permite que você obtenha tokens
de curta duração para se comunicar com o seu cloud provider. Neste caso
vamos utilizar a AWS, mas o mesmo conceito também pode ser utilizado
para Azure e GCP.

A grande vantagem dessa integração entre AWS e GitLab usando OpenID
Connect é a segurança: é gerado um token de curta duração, e não é
necessário criar um **Access Key** e de um **Secret Access
Key**. Será utilizado o conceito de role da AWS e essa role será
executada apenas em um determinado repositório do GitLab.

## Pré-requisitos

Para prosseguir com esse tutorial é necessário ter os seguintes
pré-requisitos:

-   Conta na AWS;
-   Conta no [GitLab.com](https://about.gitlab.com/) (SaaS);

## Configuração OIDC

No primeiro passo vamos configurar o OIDC na AWS, mais especificamente
no IAM. Acesse o serviço **IAM** e a opção **Identity providers**. Deve
aparecer a tela abaixo:

![Tela Identity providers do IAM ainda sem nenhum provedor cadastrado](/images/gitlab-2.webp)

Clicando na opção **Add provider**, será exibida a tela abaixo, onde
vamos iniciar a configuração do OIDC:

![Tela Add an Identity provider do IAM com as opções SAML e OpenID Connect](/images/gitlab-3.webp)

Selecione a opção OpenID Connect. Nos campos **Provider URL** e
**Audience**, configure a URL <https://gitlab.com> conforme
imagem abaixo:

![Cadastro do provedor OpenID Connect com a URL e o audience https://gitlab.com](/images/gitlab-4.webp)

Você deve clicar na opção **Get thumbprint** onde será gerado um
certificado thumbprint que valida a CA do OIDC Provider (no nosso caso,
o GitLab). Depois, clique em **Add provider** para finalizar a
configuração do OIDC. Será criado um provider conforme
imagem abaixo:

![Provedor gitlab.com do tipo OpenID Connect criado na lista de Identity providers do IAM](/images/gitlab-5.webp)

## Criação Role AWS

No segundo passo vamos criar uma role na AWS indo na opção **IAM** e
depois **Roles** (Funções em português) e depois na opção **Create Role**
deverá aparecer uma tela similar à abaixo:

![Tela Select trusted entity da criação de role no IAM](/images/gitlab-6.webp)

Selecione a opção **Web Identity** e selecionar o **Identity
provider** e o **Audience** criado no passo anterior conforme abaixo:

![Criação da role com o tipo Web identity, o provedor gitlab.com e o audience https://gitlab.com](/images/gitlab-7.webp)

Depois vamos para o step 2 onde vamos selecionar as permissões dessa
role. Neste caso vamos usar a política
[AmazonS3ReadOnlyAccess](https://us-east-1.console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess)
para apenas listar os buckets S3 de uma conta na AWS.

Depois vamos para o step 3 onde vamos apenas colocar o nome da Role e
clicar na opção **Create role**.

Depois da criação da Role vamos editar a relação de confiança dessa role
para ela ser menos permissiva. Abaixo segue a trust policy dessa role:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::aws_account_id:oidc-provider/gitlab.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "gitlab.com:sub": "project_path:user_gitlab/name_repository:ref_type:branch:ref:main"
                }
            }
        }
    ]
}
```

Onde:

-   **aws_account_id** é o id da sua conta da AWS;
-   **user_gitlab** é o nome do usuário utilizado no gitlab. Também
    aceita como opção um projeto do GitLab;
-   **name_repository** é o nome do repository criado no gitlab;

## Configuração GitLab

No terceiro passo vamos criar um repositório no gitlab, criar uma
pipeline simples que assuma a role criada no passo anterior e liste os
buckets S3 da conta AWS.

Criar um repositório no gitlab e adicionar a seguinte variável de
ambiente indo na opção Settings \>\> CI/CD \>\> Variables. Deverá
mostrar uma tela similar a abaixo:

![Variável ROLE_ARN cadastrada como mascarada nas variáveis de CI/CD do GitLab](/images/gitlab-8.webp)

onde:

-   **aws_account_id** é o id da sua conta da AWS;
-   **name_role** é o nome da role criada na AWS;

Esse valor também pode ser copiado diretamente da role criada na AWS, no
campo ARN.

Após isso criar o arquivo **.gitlab-ci.yml** com o conteúdo abaixo:

``` bash
image:
  name: "amazon/aws-cli:latest"
  entrypoint:
    - ""

stages:
  - "assume-role"

job_assume_role:
    stage: "assume-role"
    script: 
      - >
        STS=($(aws sts assume-role-with-web-identity
        --role-arn ${ROLE_ARN}
        --role-session-name "GitLabRunner-${CI_PROJECT_ID}-${CI_PIPELINE_ID}"
        --web-identity-token ${CI_JOB_JWT_V2}
        --duration-seconds 3600
        --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]'
        --output text))
      - export AWS_ACCESS_KEY_ID="${STS[0]}"
      - export AWS_SECRET_ACCESS_KEY="${STS[1]}"
      - export AWS_SESSION_TOKEN="${STS[2]}"
      - aws sts get-caller-identity
      - aws s3 ls
```

Com isso, será executada uma pipeline simples que lista os buckets S3 da
conta AWS.

## Conclusão

O objetivo desse post foi demonstrar a configuração do OpenID Connect
entre AWS e GitLab. Esse mesmo procedimento pode ser feito com o GitHub,
conforme a [documentação do GitHub sobre OIDC na
AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services),
e mostrei o passo a passo em
[GitHub Actions com OpenID Connect na AWS, sem access keys](/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys/).
