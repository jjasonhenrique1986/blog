---
title: "Usando o AWS App Runner para subir uma aplicação"
description: "Como usar o AWS App Runner para subir uma aplicação a partir de uma imagem pública do ECR."
images: ["/images/capa-usando-o-aws-app-runner-para-subir-uma-aplicacao.png"]
author: Jason
date: 2023-12-02
tags: [ "AWS", "Containers" ]
type: post
---

O objetivo desse artigo é demonstrar a utilização do serviço AWS App
Runner. Vamos subir uma aplicação usando uma imagem pública do ECR para
testar o serviço.

## O que é AWS App Runner 

AWS App Runner é um serviço da AWS que permite implantar e gerenciar
aplicações web e APIs sem precisar gerenciar a infraestrutura, como load
balancer e autoscaling. Com ele, é possível subir uma aplicação
rapidamente.

Para mais informações, veja a [página do AWS App
Runner](https://aws.amazon.com/pt/apprunner/).

## Aplicação usando um ECR Público 

Vamos subir essa aplicação que está no ECR público: a imagem
[hello-app-runner](https://gallery.ecr.aws/aws-containers/hello-app-runner).

Acesse a sua conta da AWS, procure pelo serviço **AWS App Runner** e
clique na opção **Create an App Runner Service**. Será apresentada a
tela abaixo:

![Tela Source and deployment do App Runner com as opções Container registry e Amazon ECR](/images/apprunner-2.webp)

Podemos usar o AWS App Runner a partir de um Container Registry, sendo o
ECR o único suportado. Você tem a opção de usar um ECR
público ou um ECR privado. Abaixo seguem as configurações utilizadas
para subir uma aplicação usando um ECR público:

![Configuração da origem com Amazon ECR Public, a imagem hello-app-runner e deploy manual](/images/apprunner-3.webp)

Na próxima tela vamos configurar o nome do serviço, a quantidade de CPU
e memória que serão utilizados, a porta e, se necessário, variáveis de ambiente.

![Configuração do serviço AppECR com 0,5 vCPU, 1 GB de memória e porta 8000](/images/apprunner-4.webp)

Na próxima tela vamos configurar o auto scaling da aplicação. Por ser
apenas um teste, vamos usar as definições padrão, mas você pode criar as
suas próprias clicando no botão Create.

![Configuração de auto scaling padrão do App Runner: concorrência de 100 requisições, de 1 a 25 instâncias](/images/apprunner-5.webp)

Na próxima tela, faça as configurações de Health Check conforme abaixo:

![Configuração do health check HTTP do App Runner no path /](/images/apprunner-6.webp)

Também podemos fazer as configurações de security, networking,
observability e tags. Na próxima tela será feito um review das
configurações da sua aplicação e depois você deve clicar no botão
**Create & Deploy**

Na próxima tela podemos acompanhar o progresso do Deploy conforme
abaixo:

![Logs de eventos do App Runner mostrando o deploy da imagem e o health check bem-sucedido](/images/apprunner-7.webp)

Você também pode verificar os logs da aplicação conforme imagem abaixo:

![Logs da aplicação no App Runner com as requisições GET respondidas com status 200](/images/apprunner-8.webp)

Para testar a aplicação basta clicar no endereço que será gerado em
Default Domain conforme imagem abaixo:

![Service overview do AppECR com status Running e o default domain do App Runner](/images/apprunner-9.webp)

Deve aparecer uma imagem conforme abaixo:

![Página da aplicação de exemplo do App Runner com a mensagem And we're live!](/images/apprunner-10.webp)

## Conclusão 

O objetivo desse artigo foi demonstrar a utilização do AWS App Runner
usando o método de Container Registry, fazendo o deploy de uma aplicação
a partir de um ECR público. No artigo seguinte demonstrei a utilização
do AWS App
Runner usando uma API localizada no GitHub:
[Deployando uma API usando AWS App Runner](/posts/2023/2023-12-23-deployando-uma-api-usando-aws-app-runner/).
