---
title: "Deployando uma API usando AWS App Runner"
description: "Segundo artigo sobre o AWS App Runner: deploy de uma API Hello World feita com FastAPI."
image: "/images/apprunner.jpg"
author: Jason
date: 2023-12-23
tags: [ "AWS", "Containers" ]
type: post
showTableOfContents: true
---

![Deployando uma API usando AWS App Runner](/images/apprunner.jpg)

Nesse segundo artigo falando sobre o App Runner vou demonstrar a criação
de uma API feita com FastAPI que retorna um Hello World. O código fonte
desse API está localizado no Github no seguinte
[repositório](https://github.com/jjasonhenrique/apprunner-example.git).

Para conferir meu primeiro artigo falando sobre o AWS App Runner basta
acessar o seguinte
[link](/posts/2023/2023-12-02-usando-o-aws-app-runner-para-subir-uma-aplicacao/).

## Pré Requisitos 

Abaixo os pré requisitos necessários para seguir esse artigo:

-   Conta AWS
-   Um repositório no Github com o código fonte informado acima.

## Configurando sua conta do Github 

Como primeiro passo devemos conectar a sua conta do Github com a sua
conta da AWS. Para isso devemos ir em **App Runner** \>\> **Connected
Accounts** e deve aparecer a tela abaixo:

![Tela Connected accounts do App Runner ainda sem conexões](/images/apprunner-11.jpg)

Agora devemos clicar na opção **Add New** selecionar o provider no meu
caso será Github mas existe a possibilidade de usar o Bitbucket também.
Você será redirecionado para uma tela para fazer login na sua conta do
Github para a configuração do **AWS Connector for Gihub** conforme
abaixo:

![Tela de login do GitHub para autorizar o AWS Connector for GitHub](/images/apprunner-12.jpg)

Você deve inserir o nome da sua conexão no meu caso será **GitHub** e
clicar no botão **Install another** conforme imagem abaixo:

![Criação de uma nova conexão chamada Github no App Runner](/images/apprunner-13.jpg)

Na próxima tela você deve configurar apenas o repositório que o código
fonte da nossa API está localizado conforme imagem abaixo:

![Instalação do AWS Connector for GitHub com acesso apenas ao repositório apprunner-example](/images/apprunner-14.jpg)

Você deve clicar no botão **Install** e com isso a sua conexão com o
github será efetuada com sucesso e retornará para a página abaixo:

![Conexão Github com o GitHub app da conta selecionado](/images/apprunner-15.jpg)

Agora você deve clicar no botão **Next** para finalizar a configuração
na AWS e deve aparecer uma tela similar abaixo informando sobre o
sucesso da conexão entre a conta AWS com a sua conta do Github.

![Conexão GitHub com status Available em Connected accounts](/images/apprunner-16.jpg)

## Fazendo o deploy da API 

Agora vamos fazer o deploy da nossa API usando o AWS App Runner. Você
deve procurar o serviço AWS App Runner depois ir na opção **Create an
App Runner Service** e depois selecionar o metódo **Source Code
Repository** deve ser carregado uma tela conforme abaixo:

![Origem do serviço configurada como repositório de código do GitHub, repositório apprunner-example e branch main](/images/apprunner-17.jpg)

Foi carregado automaticamente as informações do repositório onde está o
código fonte da nossa API. Agora basta clicar em Next e vamos configurar
as opções de build conforme imagem abaixo:

![Configuração do build com runtime Python 3, comando pip install -r requirements.txt, start com uvicorn e porta 8080](/images/apprunner-18.jpg)

Na próxima tela vamos configurar o nome do serviço, utilização de CPU e
memória, configuração de Autoscaling, configuração de Health Check e
outros. Para mais informações sobre essas configurações consultar o
primeiro artigo sobre o AWS App Runner nesse
[link](https://jjasonhenrique.me/2023/12/02/usando-o-aws-app-runner-para-subir-uma-aplicacao/).

![Configuração do serviço API-Python com 0,25 vCPU e 0,5 GB de memória](/images/apprunner-19.jpg)

Na opção de Health Check alterar o protocolo para HTTP conforme abaixo:

![Configuração do health check HTTP do serviço API-Python no path /](/images/apprunner-20.jpg)

Depois vamos avançar para a próxima tela e clicar na opção **Create &
Deploy.** Na tela abaixo podemos verificar que a API foi implementada
com sucesso.

![Logs de eventos do App Runner com o build do código do GitHub e o deploy concluído com sucesso](/images/apprunner-21.jpg)

Para acessar a API basta acessar o link informado no campo **Default
Domain** deve ser apresentado uma tela conforme a imagem abaixo:

![Navegador acessando o domínio do App Runner e exibindo Hello World](/images/apprunner-22.jpg)

## Conclusão 

O intuito desse artigo vou demonstrar o deploy de uma simples API
localizada no Github executando no serviço AWS App Runner.

