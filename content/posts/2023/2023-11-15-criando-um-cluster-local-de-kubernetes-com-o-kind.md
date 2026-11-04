---
title: Criando um cluster local de kubernetes com o kind
description: "Passo a passo para criar um cluster Kubernetes local com o kind para testes e estudos."
images: ["/images/capa-criando-um-cluster-local-de-kubernetes-com-o-kind.png"]
author: Jason
date: 2023-11-15
tags: [ "Kubernetes", "Containers" ]
type: post
---

O objetivo desse artigo é fazer um passo a passo de como subir um
cluster Kubernetes localmente com o kind, para que você possa fazer
testes ou estudar Kubernetes.

Lembrando que esse tutorial está focado na criação de um cluster
Kubernetes local em uma máquina Linux (no meu caso, uma distribuição Debian).

## Instalação Docker 

O primeiro passo é a instalação do Docker para conseguir subir o kind.
Para efetuar a instalação do Docker devemos executar o comando abaixo:

``` bash
curl -fsSL https://get.docker.com | sh
```

Para verificar se o Docker foi instalado com sucesso executar o comando
abaixo:

``` bash
docker 
```

## Instalação kubectl 

Devemos fazer a instalação do utilitário de linha de comando kubectl
para conseguir administrar o cluster kubernetes via linha de comando.
Abaixo, os comandos que devem ser executados:

``` bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
```

Para validar a instalação do kubectl executar o comando abaixo:

``` bash
kubectl
```

## Instalação kind 

Para a instalação do kind devemos executar os comandos abaixo:

``` bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/
```

Para a verificação da instalação do kind executar o comando abaixo:

``` bash
kind 
```

## Criação cluster kubernetes

Agora vamos fazer a criação de um cluster kubernetes localmente
executando o kind. Vou fazer a criação do cluster de forma declarativa
usando um arquivo yaml conforme abaixo:

``` bash
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: nome-do-cluster
runtimeConfig:
  "api/alpha": "false"
nodes:
- role: control-plane
- role: worker
- role: worker
```

Crie um arquivo chamado **create_cluster.yaml**
com o conteúdo acima e execute o comando abaixo:

``` bash
kind create cluster --config create_cluster.yaml
```

Deve aparecer uma saída conforme abaixo:

![Saída do kind create cluster criando o cluster cluster-jason e configurando o contexto do kubectl](/images/kind-2.webp)

Para verificar se o cluster foi criado corretamente executar os comandos
abaixo:

```bash
kubectl get nodes
kubectl get pods -A
```

Para deletar o cluster criado basta executar o comando abaixo:

```bash
kind delete clusters nome-do-cluster
```

## Instalação Helm 

Agora vamos instalar o Helm, que é um gerenciador de pacotes do
Kubernetes para instalar aplicações. Primeiramente, faça o download do
pacote da última versão nas [releases do Helm no
GitHub](https://github.com/helm/helm/releases).

Depois executar os comandos abaixo:

```bash 
tar -zxvf nome-do-pacote
sudo mv linux-amd64/helm /usr/local/bin
```

Para verificar se o helm foi instalado com sucesso, executar o comando
abaixo:

```bash
helm
```

## Conclusão 

O objetivo desse artigo foi criar um tutorial para a criação de um
cluster kubernetes localmente e também alguns pacotes essenciais para
administração do cluster. Abaixo alguns links para te ajudar em caso de
dúvida:

-   <https://helm.sh/docs/intro/install/>
-   <https://kind.sigs.k8s.io/docs/user/quick-start/>
-   <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/>
