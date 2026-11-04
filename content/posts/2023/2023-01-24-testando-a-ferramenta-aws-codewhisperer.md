---
title: "Testando a ferramenta AWS CodeWhisperer"
description: "Testando o AWS CodeWhisperer, ferramenta da AWS que sugere código usando Machine Learning, similar ao GitHub Copilot."
images: ["/images/capa-testando-a-ferramenta-aws-codewhisperer.png"]
author: Jason
date: 2023-01-24
tags: [ "AWS", "Segurança", "IA" ]
type: post
---

> **Atualização (novembro de 2026):** em abril de 2024 o CodeWhisperer foi
> incorporado ao **Amazon Q Developer**, e o nome CodeWhisperer deixou de
> ser usado. As listas de linguagens e IDEs abaixo são as da época do post.

O objetivo desse post é demonstrar a utilização do AWS CodeWhisperer,
uma ferramenta que sugere código para um
desenvolvedor usando Machine Learning e Inteligência Artificial. Essa
ferramenta é similar ao GitHub Copilot.

Essa ferramenta suporta as seguintes linguagens:

-   C#
-   Java
-   JavaScript
-   Python
-   TypeScript

E se integra com as seguintes IDEs:

-   IntelliJ IDEA
-   PyCharm
-   VS Code
-   AWS Cloud9
-   AWS Lambda

Para essa demonstração irei utilizar o VS Code. Para mais informações,
veja a [página do CodeWhisperer na
AWS](https://aws.amazon.com/pt/codewhisperer/).

O serviço também permite fazer um scan de segurança do seu código.

## Pré-requisitos 

Para testar esse serviço da AWS você deve ter os seguintes
pré-requisitos:

-   VS Code instalado. Faça o download na [página de download do VS
    Code](https://code.visualstudio.com/download).
-   Instalar a extensão AWS Toolkit no VS Code conforme imagem abaixo:

![Extensão AWS Toolkit no marketplace do VS Code](/images/codewhisperer-2.webp)

## Instalação e Configuração 

Após a instalação da extensão AWS Toolkit, aparecerá um ícone da AWS no
lado esquerdo, similar à imagem abaixo:

![Painel Developer Tools do AWS Toolkit conectado com AWS Builder ID e as opções do CodeWhisperer](/images/codewhisperer-3.png)

Para ativar a utilização do serviço AWS CodeWhisperer será necessário
configurar a conexão com a AWS.

Eu utilizei a opção **"Use a personal email to sign up and sign in with
AWS Builder ID"**. Ao clicar nessa opção, uma página web é aberta para
efetuar a criação do seu AWS Builder ID.

![Opções de conexão do AWS Toolkit: AWS Builder ID, IAM Identity Center ou credenciais IAM](/images/codewhisperer-4.png)

## Demonstração 

Para a demonstração eu criei um arquivo com a extensão .py e adicionei
um comentário sobre o que gostaria de criar, por exemplo "Create simple
API with fastAPI". Com isso, ele vai dando sugestões de código, conforme
a imagem abaixo:

![VS Code com o CodeWhisperer sugerindo o código de uma API FastAPI a partir de um comentário](/images/codewhisperer-5.webp)

Esse foi o resultado final apresentado pelo AWS CodeWhisperer

![Código de uma API FastAPI com as rotas / e /about gerado com as sugestões do CodeWhisperer](/images/codewhisperer-6.png)

## Conclusão 

Achei bem legal esse serviço da AWS. Ele pode ser uma alternativa bem
interessante ao GitHub Copilot.