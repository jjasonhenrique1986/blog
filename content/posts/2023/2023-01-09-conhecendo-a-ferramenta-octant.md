---
title: "Conhecendo a ferramenta Octant"
description: "Conheça o Octant, projeto open source que permite visualizar e gerenciar um cluster Kubernetes por interface gráfica."
image: "/images/octant.png"
author: Jason
date: 2023-01-09
tags: [ "Kubernetes", "Observabilidade", "Containers" ]
type: post
showTableOfContents: true
---

![Conhecendo a ferramenta Octant](/images/octant.png)

> **Atualização (novembro de 2026):** o Octant foi descontinuado e o
> repositório foi arquivado em janeiro de 2023 (hoje em
> [vmware-archive/octant](https://github.com/vmware-archive/octant)), então
> não recebe mais correções. Para uma interface gráfica do Kubernetes, veja
> o [Headlamp](https://github.com/kubernetes-sigs/headlamp), projeto do
> kubernetes-sigs, ou o [k9s](https://github.com/derailed/k9s), no terminal.

A ferramenta Octant é um projeto Open Source que permite a visualização
via interface gráfica do seu cluster Kubernetes, permitindo fazer todo o
gerenciamento do cluster e das aplicações por meio dela.

A ferramenta Octant era mantida pela VMware. Abaixo, links com mais informações:

- https://octant.dev/
- https://github.com/vmware-tanzu/octant

## Pré-requisitos 

Para testar essa ferramenta é necessário ter um cluster kubernetes
local. Pode ser usado o kind ou o rancher desktop para teste. Abaixo,
links com mais informações:

- https://kind.sigs.k8s.io/ (veja também [como criar um cluster local com o kind](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/))
- https://rancherdesktop.io/

Também é necessário que o kubeconfig esteja configurado, pois a
ferramenta coleta informações desse arquivo conforme imagem abaixo:

![Tela de preferências do Octant mostrando o caminho do kubeconfig em uso](/images/octant-2.webp)

Para mais informações, veja a [documentação do
kubeconfig](https://kubernetes.io/pt-br/docs/concepts/configuration/organize-cluster-access-kubeconfig/).

## Instalação 

Para utilizar, é necessário instalar um cliente na sua máquina local,
que suporta Windows, Linux e Mac.

Para fazer o download basta acessar o link abaixo:

-  https://github.com/vmware-tanzu/octant/releases/tag/v0.25.1

No momento em que escrevo este post, essa é a última versão da ferramenta.

## Overview 

Nessa seção será apresentado um overview da ferramenta.

![Tela Workloads do Octant com os deployments do Kubecost e o uso de memória e CPU de cada um](/images/octant-3.webp)

Na imagem acima conseguimos ter a visualização de todos os workloads que
estão rodando em determinado namespace. Assim como o uso de CPU e
Memória para cada workload.

![Octant mostrando o deployment kubecost-cost-analyzer e o grafo dos recursos relacionados](/images/octant-4.webp)

![Aba Resource Viewer do Octant com o grafo do deployment kubecost-cost-analyzer, replicaset, pods e configmaps](/images/octant-5.webp)

Acima outros exemplos de visualizações.

Permite a visualização dos objetos do kubernetes que determinado
workload está utilizando. Abaixo, alguns exemplos:

![Overview do namespace kubecost no Octant com a lista de deployments e pods](/images/octant-6.webp)

![Lista de services do namespace kubecost no Octant, com tipo, IP e portas](/images/octant-7.webp)

![Aba Summary do deployment kubecost-cost-analyzer no Octant com configuração, status e pods](/images/octant-8.webp)

É possível fazer um port-forward com apenas um clique

![Pod template do deployment no Octant com o container cost-model e os botões de port forward das portas 9003 e 9090](/images/octant-9.png)

Permite visualizar as variáveis de ambiente

![Variáveis de ambiente do container listadas no Octant, abaixo das portas com port forward](/images/octant-10.webp)

Visualização do yaml utilizado pelo Deploy

![Aba YAML do deployment kubecost-cost-analyzer no Octant](/images/octant-11.webp)

Visualização de logs e conexão ao pod através da ferramenta

![Aba Logs de um pod do kubecost-cost-analyzer no Octant](/images/octant-12.webp)

![Aba Terminal de um pod do kubecost-cost-analyzer no Octant, com um shell aberto no container](/images/octant-13.webp)

É possível visualizar os eventos, conforme a imagem abaixo:

![Tela de eventos de um namespace no Octant](/images/octant-14.webp)

Permite aplicar um yaml através da ferramenta

![Janela Apply YAML do Octant para aplicar manifestos no cluster](/images/octant-15.webp)

Também é possível integrar a ferramenta com outros plugins conforme link
abaixo:

- https://github.com/topics/octant-plugin

## Conclusão 

O objetivo desse post foi demonstrar a utilização da ferramenta Octant e
como ela pode ajudar no gerenciamento de um cluster Kubernetes via
interface gráfica, aumentando a sua produtividade.

Existem ferramentas similares como o Kubernetes-Dashboard e o Lens.
Abaixo links com mais informações:

- [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
- [Lens](https://k8slens.dev/)