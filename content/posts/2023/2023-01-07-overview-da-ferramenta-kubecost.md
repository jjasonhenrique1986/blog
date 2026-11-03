---
title: "Overview da ferramenta kubecost"
description: "Um overview da console do Kubecost para monitorar e gerenciar o custo do Kubernetes e do cloud provider (AWS, Azure e GCP)."
image: "/images/kubecost.jpg"
author: Jason
date: 2023-01-07
tags: [ "AWS", "FinOps", "Kubernetes" ]
type: post
showTableOfContents: true
---

![Overview da ferramenta kubecost](/images/kubecost.jpg)

> **Atualização (novembro de 2026):** o Kubecost foi comprado pela IBM em
> 2024. A documentação oficial agora fica no
> [IBM Docs](https://www.ibm.com/docs/en/kubecost/self-hosted/3.x) e o chart
> está no repositório [kubecost/kubecost](https://github.com/kubecost/kubecost),
> já na versão 3.x. Telas e comandos deste post são da versão da época.

O objetivo desse post é fazer um overview da console de administração da
ferramenta kubecost que ajuda no gerenciamento e monitoramento do custo
de um ambiente kubernetes e também do custo do cloud provider (AWS,
Azure e GCP) que você utiliza.

## Pré-Requisitos

Como pré-requisito é necessário já ter feito a instalação do kubecost
para fazer a instalação basta seguir
esse [link](/posts/2023/2023-01-04-instalando-a-ferramenta-kubecost/) com
as intruções.

## Overview da console do Kubecost 

Nas seções abaixo irei apresentar um pequeno overview da console de
adminstração do kubecost.

### Seção Overview 

Abaixo alguns prints da seção overview:

![Seção Overview do Kubecost com os custos do Kubernetes, custos de cloud, economia possível e eficiência do cluster](/images/kubecost-6.jpg)

![Seção Overview do Kubecost com o detalhamento de custo por namespace e o gráfico de tendência por namespace](/images/kubecost-7.jpg)

As seguintes informações estão disponíveis:

- Custo total do Kubernetes nos últimos 7 dias;
- Custo total da Cloud nos últimos 7 dias (é necessário a integração
    com o cloud provider mais informações
    nesse [link](https://docs.kubecost.com/install-and-configure/install/cloud-integration/multi-cloud));
- Custo total dos últimos 7 dias;
- Possible Savings são recomendações que podem ser aplicadas ao
    ambiente;
- Efficiency é um calculo feito baseado na utilização de CPU e Memória
    durante determinado tempo;
- Quantidade de nós e pods por cluster;
- Namespaces com maior gasto;

### Seção Savings 

Abaixo alguns prints da seção Savings:

![Seção Savings do Kubecost com a economia mensal estimada e as recomendações, como ajustar requests de containers e redimensionar discos](/images/kubecost-8.jpg)

![Recomendação do Kubecost para redimensionar discos locais com baixa utilização](/images/kubecost-9.jpg)

Na seção savings são feitas recomendações para o ambiente que irá gerar
economia para o ambiente kubernetes e para cloud:

- Recomendação de Request/Limit para CPU e Memória;
- Mostra os recursos não utilizados na Cloud;
- Mostra Workloads abandonados;
- Resize de disco no node ou resize de node;

### Seção Assets e Allocations 

Abaixo alguns prints da seção Allocations:

![Seção Allocations do Kubecost com o custo acumulado por namespace em gráfico e tabela](/images/kubecost-10.jpg)

![Detalhe do namespace kubecost na seção Allocations, com custo por recurso, eficiência de CPU e RAM e economia sugerida](/images/kubecost-11.jpg)

Na seção Allocations são mostrados informações de custo separado por
CPU, RAM, Network e LoadBalance separado por namespace.

Prints da seção Assets:

![Seção Assets do Kubecost com o custo dos últimos 7 dias agrupado por tipo de recurso](/images/kubecost-12.jpg)

![Seção Assets do Kubecost detalhando o custo dos discos do cluster](/images/kubecost-13.jpg)

Na seção Assets mostra as seguintes informações:

- Mostra os custos baseados nos recursos por exemplo Node, Disk,
    Network;
- Tem amostragem por data (lembrando que na versão free mostra
    informações de apenas 15 dias);
- Mostra o quanto cada node ou cada disk no ambiente está gastando;

### Seção Health 

Abaixo prints da seção Health:

![Seção Health do Kubecost com nota de saúde 94 e a lista de verificações do cluster](/images/kubecost-14.jpg)

![Continuação da seção Health do Kubecost com as verificações de confiabilidade, armazenamento e rede e suas severidades](/images/kubecost-15.jpg)

Na seção Health é feito um checklist com boas práticas baseado em
confiabilidade e performance. Nesse checklist é feito um score de 0 a
100. Abaixo algumas dessas checagens:

- Se existem pods com status CrashLoopBackOff;
- Se existem pods com status pending;
- Pods com baixo uso de cpu;
- Memoria e Disco Pressure;
- Custo do Cluster aumentou mais de 10%;
- Evento de Out Of Memory;

### Seção Alerts e Reports 

Na seção reports permite você criar relatórios baseado na seção Asset e
Allocation conforme imagem abaixo:

![Seção Reports do Kubecost com os relatórios salvos de alocação por namespace e de assets por tipo](/images/kubecost-16.jpg)

Na seção Alerts possui a funcionalidade de enviar alertas para um canal
do slack. Segue abaixo imagem:

![Seção Alerts do Kubecost com a configuração de alertas de saúde e do webhook do Slack](/images/kubecost-17.jpg)

Abaixo tipos de alertas que podem ser criados:

![Lista de tipos de alerta do Kubecost: Allocation Budget, Allocation Efficiency, Allocation Recurring Update, Allocation Spend Change, Asset Budget e Cloud Report](/images/kubecost-18.jpg)

## Conclusão 

O intuito desse post foi fazer um overview sobre a console de
administração do kubecost. Para mais informações do kubecost pode
consultar a documentação através
desse [link](https://docs.kubecost.com/).