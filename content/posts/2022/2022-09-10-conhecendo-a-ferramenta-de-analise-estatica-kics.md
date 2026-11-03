---
title: "Conhecendo a ferramenta de analise estática kics"
description: "Como usar o KICS para fazer análise estática de código de Infraestrutura como Código, como Terraform e CloudFormation."
image: "/images/kics.jpg"
author: Jason
date: 2022-09-10
tags: [ "CI/CD", "Segurança", "IaC" ]
type: post
showTableOfContents: true
---
![Conhecendo a ferramenta de analise estática kics](/images/kics.jpg)

Nesse artigo irei demonstrar a utilização de um ferramenta
chamada **kics** ou **keeping infrastructure as code secure** tem a
função de executar uma analise estática em códigos de Infraestrutura
como Código como por exemplo Terraform, Cloudformation e outros.

O kics é uma ferramenta Open Source desenvolvida em Golang usando OPA
(Open Policy Agent). Para mais informações sobre OPA consulte
esse [link](https://www.openpolicyagent.org/docs/latest/#overview). O OPA permite você aplicar políticas para seus
microsserviços, kubernetes, pipelines de CI/CD, API Gateways e muitas
outras opções.

O kics faz uma analise estática procurando por vulnerabilidades de
segurança, questões de compliance e erros de configurações em suas
ferramentas de Infraestrutura como Código. No momento que escrevo esse
artigo as seguintes ferramentas abaixo são suportadas:

![Logos das ferramentas suportadas pelo KICS, como Terraform, Kubernetes, Docker, Ansible, CloudFormation, Helm, OpenAPI, AWS CDK e gRPC](/images/kics-2.jpg)

Através do kics você pode criar novas queries conforme a necessidade do
seu ambiente. Mais informações
nesse [link](https://docs.kics.io/latest/creating-queries/).

## Instalação 

Para a utilização do kics você tem duas opções instalar através do
código fonte ou utilizar o kics através de uma imagem docker. No meu
caso vou utilizar uma imagem docker.

### Docker 

``` bash
docker pull checkmarx/kics:latestdocker run -t -v "{path_to_host_folder_to_scan}":/path checkmarx/kics scan -p "/path" -o "/path/"
```

Para mais informações sobre a instalação pode consultar
esse [link](https://docs.kics.io/latest/getting-started/).

## Demonstração 

Para essa demonstração irei utilizar o
seguinte [repositório](https://github.com/addamstj/tfcourse). Por padrão o kics gera um arquivo json para melhorar
a visualização vamos gerar um arquivo html.

Para executar basta executar o comando abaixo:

``` bash
nerdctl run -it -v "C:\Users\jhmrj\Documents\tfcourse":/tmp checkmarx/kics scan -p "/tmp/" -o "/tmp/results" --report-formats "html"
```

**Observação:** Estou utilizando o utilitário nerdctl pois na minha
máquina estou executando o containerd. No docker irá funcionar da mesma
forma.

![Terminal executando o KICS em container com nerdctl e o resumo do scan: 21 arquivos analisados e 1040 queries carregadas](/images/kics-3.jpg)

Abaixo algumas das regras identificadas pelo kics:

![Resultado do KICS apontando a regra Variable Without Description em variáveis do Terraform](/images/kics-4.jpg)

![Resultado do KICS apontando a regra RDS With Backup Disabled, de severidade média](/images/kics-5.jpg)

![Resultado do KICS apontando a regra EC2 Instance Has Public IP, de severidade alta, em recursos aws_instance](/images/kics-6.jpg)

![Resumo do scan do KICS: 10 achados HIGH, 19 MEDIUM, 11 LOW e 64 INFO, com o relatório salvo em /tmp/results/results.html](/images/kics-7.jpg)

Foi gerado um arquivo html no caminho /tmp/results/results.html. Abaixo
segue alguns prints desse report.

![Relatório HTML do KICS com o total de vulnerabilidades por severidade e o achado DB Instance Storage Not Encrypted](/images/kics-8.jpg)

![Detalhe do relatório HTML do KICS para o achado EC2 Instance Has Public IP, com os arquivos e linhas afetados](/images/kics-9.jpg)

## Integração 

Com o kics você consegue integrar com várias ferramentas de CI/CD e
também existe um plugin para vscode.
Nesse [link](https://docs.kics.io/latest/integrations/) você consegue verificar todas as integrações
disponíveis.

Para scan de vulnerabilidades em imagens de container, que complementa a
análise de IaC, veja o post sobre o [Trivy](/posts/2023/2023-11-05-conhecendo-a-ferramenta-trivy/).

## Referências 

- [Github Kics](https://github.com/Checkmarx/kics)
- [Documentação Oficial](https://docs.kics.io/latest/getting-started/)