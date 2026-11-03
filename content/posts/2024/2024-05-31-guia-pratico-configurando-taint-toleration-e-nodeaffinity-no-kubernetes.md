---
title: "Guia Prático: Configurando Taint, Toleration e NodeAffinity no Kubernetes"
description: "Guia prático de Taint, Toleration e NodeAffinity para controlar em quais nós os pods executam no Kubernetes, usando o kind."
image: "/images/kubernetes.png"
author: Jason
date: 2024-05-31
tags: [ "Containers", "Kubernetes", "EKS" ]
type: post
showTableOfContents: true
---

![Guia Prático: Configurando Taint, Toleration e NodeAffinity no Kubernetes](/images/kubernetes.png)

O objetivo desse post é demonstrar como fazer os pods da sua aplicação
executarem em um determinado nó, usando o kind. Essa configuração também
vale para um Kubernetes em algum cloud provider, como o EKS da AWS.

## Pré-requisitos

Para fazer as demonstrações desse post, são necessários os seguintes
pré-requisitos:

-   Ter o kubectl instalado.
-   Ter um cluster local usando o kind. Para isso, siga esse
    [post](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/).

## Conceitos 

Antes de partir para a parte prática é necessário nivelar alguns
conceitos que serão usados para essa configuração. Abaixo seguem esses
conceitos:

### Taint 

O conceito de taint dentro do Kubernetes tem como principal função
bloquear ou repelir que determinados pods executem ou sejam schedulados
para determinado nó. Dentro do taint temos dois principais tipos de
efeitos:

-   **NoSchedule** -- a partir da configuração do taint, nenhum pod novo
    será alocado nesse nó, mas os pods que já estiverem nele continuarão
    executando.



-   **NoExecute** -- a partir da configuração do taint, nenhum pod será
    executado naquele nó: inclusive os pods que já estiverem nele serão
    terminados e alocados em outro nó sem esse taint.

Além das duas opções acima temos outra opção de effect chamada
**PreferNoSchedule** que tem como principal função ser um efeito mais
soft ou preferencial. Com esse efeito, o scheduler do Kubernetes tenta
não alocar pods no nó com esse taint, mas não é garantido.

Abaixo, a sintaxe utilizada para configurar um taint em um nó:

```bash
kubectl taint node name-node key=value:effect

kubectl taint node node01 enable=true:NoExecute
```

Para desconfigurar um taint usar o comando abaixo:

```bash
kubectl taint node name-node key=value:effect -

kubectl taint node node01 enable=true:NoExecute -
```

Para verificar se um node está com o taint configurado usar o comando
abaixo:

```bash
kubectl describe node name-node
```

### Toleration 

O conceito de toleration dentro do Kubernetes tem como principal função
permitir que determinados pods de um deploy sejam alocados e executados
em um nó que tenha um taint configurado. Abaixo, um exemplo de
configuração que deve ser inserida no arquivo de manifesto do seu
deploy para configurar um toleration:

```bash
tolerations:
- key: "enable"
  operator: "Equal"
  value: "true"
  effect: "NoExecute"
```

### NodeAffinity

O conceito de nodeAffinity dentro do Kubernetes tem a função de garantir
que um determinado pod vá para um determinado nó. Abaixo, um exemplo de
configuração que deve ser inserida no arquivo de manifesto do
seu deploy para configurar um nodeAffinity:

```bash
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
```

Para fazer a configuração de um nodeAffinity é necessário criar uma
label para o node ou usar uma label que já exista.

## Criação de deploy 

Agora vamos para a parte prática. Para isso, é necessário que você crie
um cluster local usando kind. Para verificar se o cluster está
operacional executar o comando abaixo:

```bash
kubectl get nodes
```

O primeiro passo é criar um namespace para nossa aplicação usando o
comando abaixo:

```bash
kubectl create namespace app
```

Agora vamos criar um deploy do nginx nesse namespace com 15 réplicas e
ver como ele se distribui entre os nós do cluster. Para isso, vamos usar
o arquivo de manifesto abaixo:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: app
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

Para gerar esse arquivo de manifesto usar o comando abaixo:

```bash
kubectl create deploy nginx --image=nginx --replicas=15 -n app --dry-run=client -o yaml > nginx.yaml
```

Agora vamos aplicar esse arquivo de manifesto usando o comando abaixo:

```bash
kubectl apply -f nginx.yaml
```

Agora vamos verificar onde os pods estão executando usando o comando
abaixo:

```bash
kubectl get pods -n app -o wide
```

Deverá apresentar uma saída conforme imagem abaixo:

![Pods do deployment nginx distribuídos entre os nodes cluster-jason-worker e cluster-jason-worker2](/images/taint-2.png)

Podemos perceber que os pods foram distribuídos entre os nós
**cluster-jason-worker** e **cluster-jason-worker2**.

## Configuração Taint e toleration 

Agora vamos partir para o segundo passo que é fazer a configuração do
taint e após isso fazer a inclusão do toleration em nosso deploy.
Abaixo, o comando utilizado para configurar o taint no nó
**cluster-jason-worker**:

```bash
kubectl taint node cluster-jason-worker enable=true:NoExecute
```

Para verificar se o taint foi aplicado com sucesso executar o comando
abaixo:

```bash
kubectl describe node cluster-jason-worker
```

E procurar pelo campo **Taints** conforme imagem abaixo:

![Saída do kubectl describe node mostrando o taint enable=true:NoExecute no cluster-jason-worker](/images/taint-3.webp)

Agora vamos fazer uma alteração no nosso deploy adicionando o bloco do
tolerations conforme arquivo abaixo:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: app
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
      tolerations:
      - key: "enable"
        operator: "Equal"
        value: "true"
        effect: "NoExecute"
```

Agora vamos executar o comando abaixo para fazer o replace dos pods com
a nova configuração:

```bash
kubectl replace -f nginx.yaml
```

Para validar se a configuração foi mesmo aplicada no cluster, execute o
comando abaixo para visualizar o arquivo de manifesto:

```bash
kubectl get deploy nginx -o yaml -n app
```

Agora vamos visualizar novamente a distribuição dos pods conforme
abaixo:

![Pods do nginx no namespace app rodando nos dois workers depois de aplicar o taint](/images/taint-4.webp)

Podemos perceber que ainda há pods nos dois nós (cluster-jason-worker e
cluster-jason-worker2). Isso acontece porque a toleration permite que o
pod rode no nó com taint, mas não obriga. Para obrigar, usamos o
nodeAffinity, que vamos ver na seção seguinte.

## Configurando um nodeAffinity 

Como visto na seção anterior, mesmo usando taint e toleration ainda há
pods nos dois nós. Para fazer a configuração no nodeAffinity vamos usar
a label já existente **kubernetes.io/hostname**. Para visualizar as
labels já existentes nos seus nós executar o comando abaixo:

```bash
kubectl get nodes --show-labels
```

Agora vamos fazer outra modificação em nosso deploy adicionando o bloco
affinity conforme arquivo abaixo:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: app
spec:
  replicas: 15
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - cluster-jason-worker
      containers:
      - image: nginx
        name: nginx
      tolerations:
      - key: "enable"
        operator: "Equal"
        value: "true"
        effect: "NoExecute"
```

Vamos fazer o replace dos pods novamente e verificar que os pods estão
apenas no nó **cluster-jason-worker** conforme imagem abaixo:

![Pods do nginx no namespace app rodando apenas no node cluster-jason-worker](/images/taint-5.webp)

Outro teste que podemos fazer para validar se a configuração está
correta é escalar o número de pods para 20 e verificar se os pods se
mantêm no mesmo nó executando o comando abaixo:

```bash
kubectl scale deploy/nginx -n app --replicas 20
```

Podemos verificar que foram criados mais 5 pods no mesmo nó conforme
imagem abaixo:

![Mais réplicas do nginx no namespace app, todas no node cluster-jason-worker](/images/taint-6.webp)

## Deletar cluster local 

Para deletar seu cluster criado usando o kind seguir os comandos abaixo:

```bash
kind get clusters
```

Para pegar o nome do cluster.

```bash
kind delete clusters name-cluster
```

Comando acima usado para deletar o cluster .

## Conclusão 

O objetivo desse post foi demonstrar como fazer os pods da sua aplicação
irem para um determinado nó, usando o kind. A mesma configuração serve
para o EKS, aplicando os taints e labels nos nós de um node group.

## Referências 

Abaixo as documentações que usei como base para escrever esse post:

-   [Taint e Toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
-   [Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
