---
title: "Testando a ferramenta AWS Copilot CLI"
description: "Testando o AWS Copilot CLI para criar de forma simplificada uma aplicação conteinerizada executando no ECS."
image: "/images/copilot.png"
author: Jason
date: 2024-01-20
tags: [ "AWS", "Containers", "ECS" ]
type: post
showTableOfContents: true
---

![Testando a ferramenta AWS Copilot CLI](/images/copilot.png)

> **Atualização (novembro de 2026):** a AWS encerrou o suporte ao Copilot
> CLI em 12 de junho de 2026, e o
> [repositório](https://github.com/aws/copilot-cli) foi arquivado. A
> ferramenta não recebe mais atualizações nem correções de segurança, então
> não a use em projetos novos. O post fica como registro.

Nesse artigo vou demonstrar como usar a ferramenta AWS Copilot CLI para
criar, de forma simplificada, uma aplicação conteinerizada executando no
ECS. Com o AWS Copilot CLI também é possível gerenciar
toda a infraestrutura da aplicação, como a infraestrutura como código, o
armazenamento, os segredos e a pipeline, tudo via linha de comando.

Para mais informações, veja a [documentação do AWS Copilot
CLI](https://aws.github.io/copilot-cli/docs/overview/).

## Pré-requisitos 

Abaixo os pré-requisitos para seguir com esse tutorial:

-   Ter uma conta AWS
-   AWS Copilot CLI instalado no seu computador
-   Ter um usuário IAM configurado na sua máquina

## Instalação 

Para instalar o AWS Copilot, execute os comandos abaixo:

``` bash
sudo curl -Lo /usr/local/bin/copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux \
   && sudo chmod +x /usr/local/bin/copilot \
   && copilot --help
```

Deve ser apresentada uma tela similar à abaixo:

![Saída do copilot -h com os comandos disponíveis do Copilot CLI](/images/copilot-2.webp)

No meu caso, utilizei um computador com Linux. Para outros sistemas
operacionais, consulte o [guia de instalação do
Copilot](https://aws.github.io/copilot-cli/docs/getting-started/install/).

## Configuração usuário AWS IAM 

Como pré-requisito para usar o AWS Copilot CLI, é necessário configurar
um usuário IAM no seu computador. Para isso, siga o post [Criando e
configurando um usuário
IAM](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/).

Para esse teste vou usar a policy **AdministratorAccess**. Caso utilize
um profile diferente do default no arquivo .aws/credentials será
necessário exportar a variável de ambiente **AWS_PROFILE** ou inserir
essa variável no arquivo .bashrc ou .zshrc. No meu caso vou exportar a
variável executando o comando abaixo:

``` bash
export AWS_PROFILE="blog"
```

Para verificar o conteúdo da variável executar o comando abaixo:

``` bash
echo $AWS_PROFILE
```

## Deploy da Aplicação 

Para testar o AWS Copilot CLI vamos fazer o deploy da aplicação de
exemplo
[amazon-ecs-cli-sample-app](https://github.com/aws-samples/amazon-ecs-cli-sample-app.git).
O primeiro passo é fazer o git clone desse repositório
usando o comando abaixo:

``` bash
git clone https://github.com/aws-samples/amazon-ecs-cli-sample-app.git example 
```

Depois, acesse a pasta example e execute o comando **copilot init**.
Será apresentada a tela abaixo:

![Comando copilot init perguntando o nome da aplicação, preenchido com teste-app-blog](/images/copilot-3.webp)

Você deve inserir o nome da aplicação. No meu caso será teste-app-blog.

Na próxima tela, será solicitada a escolha do tipo de workload da sua
aplicação. Para essa demonstração iremos utilizar a opção **Load
Balanced Web Service** mas existem outras opções conforme abaixo:

![Lista de tipos de workload do Copilot, como Request-Driven Web Service, Load Balanced Web Service e Scheduled Job](/images/copilot-4.webp)

Agora você deve escolher um nome para seu serviço. No meu caso será
teste-app-blog-svc conforme abaixo:

![Copilot perguntando o nome do serviço, preenchido com teste-app-blog-svc](/images/copilot-5.webp)

Agora será solicitada a localização do arquivo Dockerfile ou a
localização de uma imagem da aplicação. Vamos utilizar o arquivo
Dockerfile que está na raiz do repositório.

![Copilot perguntando qual Dockerfile usar, com ./Dockerfile selecionado](/images/copilot-6.webp)

Nesse próximo passo, serão criadas todas as roles de infraestrutura
necessárias para o funcionamento da nossa aplicação. Após essa criação
será perguntado se você gostaria de fazer deploy da sua aplicação
conforme imagem abaixo:

![Copilot criando as roles de infraestrutura da aplicação e o manifest do serviço, e perguntando se deve fazer deploy de um environment](/images/copilot-7.webp)

Na próxima tela, será solicitada a criação de um environment. Esse
environment terá o nome de dev conforme imagem abaixo:

![Copilot perguntando o nome do environment, preenchido com dev](/images/copilot-8.webp)

Na próxima tela, será criada toda a infraestrutura, como o repositório
do ECR, um bucket S3, a parte de rede e o cluster ECS.

![Copilot criando o environment dev em sa-east-1 com VPC, subnets, Internet Gateway e cluster ECS](/images/copilot-9.webp)

Na próxima tela, o AWS Copilot CLI vai fazer o build da imagem da nossa
aplicação e fazer o push para o repositório ECR criado anteriormente
conforme abaixo:

![Build da imagem Docker do serviço e push para o repositório no ECR](/images/copilot-10.webp)

Na próxima tela a aplicação será configurada dentro do cluster ECS, será
feito o deploy e será disponibilizada a URL da aplicação, conforme a
imagem abaixo:

![Copilot criando o load balancer, a task definition e o serviço ECS e exibindo a URL do serviço](/images/copilot-11.webp)

Acessando a URL da aplicação deve retornar uma página similar a abaixo:

![Navegador acessando a URL do load balancer e exibindo a página com o logo do AWS Copilot](/images/copilot-12.webp)

## Interagindo com AWS Copilot CLI 

Com a aplicação já deployada no seu cluster ECS podemos usar o AWS
Copilot CLI para consultar informações da nossa aplicação. Abaixo,
alguns exemplos:

``` bash
copilot svc status
```

![Saída do copilot svc status com a task em execução e os health checks saudáveis](/images/copilot-13.webp)

``` bash
copilot svc ls
```

![Saída do copilot svc ls listando o serviço teste-app-blog-svc do tipo Load Balanced Web Service](/images/copilot-14.webp)

``` bash
copilot app show
```

![Saída do copilot app show com a aplicação teste-app-blog, o environment dev em sa-east-1 e o serviço](/images/copilot-15.webp)

Existem muitas outras opções que você pode utilizar com o AWS Copilot CLI.

## Deletando sua aplicação 

Agora vamos deletar a aplicação criada usando o comando abaixo:

``` bash
copilot app delete
```

Será solicitada uma confirmação e depois será feita a exclusão de todos
os recursos criados pelo AWS Copilot CLI, conforme imagem
abaixo:

![Saída do copilot app delete removendo o serviço, o environment e os recursos da aplicação](/images/copilot-16.webp)

## Conclusão 

O objetivo desse artigo foi demonstrar a utilização da ferramenta AWS
Copilot CLI implementando toda a infraestrutura e uma aplicação de
teste.