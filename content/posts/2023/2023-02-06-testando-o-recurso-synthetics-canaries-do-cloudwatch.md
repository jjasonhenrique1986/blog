---
title: "Testando o recurso Synthetics Canaries do CloudWatch"
description: "Como criar um Synthetics Canary no CloudWatch para checar periodicamente se uma URL ou API está respondendo com sucesso."
image: "/images/synthetic.png"
author: Jason
date: 2023-02-06
tags: [ "AWS", "Observabilidade" ]
type: post
showTableOfContents: true
---

![Testando o recurso Synthetics Canaries do Cloudwatch](/images/synthetic.png)

O objetivo desse post é demonstrar a criação de um synthetic canary na
AWS. É um recurso do CloudWatch que tem como principal função de testar
se uma URL ou uma API está
retornando status de sucesso (HTTP 200), com uma checagem a cada minuto,
por exemplo. Esse recurso existe em outras ferramentas de
observabilidade com que já trabalhei, como Dynatrace e Datadog.

Para mais informações, consulte a [documentação do CloudWatch
Synthetics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html).

## Pré-requisitos

Para testar esse serviço da AWS, foi criada uma API em Python usando o
FastAPI. Para saber mais informações sobre o que é o FastAPI e como
criar uma API usando FastAPI veja o [tutorial de FastAPI da Python
Academy](https://pythonacademy.com.br/blog/como-usar-o-fastapi-para-construir-apis-no-python)
e a [documentação oficial do FastAPI](https://fastapi.tiangolo.com/).

Para a instalação do FastAPI utilizarei uma EC2 com as seguintes
configurações abaixo:

![Resumo da criação da instância EC2 com Ubuntu 22.04 e tipo t2.micro](/images/synthetic-2.webp)

**Lembrando que essa EC2 deve possuir um IP público.**

## Instalação FastAPI

Após a criação da EC2 na AWS, devem ser executados os seguintes comandos
abaixo para a instalação do FastAPI:

``` bash
sudo apt update
sudo apt install python3-pip -y
pip install fastapi
pip install uvicorn
```

Para validar a instalação execute o comando abaixo:

``` bash
pip show fastapi
```

Deverá retornar algo conforme imagem abaixo:

![Saída do pip show fastapi mostrando a versão 0.89.1 instalada na instância](/images/synthetic-3.png)

Após a instalação do FastAPI devemos criar o arquivo **main.py**
conforme abaixo:

``` bash
from fastapi import FastAPI

app = FastAPI()

@app.get("/")

def index():
    return "Hello World"
```

Após salvar o arquivo acima, execute o comando abaixo para
iniciar a execução do FastAPI:

``` bash
uvicorn main:app --host 0.0.0.0 --port 8080
```

No meu caso estou usando a porta 8080. Será necessário liberar essa
porta no security group da EC2 conforme imagem abaixo:

![Regra do security group liberando a porta 8080 para 0.0.0.0/0](/images/synthetic-4.webp)

Utilizando o DNS público ou o IP público da sua EC2 na porta 8080 devemos
ter um resultado similar conforme imagem abaixo:

![Navegador acessando a API na porta 8080 da instância EC2 e exibindo Hello World](/images/synthetic-5.webp)

## Configuração Synthetics Canaries

Para continuar a configuração do recurso Synthetic Canaries do
CloudWatch é necessário já ter criado a API conforme seção acima.

Para iniciar a configuração você deve ir na opção **Application
Monitoring** \>\> **Synthetics Canaries** no serviço do **CloudWatch**
conforme imagem abaixo:

![Menu Application monitoring do CloudWatch com a opção Synthetics Canaries](/images/synthetic-6.webp)

Será mostrada uma tela conforme abaixo:

![Tela Synthetics Canaries do CloudWatch ainda sem nenhum canary criado](/images/synthetic-7.webp)

Para efetuar a criação basta clicar na opção **Create Canary** e será
apresentada a tela abaixo:

![Tela Create canary com a opção Use a blueprint e o blueprint Heartbeat monitoring selecionados](/images/synthetic-8.webp)

Nesta primeira seção temos a opção de usar padrões já criados pela AWS,
usar nossos próprios scripts e importar de um bucket S3. No meu caso vou
usar a primeira opção.

![Seção Canary builder com o nome helloworld e a URL da API na porta 8080](/images/synthetic-9.webp)

Nesta segunda seção você irá definir o nome do Canary e o endpoint URL
que será checado.

![Script editor do canary com o runtime syn-python-selenium-1.3 e o script Python gerado pelo blueprint](/images/synthetic-10.png)

Nesta terceira seção será escolhido o runtime que será executado. No meu
caso eu optei pelo runtime que utiliza o python3.8 + selenium + chromium
mas existem outras opções. Também é possível configurar variáveis de
ambiente para a execução do script.

![Agendamento do canary configurado para rodar continuamente a cada 1 minuto](/images/synthetic-11.webp)

Nesta quarta seção você irá configurar de quanto em quanto tempo será
executada a checagem do endpoint URL.

Outras opções que podem ser configuradas nessa mesma tela:

-   Tempo de retenção dos resultados;
-   Bucket onde serão salvos os resultados;
-   Criação de uma IAM Role;
-   Criação de um alarme no CloudWatch;
-   Permite que a checagem só execute dentro de uma determinada VPC;
-   Configuração de Tags;

Após a criação, será apresentado o dashboard abaixo:

![Mensagem de sucesso na criação do canary helloworld, ainda no estado Creating](/images/synthetic-12.webp)

![Canary helloworld executando com status Passed e 100% de sucesso](/images/synthetic-13.webp)


Parei a execução da API para verificar se o canary alerta a falha,
conforme imagem abaixo:

![Canary helloworld com status Failed depois que a API deixou de responder](/images/synthetic-14.webp)

Também é possível criar o Synthetics Canaries pelo Terraform, com o
recurso
[aws_synthetics_canary](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/synthetics_canary).

## Conclusão

O intuito desse post foi demonstrar a criação de uma simples API e a
checagem dela utilizando o recurso Synthetics Canaries do CloudWatch.

