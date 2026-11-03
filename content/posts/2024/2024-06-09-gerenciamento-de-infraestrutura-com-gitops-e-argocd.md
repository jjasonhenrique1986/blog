---
title: "Gerenciamento de Infraestrutura com GitOps e ArgoCD"
description: "Tutorial de GitOps: instalando o ArgoCD em um cluster Kubernetes com kind e fazendo o deploy de uma aplicação nginx."
image: "/images/argo.png"
author: Jason
date: 2024-06-09
tags: [ "Containers", "Kubernetes", "GitOps", "IaC" ]
type: post
showTableOfContents: true
featured: true
---

![Gerenciamento de Infraestrutura com GitOps e ArgoCD](/images/argo.png)

O objetivo desse artigo é criar um tutorial de como instalar o ArgoCD no
kubernetes usando kind e também vamos subir uma aplicação (o nosso
famoso e querido nginx) usando o ArgoCD.

## Pré-requisitos 

Para fazer a instalação do ArgoCD e acompanhar esse artigo, são
necessários os seguintes pré-requisitos:

-   Ter o helm instalado.
-   Ter o kubectl instalado.
-   Ter uma conta no GitHub.
-   Ter um cluster local usando o kind, como mostrei em [Criando um
    cluster local de kubernetes com o
    kind](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/).

## Conceitos 

Antes de partir para a parte prática, vamos dar alguns passos atrás para
entendermos sobre GitOps.

GitOps é uma forma poderosa de gerenciar e automatizar a sua
infraestrutura de forma declarativa usando um repositório git.


![Diagrama do GitOps: o desenvolvedor faz commit no Git, o ArgoCD puxa as mudanças e sincroniza o estado no Kubernetes](/images/argo-2.webp)

Basicamente, o desenvolvedor e/ou engenheiro DevOps mantém todas as
definições de infraestrutura em um repositório, e o ArgoCD faz
verificações periódicas. Caso haja alguma modificação no repositório, é
iniciado um novo deploy da aplicação no cluster Kubernetes.

Com GitOps também é possível fazer o reconcile das definições, ou seja,
caso alguém faça uma alteração manual, por exemplo alterando o número de
réplicas, o ArgoCD compara com o que está no repositório git e reverte a
alteração.

## Instalação ArgoCD 

Para seguir com a instalação, é necessário criar um cluster local usando
o kind. Depois, crie um namespace chamado argocd
conforme comando abaixo:

```bash
kubectl create namespace argocd
```

Vamos fazer a instalação do ArgoCD utilizando o Helm. Mais informações
na [página do chart do ArgoCD no Artifact
Hub](https://artifacthub.io/packages/helm/argo-cd-oci/argo-cd).

Abaixo o comando utilizado para a instalação do ArgoCD:

```bash
helm install argocd oci://ghcr.io/argoproj/argo-helm/argo-cd --version 7.1.3 -n argocd
```

A instalação será iniciada e, se tiver sucesso, será apresentada uma
tela conforme abaixo:

![Saída da instalação do ArgoCD com Helm e as instruções para acessar a interface e pegar a senha inicial do admin](/images/argo-3.webp)

Para verificar se os pods subiram corretamente podemos executar o
seguinte comando:

```bash
kubectl get pods -n argocd
```

## Acessando o ArgoCD

Nesse passo vamos recuperar a senha de admin do ArgoCD que está em um
secret executando o comando abaixo:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Você deve salvar o valor apresentado no seu terminal.

Agora vamos acessar a console do ArgoCD fazendo um port-forward
apontando para o service do ArgoCD usando o comando abaixo:

```bash
kubectl port-forward service/argocd-server -n argocd 8080:443
```

Através de qualquer navegador de Internet acessar o endereço
[https://127.0.0.1:8080](https://127.0.0.1:8080) (com **https**, porque o
port-forward aponta para a porta 443 do ArgoCD). O navegador deve
apresentar um erro de certificado, que pode ser ignorado nesse teste. Deverá ser
apresentada a tela abaixo:

![Tela de login do ArgoCD](/images/argo-4.webp)

Agora você deve usar o usuário admin e a senha recuperada através do
secret feito no passo anterior para acessar a console do ArgoCD. Deverá
aparecer uma tela igual à abaixo:

![Tela inicial do ArgoCD ainda sem aplicações](/images/argo-5.webp)

## Criando um repositório git 

Agora vamos criar um repositório git com as definições de um nginx
conforme abaixo:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
  namespace: argocd
spec:
  replicas: 2
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

Se quiser, pode usar o repositório
[nginx-example](https://github.com/jjasonhenrique/nginx-example), que
criei para esse artigo.

## Criando a conexão entre um repositório git e o ArgoCD 

Agora vamos fazer a conexão entre o servidor do ArgoCD e o seu
repositório criado no passo anterior. Para isso vamos na opção
**Settings** do lado esquerdo e depois em **Repositories** e clicar em
**Connect Repo**. Será apresentada a tela abaixo:

![Tela de conexão de repositório do ArgoCD com o método via SSH](/images/argo-6.webp)

Nessa opção você consegue adicionar tanto um repositório do git como um
repositório de helm chart.

No nosso caso, deve ser configurado conforme abaixo:

![Conexão do repositório nginx-example no ArgoCD via HTTPS](/images/argo-7.webp)

Depois, clique no botão **Connect** e será apresentada a conexão com o
repositório do GitHub, conforme abaixo:

![Repositório nginx-example conectado no ArgoCD com status Successful](/images/argo-8.webp)

## Criação da App usando a console do ArgoCD

Para criar a nossa aplicação usando o ArgoCD, vá no menu
**Applications** e depois clicar na opção **New App**
conforme imagem abaixo:

![Formulário de nova aplicação no ArgoCD com as seções General e Sync Policy](/images/argo-9.webp)

Agora vamos fazer as configurações da nossa aplicação. Abaixo, a imagem
das primeiras configurações:

![Aplicação nginx no projeto default com sync automático, prune, self heal e auto create namespace](/images/argo-10.webp)

Em **General** temos as seguintes configurações abaixo:

-   **Application Name** vai ser o nome da nossa aplicação.
-   **Project** podemos selecionar em qual projeto dentro do ArgoCD a
    nossa aplicação será criada. Como não criamos nenhum projeto vamos
    colocar no projeto **Default** que já vem criado.
-   **Sync Policy** vamos alterar para **Automatic** e habilitar as duas
    opções abaixo:
    -   **Prune Resources**: quando habilitada, o ArgoCD apaga do
        cluster os recursos que foram removidos do repositório git. Sem
        ela, um recurso apagado do git continua rodando no cluster.
    -   **Self Heal**: quando habilitada, o ArgoCD desfaz alterações
        feitas direto no cluster, como um `kubectl scale` que muda a
        quantidade de réplicas, e volta ao estado definido no git.
-   **Auto-Create Namespace**: caso o namespace não exista, ele será
    criado no cluster Kubernetes.

Agora vamos partir para a segunda parte das configurações conforme imagem
abaixo:

![Origem da aplicação apontando para o repositório nginx-example, path deploy, e destino no namespace nginx do próprio cluster](/images/argo-11.webp)

Em **Source** temos as seguintes configurações abaixo:

-   **Repository URL** será a URL do repositório git em que está a
    definição da nossa app.
-   **Revision** vamos deixar o valor default que é HEAD mas podemos
    configurar outras branches ou tags.
-   **Path** é o diretório do repositório onde está a definição do nosso
    arquivo yaml.

Em **Destination** temos as seguintes configurações abaixo:

-   **Cluster URL** vamos passar a URL do nosso cluster kubernetes como
    só temos um cluster, aparecerá apenas uma opção, mas existe a
    possibilidade de configurar mais de um cluster.
-   **Namespace** é o nome do namespace que será criado no nosso cluster
    caso não exista, e onde a aplicação será criada.

Agora basta clicar no botão **Create**, e a aplicação será apresentada
conforme abaixo:

![Aplicação nginx no ArgoCD com status Healthy e Synced](/images/argo-12.webp)

Clicando sobre o quadro da nossa aplicação você terá uma visão de todos
os objetos criados no kubernetes para esse deploy conforme abaixo:

![Árvore da aplicação nginx no ArgoCD com o deployment, o replicaset e os pods, todos saudáveis](/images/argo-13.webp)

## Criação da aplicação via linha de comando

No passo anterior criamos a nossa aplicação usando a console do ArgoCD
mas podemos também criar via linha de comando usando o kubectl. Para
isso criar um arquivo chamado app.yaml com conteúdo abaixo:

```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/jjasonhenrique/nginx-example.git'
    path: deploy
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Depois disso executar o comando abaixo:

```bash
kubectl apply -f app.yaml
```

Para verificar o estado da sua aplicação no ArgoCD você pode usar o
comando abaixo:

```bash
kubectl get apps -n argocd 
```

Deve ser apresentada uma tela similar à abaixo:

![Saída do kubectl mostrando a aplicação nginx com status Synced e Healthy](/images/argo-14.png)

## Validação

Nesse passo vamos fazer uma alteração manual na nossa aplicação
diretamente no cluster alterando o número de réplicas para 10 usando o
comando abaixo:

```bash
kubectl scale deploy/nginx --replicas 10 -n nginx
```

Você irá perceber que instantaneamente serão terminados vários pods e
ficarão apenas 2 pods como Running, conforme está configurado na
definição
que está no git. Isso é o conceito de reconcile do ArgoCD. Abaixo você
pode ver uma imagem com os pods sendo terminados:

![Pods do nginx sendo encerrados depois que o número de réplicas foi reduzido no repositório](/images/argo-15.webp)

Você também pode verificar os eventos do kubernetes usando o comando
abaixo:

```bash
kubectl get events -n nginx
```

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

O objetivo desse artigo foi explicar um pouco sobre GitOps, demonstrar a
instalação do ArgoCD e a criação de uma aplicação usando um repositório
git.

Lembrando que a forma como o ArgoCD foi instalado é apenas para testes.
Para ambiente de produção é recomendável a criação do ArgoCD no modo HA.
