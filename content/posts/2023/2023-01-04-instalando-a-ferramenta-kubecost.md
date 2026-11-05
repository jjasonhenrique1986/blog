---
title: "Instalando a ferramenta kubecost"
description: "Passo a passo da instalação do Kubecost, ferramenta de monitoramento e gerenciamento de custos para Kubernetes e cloud."
images: ["/images/capa-instalando-a-ferramenta-kubecost.png"]
author: Jason
date: 2023-01-04
lastmod: 2026-11-03
tags: [ "AWS", "FinOps", "Kubernetes" ]
type: post
---

> **Atualização (novembro de 2026):** o Kubecost foi comprado pela IBM em
> 2024. A documentação oficial agora fica no
> [IBM Docs](https://www.ibm.com/docs/en/kubecost/self-hosted/3.x) e o chart
> está no repositório [kubecost/kubecost](https://github.com/kubecost/kubecost),
> já na versão 3.x. O manifesto `kubecost.yaml` usado na instalação abaixo
> não está mais disponível: siga a instalação com Helm da documentação
> oficial. Telas e comandos deste post são da versão da época.

O intuito deste post é demonstrar a instalação do Kubecost, uma ferramenta
de monitoramento e gerenciamento de custos para um ambiente Kubernetes e
também para cloud, com suporte aos principais cloud providers (AWS,
Azure e
GCP).

O kubecost possui dois tipos de licença (free e paga). Na versão free as
informações coletadas ficam disponíveis apenas 15 dias.

Para mais informações sobre kubecost acessar os links abaixo:

- https://docs.kubecost.com/
- https://github.com/kubecost/cost-analyzer-helm-chart

## Pré-requisitos 

Como pré-requisito é necessário ter um cluster de kubernetes para a
instalação do kubecost. Para essa demonstração pode ser utilizado o kind
ou o rancher desktop.

Para criar um cluster com o kind, veja o [site oficial do
kind](https://kind.sigs.k8s.io/) ou o meu passo a passo de
[como criar um cluster local com o kind](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/).

Para o Rancher Desktop, veja o [site oficial do Rancher
Desktop](https://rancherdesktop.io/).

## Instalação 

No meu cenário, como é um ambiente de teste, vou fazer a instalação
utilizando o kubectl, mas ela também pode ser feita via helm.

Crie um namespace chamado kubecost, onde a aplicação será implantada,
com o comando abaixo:

``` bash
kucebctl create ns kubecost
```

Execute o comando abaixo para a instalação:

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubecost/cost-analyzer-helm-chart/master/kubecost.yaml -n kubecost
```

Serão criados os seguintes objetos, conforme a imagem abaixo:

![Saída do kubectl apply criando os objetos do Kubecost no namespace kubecost](/images/kubecost-2.webp)

Abaixo, um overview dos objetos criados pelo Kubecost:

![Saída do kubectl get all no namespace kubecost com pods, services, daemonset e deployments em execução](/images/kubecost-3.webp)

A instalação padrão inclui um prometheus-server, que é responsável por
coletar as informações do cluster. É possível utilizar o seu próprio
servidor do Prometheus, porém é recomendável usar esse Prometheus
separado.

## Acessando a console do kubecost 

Para acessar a console do kubecost será utilizado o comando kubectl
port-forward para testar localmente. Segue abaixo o comando:

``` bash
 kubectl port-forward service/kubecost-cost-analyzer 9090:9090 -n kubecost
```

Abaixo, alguns prints da console do Kubecost:

![Tela Overview do Kubecost com custos do cluster, economia possível e gráfico de tendência de custo](/images/kubecost-4.webp)

![Tela Overview do Kubecost com o custo por namespace e a seção Cloud Costs Breakdown](/images/kubecost-5.webp)

## Conclusão 

O intuito desse post foi demonstrar a instalação do kubecost. No próximo
post mostro as principais funcionalidades da console do
kubecost: [Overview da ferramenta kubecost](/posts/2023/2023-01-07-overview-da-ferramenta-kubecost/).