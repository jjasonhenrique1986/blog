---
title: Conhecendo a ferramenta trivy
description: "Conheça o Trivy, scanner de segurança da Aqua Security e tópico cobrado no exame CKS (Certified Kubernetes Security Specialist)."
images: ["/images/capa-conhecendo-a-ferramenta-trivy.png"]
author: Jason
date: 2023-11-05
tags: [ "Kubernetes", "Segurança", "Containers", "CI/CD" ]
type: post
showTableOfContents: true
---

O Trivy é uma ferramenta desenvolvida pela empresa Aqua Security e
também é um dos tópicos cobrados no exame CKS (Certified Kubernetes
Security Specialist). Abaixo, alguns links com
mais informações do trivy:

-   <https://github.com/aquasecurity/trivy>
-   <https://aquasecurity.github.io/trivy/>

Com o trivy você consegue fazer as seguintes verificações:

-   Scan de Vulnerabilidades
-   Erros de configuração em IaC
-   Detectar dados sensíveis

É possível escanear os seguintes targets:

-   Imagem de Container
-   Repositório de algum sistema de versionamento de código como por
    exemplo o GitHub
-   Sistema de Arquivos
-   Clusters Kubernetes
-   Contas AWS

O objetivo desse artigo é demonstrar o scan de vulnerabilidades em uma
imagem de container e em um repositório do github usando o Trivy.

## Instalação 

Para usar o trivy você pode instalar diretamente na sua máquina ou usar
o trivy como um container do Docker.

### Instalação no Linux 

No meu caso vou demonstrar a instalação em uma máquina Linux. A última
versão está nas [releases do Trivy no
GitHub](https://github.com/aquasecurity/trivy/releases). Abaixo os
comandos que você deve executar para fazer o download e a instalação:

``` bash
wget https://github.com/aquasecurity/trivy/releases/download/v0.46.0/trivy_0.46.0_Linux-64bit.deb
sudo dpkg -i trivy_0.46.0_Linux-64bit.deb
```

Para validar a instalação executar o comando abaixo:

``` bash
trivy version
```

### Docker 

Para utilizar o trivy usando um container basta usar a imagem
**aquasec/trivy:latest**. Por exemplo, para verificar a versão do Trivy,
execute o comando abaixo:

``` bash
docker run aquasec/trivy:latest version
```

## Scan de Imagem de Container 

Para executar um scan de vulnerabilidades em alguma imagem de container
basta executar o comando abaixo:

``` bash
trivy image nginx

docker run aquasec/trivy:latest image nginx
```

Onde **nginx** é o nome da imagem de container

Deverá aparecer uma saída similar à abaixo:

![Resultado do trivy image nginx com a tabela de vulnerabilidades por biblioteca e severidade](/images/trivy-2.webp)

Você pode filtrar por severidade HIGH e CRITICAL usando o comando abaixo:

``` bash
trivy image nginx --severity=HIGH --severity=CRITICAL
```

Agora deverão aparecer somente as vulnerabilidades com a severidade HIGH
e CRITICAL conforme imagem abaixo:

![Resultado do trivy image nginx filtrado pelas severidades HIGH e CRITICAL](/images/trivy-3.webp)

Na coluna **status**, podemos verificar que o status está como
**affected**: isso quer dizer que ainda não existe uma versão corrigida
para essas vulnerabilidades.

## Scan de repositório 

Agora vamos fazer um scan de vulnerabilidades no repositório do
[Prometheus](https://github.com/prometheus/prometheus), executando o
comando abaixo filtrando apenas as vulnerabilidades com severidade HIGH.

``` bash
trivy repository https://github.com/prometheus/prometheus --severity=HIGH
```

Abaixo segue a saída do comando:

![Resultado do trivy repository no repositório do Prometheus com as vulnerabilidades de severidade HIGH](/images/trivy-4.webp)

Nesse caso, a coluna **status** está como **fixed**: isso quer dizer que
já existe uma versão corrigida. Na coluna **Fixed Version** está a
versão em que a vulnerabilidade foi corrigida.

## Integração 

O Trivy se integra com diversas ferramentas que permitem adicionar
segurança ao workflow da sua aplicação, como ferramentas de CI/CD e
IDEs. Veja todas as integrações na [documentação do ecossistema do
Trivy](https://aquasecurity.github.io/trivy/v0.46/ecosystem/).

## Conclusão 

O objetivo desse artigo foi demonstrar um pouco da utilização do trivy
para efetuar scan de vulnerabilidades.

Para análise estática de código de infraestrutura, veja o post sobre o
[kics](/posts/2022/2022-09-10-conhecendo-a-ferramenta-de-analise-estatica-kics/). E para ver o Trivy rodando junto com outros
scanners e uma revisão com IA em Pull Requests, veja
[Revisão de PR de infraestrutura com IA, parte 3](/posts/2026/2026-11-03-revisao-de-pr-de-infraestrutura-com-ia-parte-3/).