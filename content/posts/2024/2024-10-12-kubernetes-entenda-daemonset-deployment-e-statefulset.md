---
title: "Kubernetes: Entenda DaemonSet, Deployment e StatefulSet"
description: "Entenda as diferenças entre DaemonSet, Deployment e StatefulSet no Kubernetes, com exemplos em um cluster local com kind."
image: "/images/kubernetes.png"
author: Jason
date: 2024-10-12
tags: [ "Kubernetes", "EKS", "Containers" ]
type: post
showTableOfContents: true
---

![Kubernetes: Entenda DaemonSet, Deployment e StatefulSet](/images/kubernetes.png)

O objetivo desse artigo é explicar e demonstrar sobre os objectos
DaemonSet, Deployment e StatefulSet no Kubernetes, a principal
ferramenta de orquestração de containers. Para os testes iremos utilizar
um cluster kubernetes local usando o kind.

Para a criação do seu cluster local usando kind você pode seguir esse
[link](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/).
Nesse cluster será criado um nó como ControlPlane e outros dois nós como
DataPlane.

## Pré Requisitos 

Para conseguir acompanhar esse artigo será necessário os seguintes pré
requisitos abaixo:

-   Ter um cluster kubernetes no nossso caso utilizaremos o kind.
-   Ter o kubectl instalado.

Para os exemplos iremos criar um namespace onde serão feitos nossos
testes. Para criar esse namespace executar o comando abaixo:

```bash
kubectl create ns app
```

## DaemonSet 

### O que é? 

O **DaemonSet** é um tipo de controlador no Kubernetes que garante que
uma cópia de um determinado Pod seja executada em cada node de um
cluster. Ele é especialmente útil para rodar serviços que precisam estar
presentes em todos os nós, como agentes de monitoramento, logs ou
sistemas de rede.

### Quando usar? 

-   **Monitoramento e Logging**: Se você precisa coletar logs ou
    métricas de todos os nós, o DaemonSet é ideal.
-   **Networking**: Usado para implantar ferramentas de rede, como
    CNI(Container Network Interface) por exemplo o weave-net, que devem
    estar presentes em todos os nós.

### Exemplo de uso: 

Para esse primeiro exemplo vamos criar um arquivo chamado daemonset.yaml
com o conteúdo abaixo:

```bash
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: app
spec:
  selector:
    matchLabels:
      name: fluent-bit
  template:
    metadata:
      labels:
        name: fluent-bit
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        resources:
          limits:
            memory: 200Mi
            cpu: 100m
```

Para aplicar o exemplo acima executar o comando abaixo:

```bash
kubectl apply -f daemonset.yaml
```

Neste exemplo, o DaemonSet implementa um agente do fluent-bit para
coletar logs de cada nó do cluster. Esse exemplo é apenas para mostrar o
funcionamento do daemonset não pretendemos fazer testes com o agente do
fluent-bit.

Para verificar o funcionamento dos pods executar o comando abaixo:

```bash
kubectl get pods -n app
```

Deverá retornar uma saída similar a imagem abaixo:

![Pods do DaemonSet fluent-bit em execução](/images/deploy-2.webp)

Como temos apenas dois nós que executam como DataPlane foram criados
apenas dois pods do fluent-bit. Para retornar o nome dos nós que os pods
estão executando adicionar a flag -o wide conforme imagem abaixo:

![Pods do fluent-bit distribuídos, um em cada node worker](/images/deploy-3.webp)

O comando abaixo mostra quantos daemonsets estão executando no cluster:

```bash
kubectl get daemonset -n app
```

![Saída do kubectl get ds mostrando o DaemonSet fluent-bit com 2 pods prontos](/images/deploy-4.webp)

## Deployment 

### O que é? 

O **Deployment** é o controlador mais utilizado no Kubernetes para
gerenciar a criação e a escalabilidade de réplicas de Pods.

### Quando usar? 

-   **Aplicações Stateless**: Ideal para a maioria das aplicações web,
    APIs ou microservices que não mantêm estado interno entre diferentes
    réplicas.

### Exemplo de uso: 

Nesse segundo exemplo vamos criar um arquivo chamado deployment.yaml com
o conteúdo abaixo:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx  
  namespace: app  
  labels:
    app: nginx
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21.1
        ports:
        - containerPort: 80
```

Para aplicar o manifesto acima executar o comando:

```bash
kubectl apply -f deployment.yaml
```

Aqui, o Deployment define dez réplicas de um container NGINX. Em caso de
falha de um Pod, o Deployment cria automaticamente uma nova réplica para
garantir a disponibilidade.

Para verificar a execução dos pods executar o comando abaixo:

```bash
kubectl get pods -n app
```

![Dez pods do deployment nginx em execução no namespace app](/images/deploy-5.webp)

Na configuração default do kubernetes podemos verificar que os pods se
distribuem entre os nós conforme imagem abaixo:

![Pods do deployment nginx distribuídos entre os nodes cluster-jason-worker e cluster-jason-worker2](/images/deploy-6.webp)

O comando abaixo mostra informações do deployment:

```bash
kubectl get deployment -n app
```

![Saída do kubectl get deployment com o nginx em 10 de 10 réplicas prontas](/images/deploy-7.webp)

## StatefulSet 

### O que é? 

O **StatefulSet** é semelhante ao Deployment, mas é utilizado para
gerenciar aplicações stateful, onde a persistência dos dados são
importantes. O StatefulSet garante que os Pods sejam implantados e
escalonados de maneira previsível.

### Quando usar? 

-   **Bancos de Dados**: Ideal para bancos de dados distribuídos, como
    MySQL, Postgress ou qualquer aplicação que exija persistência de
    dados entre os pods. Também utilizado para aplicações que necessitam
    de um sequência lógica.

### Exemplo de uso: 

Nesse terceiro exmeplo vamos criar um arquivo chamado statefulset.yaml
com o conteúdo abaixo:

```bash
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: app
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

Neste exemplo, um StatefulSet é usado para criar três réplicas de um
container NGINX, cada uma com um volume de armazenamento persistente
separado.

Para verificar a execução dos pods executar o comando abaixo:

```bash
kubectl get pods -n app
```

![Pods web-0, web-1 e web-2 do StatefulSet criados em ordem](/images/deploy-8.webp)

Outro ponto que podemos verificar é a sequência lógica na subida dos
pods sendo que o primeiro pods foi o web-0, o segundo web-1 e o terceiro
web-2 isso pode ser uma premissa para alguma aplicação.

O comando abaixo mostra informações do statefulset:

```bash
kubectl get statefulset -n app
```

## Bônus: Replicaset 

O **ReplicaSet** é um objecto que tem como responsabilidade manter um
número especifico de réplicas num deployment. Ele é criado
automaticamente quando criamos um deployment. Abaixo o comando para
verificar o replicaset:

```bash
kubectl get replicaset -n app
```

![Saída do kubectl get rs com o replicaset do nginx e 10 réplicas](/images/deploy-9.webp)

Outro ponto do replicaset é que sempre que é feito um novo deploy sempre
será criado um novo replicaset e as replicas estarão disponiveis no novo
replicaset. Abaixo vamos similar um novo deployment executando o comando
abaixo:

```bash
kubectl set image deploy/nginx nginx=nginx:1.19 -n app
```

Na imagem abaixo podemos perceber que foi criado um novo replicaset e os
pods do deployment foram transferidos para esse novo replicaset.

![Dois replicasets do nginx depois da atualização: o novo com 10 réplicas e o antigo com 0](/images/deploy-10.webp)

## Conclusão 

O principal objetivo desse artigo foi demonstrar o que é e como utilizar
os seguintes objectos deployment, daemonset, stateful e replicaset no
kubernetes.

Espero que esse artigo tenha sido útil pra você que mexe há pouco ou
muito tempo com Kubernetes.
