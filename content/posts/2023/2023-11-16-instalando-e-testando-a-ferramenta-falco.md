---
title: "Instalando e Testando a ferramenta Falco"
description: "Instalando e testando o Falco, ferramenta da Sysdig que detecta ameaças em near real time em Kubernetes, containers e cloud."
image: "/images/falco.png"
author: Jason
date: 2023-11-16
tags: [ "Kubernetes", "Segurança", "Containers" ]
type: post
showTableOfContents: true
---

![Instalando e Testando a ferramenta Falco](/images/falco.png)

O Falco é uma ferramenta desenvolvida pela empresa Sysdig e tem a função
de detectar ameaças em near real time na cloud, em workloads executando
no Kubernetes e em containers.

O Falco pode monitorar eventos de várias origens, como o kernel do
Linux, a API do Kubernetes e containers. Ele também se integra com
ferramentas de mensagens, como Slack e Teams, e com o Prometheus. Para
isso, é necessário usar um daemon do Falco chamado falcosidekick.

Para mais informações do falco consultar os links abaixo:

-   <https://github.com/falcosecurity/falcosidekick>
-   <https://falco.org/docs/>

O falco também é um dos tópicos abordados no exame de certificação CKS
(Certified Kubernetes Security Specialist).

## Pré-requisitos 

Nessa demonstração do funcionamento do falco vamos precisar dos
seguintes pré-requisitos:

-   Cluster Kubernetes local, que pode ser criado com o kind. Mais
    informações nesse
    [link](https://kind.sigs.k8s.io/docs/user/quick-start/). Também criei esse
    [artigo](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/)
    para ajudar no processo de criação de um cluster local.
-   Helm instalado. Mais informações nesse
    [link](https://helm.sh/docs/intro/install/).
-   kubectl instalado. Mais informações nesse
    [link](https://kubernetes.io/docs/tasks/tools/).

## Instalação 

Antes de fazer a instalação do falco você deve adicionar o repositório
do falco no helm executando os comandos abaixo:

``` bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

Também deve ser criado um namespace separado para a instalação do falco
usando o comando abaixo:

``` bash
kubectl create namespace falco
```

Agora vamos efetuar a instalação do falco executando o comando abaixo:

``` bash
helm install falco -n falco --set driver.kind=ebpf --set tty=true falcosecurity/falco
```

Para verificar se o falco subiu corretamente executar o comando abaixo:

``` bash
kubectl get pods -n falco
```

Deverá apresentar uma saída conforme a imagem abaixo:

![Saída do kubectl get pods com os três pods do Falco em execução](/images/falco-2.webp)

Será criado um DaemonSet do Falco, que executa um pod em cada node do
cluster. No meu caso, há 3 pods do Falco. Para verificar
executar o comando abaixo:

``` bash
kubectl get daemonset -n falco
```

Agora vamos verificar os logs dos pods do Falco executando
o comando abaixo:

``` bash
kubectl logs -f daemonset/falco -n falco
```

Deverá aparecer uma saída similar à abaixo:

![Logs do pod do Falco mostrando a versão 0.36.2 e o carregamento das regras](/images/falco-3.webp)

## Demonstração 

Agora vamos simular o funcionamento do falco criando um pod, conectando
nesse pod usando o kubectl e verificar os logs do falco.

Abaixo comando para criar um pod para esse teste:

``` bash
kubectl run alpine --image alpine -- sh -c "sleep infinity"
```

Agora vamos conectar no pod criado usando o comando abaixo:

``` bash
kubectl exec -it alpine -- sh
```

Verificando os logs do Falco, haverá uma saída similar à abaixo:

``` bash
15:27:42.692441989: Notice A shell was spawned in a container with an attached terminal (evt_type=execve user=root user_uid=0 user_loginuid=-1 process=sh proc_exepath=/bin/busybox parent=runc command=sh terminal=34816 exe_flags=EXE_WRITABLE container_id=cf009dcc3814 container_image=<NA> container_image_tag=<NA> container_name=<NA> k8s_ns=<NA> k8s_pod_name=<NA>)
```

Através do evento acima podemos constatar que alguém conectou no
terminal do container com o id cf009dcc3814.

## Conclusão 

O objetivo desse artigo foi fazer a instalação e a demonstração da
ferramenta falco usando um cluster local do kubernetes.

O falco também pode ser instalado em uma máquina virtual executando
Linux, como mostrei no artigo seguinte:
[Testando o falco em uma máquina virtual](/posts/2023/2023-11-22-testando-o-falco-em-uma-maquina-virtual/).