---
title: "Instalando a ferramenta kubecost"
description: "Passo a passo da instalação do Kubecost, ferramenta de monitoramento e gerenciamento de custos para Kubernetes e cloud."
image: "/images/kubecost.png"
author: Jason
date: 2023-01-04
tags: [ "AWS", "FinOps", "Kubernetes" ]
type: post
showTableOfContents: true
---

![Instalando a ferramenta kubecost](/images/kubecost.png)

> **Atualização (novembro de 2026):** o Kubecost foi comprado pela IBM em
> 2024. A documentação oficial agora fica no
> [IBM Docs](https://www.ibm.com/docs/en/kubecost/self-hosted/3.x) e o chart
> está no repositório [kubecost/kubecost](https://github.com/kubecost/kubecost),
> já na versão 3.x. O manifesto `kubecost.yaml` usado na instalação abaixo
> não está mais disponível: siga a instalação com Helm da documentação
> oficial. Telas e comandos deste post são da versão da época.

O intuito deste post é demostrar a instalação do kubecost uma ferramenta
de monitoramento e gerenciamento de custos para um ambiente kubernetes e
também para cloud suportando os principais cloud providers (AWS, Azure e
GCP).

O kubecost possui dois tipos de licença (free e paga). Na versão free as
informações coletadas ficam disponíveis apenas 15 dias.

Para mais informações sobre kubecost acessar os links abaixo:

- https://docs.kubecost.com/
- https://github.com/kubecost/cost-analyzer-helm-chart

## Pré-Requisitos 

Como pré-requisito é necessário ter um cluster de kubernetes para a
instalação do kubecost. Para essa demonstração pode ser utilizado o kind
ou o rancher desktop.

Para mais informações sobre a instalação e criação de um cluster usando
kind utilizar
esse [link](https://kind.sigs.k8s.io/) ou o meu passo a passo de
[como criar um cluster local com o kind](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/).

Para informações sobre o rancher desktop acesse
esse [link](https://rancherdesktop.io/).

## Instalação 

No meu cenário como é um ambiente de teste vou fazer a instalação
utilizando o kubectl mas a instalação também pode ser feita via helm.

Criar um namespace chamado kubecost onde será deployado a aplicação.
Segue abaixo comando:

``` bash
kucebctl create ns kubecost
```

Executar o comando abaixo para instalação:

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubecost/cost-analyzer-helm-chart/master/kubecost.yaml -n kubecost
```

Será deployado os seguintes objetos conforme imagem abaixo:

![Saída do kubectl apply criando os objetos do Kubecost no namespace kubecost](/images/kubecost-2.webp)

Abaixo overview dos objetos deployados pelo kubecost

![Saída do kubectl get all no namespace kubecost com pods, services, daemonset e deployments em execução](/images/kubecost-3.webp)

Na instalação default é efetuado a instalação de um deploy do
prometheus-server que será responsável por coletar as informações do
cluster. É possivel utilizar seu próprio servidor do prometheus porém é
recomendavel a utilização desse prometheus apartado.

## Acessando a console do kubecost 

Para acessar a console do kubecost será utilizado o comando kubectl
port-forward para testar localmente. Segue abaixo comando:

``` bash
 kubectl port-forward service/kubecost-cost-analyzer 9090:9090 -n kubecost
```

Abaixo alguns prints da console do kubecost

![Tela Overview do Kubecost com custos do cluster, economia possível e gráfico de tendência de custo](/images/kubecost-4.webp)

![Tela Overview do Kubecost com o custo por namespace e a seção Cloud Costs Breakdown](/images/kubecost-5.webp)

## Conclusão 

O intuito desse post foi demonstrar a instalação do kubecost. No próximo
post estarei mostrando as principais funcionalidades da console do
kubecost: [Overview da ferramenta kubecost](/posts/2023/2023-01-07-overview-da-ferramenta-kubecost/).