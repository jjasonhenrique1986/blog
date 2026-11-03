---
title: "Deployando uma API usando AWS App Runner"
description: "Segundo artigo sobre o AWS App Runner: deploy de uma API Hello World feita com FastAPI."
image: "/images/apprunner.png"
author: Jason
date: 2023-12-23
tags: [ "AWS", "Containers" ]
type: post
showTableOfContents: true
---

![Deployando uma API usando AWS App Runner](/images/apprunner.png)

Nesse segundo artigo falando sobre o App Runner vou demonstrar a criação
de uma API feita com FastAPI que retorna um Hello World. O código-fonte
dessa API está no repositório
[apprunner-example](https://github.com/jjasonhenrique/apprunner-example.git),
no GitHub.

O primeiro artigo sobre o App Runner é o [Usando o AWS App Runner para
subir uma
aplicação](/posts/2023/2023-12-02-usando-o-aws-app-runner-para-subir-uma-aplicacao/).

## Pré-requisitos 

Abaixo os pré-requisitos necessários para seguir esse artigo:

-   Conta AWS
-   Um repositório no GitHub com o código fonte informado acima.

## Configurando sua conta do GitHub 

Como primeiro passo devemos conectar a sua conta do GitHub com a sua
conta da AWS. Para isso devemos ir em **App Runner** \>\> **Connected
Accounts** e deve aparecer a tela abaixo:

![Tela Connected accounts do App Runner ainda sem conexões](/images/apprunner-11.webp)

Agora devemos clicar na opção **Add New** e selecionar o provider. No
meu caso será o GitHub, mas existe a possibilidade de usar o Bitbucket
também.
Você será redirecionado para uma tela para fazer login na sua conta do
GitHub para a configuração do **AWS Connector for GitHub**, conforme abaixo:

![Tela de login do GitHub para autorizar o AWS Connector for GitHub](/images/apprunner-12.webp)

Insira o nome da sua conexão (no meu caso, **GitHub**) e clique no botão
**Install another** conforme imagem abaixo:

![Criação de uma nova conexão chamada Github no App Runner](/images/apprunner-13.webp)

Na próxima tela você deve selecionar apenas o repositório em que está o
código-fonte da nossa API, conforme imagem abaixo:

![Instalação do AWS Connector for GitHub com acesso apenas ao repositório apprunner-example](/images/apprunner-14.webp)

Você deve clicar no botão **Install** e com isso a sua conexão com o
github será efetuada com sucesso e retornará para a página abaixo:

![Conexão Github com o GitHub app da conta selecionado](/images/apprunner-15.webp)

Agora você deve clicar no botão **Next** para finalizar a configuração
na AWS e deve aparecer uma tela similar à abaixo informando sobre o
sucesso da conexão entre a conta AWS com a sua conta do GitHub.

![Conexão GitHub com status Available em Connected accounts](/images/apprunner-16.webp)

## Fazendo o deploy da API 

Agora vamos fazer o deploy da nossa API usando o AWS App Runner. Procure
o serviço AWS App Runner, vá na opção **Create an App Runner Service** e
selecione o método **Source Code Repository**. Deve ser carregada uma
tela conforme abaixo:

![Origem do serviço configurada como repositório de código do GitHub, repositório apprunner-example e branch main](/images/apprunner-17.webp)

Foram carregadas automaticamente as informações do repositório onde está
o código-fonte da nossa API. Agora basta clicar em Next e vamos
configurar
as opções de build conforme imagem abaixo:

![Configuração do build com runtime Python 3, comando pip install -r requirements.txt, start com uvicorn e porta 8080](/images/apprunner-18.webp)

Na próxima tela vamos configurar o nome do serviço, utilização de CPU e
memória, configuração de Autoscaling, configuração de Health Check e
outros. Para mais informações sobre essas configurações consulte o
[primeiro artigo sobre o AWS App
Runner](/posts/2023/2023-12-02-usando-o-aws-app-runner-para-subir-uma-aplicacao/).

![Configuração do serviço API-Python com 0,25 vCPU e 0,5 GB de memória](/images/apprunner-19.webp)

Na opção de Health Check alterar o protocolo para HTTP conforme abaixo:

![Configuração do health check HTTP do serviço API-Python no path /](/images/apprunner-20.webp)

Depois vamos avançar para a próxima tela e clicar na opção **Create &
Deploy.** Na tela abaixo podemos verificar que a API foi implementada
com sucesso.

![Logs de eventos do App Runner com o build do código do GitHub e o deploy concluído com sucesso](/images/apprunner-21.webp)

Para acessar a API basta acessar o link informado no campo **Default
Domain**. Deve ser apresentada uma tela conforme a imagem abaixo:

![Navegador acessando o domínio do App Runner e exibindo Hello World](/images/apprunner-22.webp)

## Conclusão 

O intuito desse artigo foi demonstrar o deploy de uma simples API
localizada no GitHub executando no serviço AWS App Runner.

