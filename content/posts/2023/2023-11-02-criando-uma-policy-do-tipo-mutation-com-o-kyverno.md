---
title: "Criando uma policy do tipo Mutation com o kyverno"
description: "Continuação da série sobre Kyverno: como criar uma policy do tipo Mutation em um cluster Kubernetes."
images: ["/images/capa-criando-uma-policy-do-tipo-mutation-com-o-kyverno.png"]
author: Jason
date: 2023-11-02
tags: [ "Kubernetes", "Segurança", "Containers" ]
type: post
showTableOfContents: true
---

O objetivo deste artigo é a criação de uma policy do tipo Mutation no
Kyverno. Esse artigo é uma continuação do primeiro artigo onde falei
como instalar o kyverno e criei uma policy do tipo validation no
kyverno. O primeiro artigo é o [Conhecendo um pouco sobre o
kyverno](/posts/2023/2023-10-29-conhecendo-um-pouco-sobre-o-kyverno/).

## Pré-requisitos 

Para conseguir criar uma policy do tipo Mutation no kyverno você
necessitará dos seguintes pré-requisitos:

-   Ter um cluster kubernetes
-   Ter o kubectl instalado
-   Ter o Kyverno já instalado no seu cluster

Para a instalação do kyverno você também pode consultar o primeiro
artigo que falei sobre o kyverno.

## Testando uma policy do tipo Mutation 

Para testar uma policy do tipo mutation no kyverno vamos criar uma
clusterpolicy que força a aplicação da label **environment: develop**
conforme abaixo:

``` bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-labels
spec:
  rules:
  - name: add-labels
    match:
      any:
      - resources:
          kinds:
          - Deployment
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            environment: develop
        spec:
          template:
            metadata:
              labels:
                environment: develop
  validationFailureAction: Enforce
```

Para aplicar a clusterpolicy criar um arquivo chamado add-labels.yaml
com o conteúdo acima e executar o comando abaixo:

``` bash
kubectl apply -f add-labels.yaml
```

Para validar a criação da cluster policy executar o comando abaixo:

``` bash
kubectl get clusterpolicy
```

Agora vamos criar um deployment sem a label **environment: develop**
conforme abaixo:

``` bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app
  name: app
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
        owner: jason
    spec:
      containers:
      - image: nginx
        name: nginx
```

Para aplicar o deployment criar um arquivo chamado nginx.yaml e executar
o comando abaixo:

``` bash
kubectl apply -f nginx.yaml
```

Para visualizar se a label foi aplicada você pode executar os comandos abaixo:

``` bash
kubectl get deploy -o jsonpath='{.items[0].metadata.labels}' -n app

kubectl get deploy -o jsonpath='{.items[0].spec.template.metadata.labels}' -n app
```

Deve retornar as labels conforme abaixo:

``` bash
{"app":"app","environment":"develop"}

{"app":"app","environment":"develop","owner":"jason"}
```

## Conclusão 

O objetivo desse artigo foi demonstrar a utilização de uma policy do
tipo Mutation no Kyverno. Existem outras policies que podem ser criadas
na [biblioteca de policies do Kyverno](https://kyverno.io/policies/).
