---
title: "Projetos"
description: "Projetos que desenvolvo no GitHub, como a action de revisão de PR de infraestrutura com IA, e os repositórios do curso de Arquitetura de Containers na AWS da LINUXtips (ECS, EKS e multi-região)."
type: page
showDate: false
showDateUpdated: false
showReadingTime: false
showWordCount: false
showViews: false
showLikes: false
---

Aqui ficam os projetos que mantenho no [GitHub](https://github.com/jjasonhenrique1986)
e os repositórios dos laboratórios que fiz no curso de Arquitetura de Containers
na AWS da [LINUXtips](https://linuxtips.io/). Quando um projeto tem post no
blog, o link está no card.

## Projetos

{{< projetos >}}

{{< projeto titulo="Revisão de PR de infraestrutura com IA" repo="github-actions" pasta="iac-ai-review" tags="GitHub Actions, Amazon Bedrock, Terraform, Kubernetes, OIDC" >}}
Action que revisa as mudanças de Terraform e Kubernetes de um Pull Request com
IA no Amazon Bedrock (Claude ou gpt-oss) e publica a revisão como comentário no
PR. Roda checkov, Trivy, kube-linter e kubeconform antes da IA e acessa a AWS
só por OIDC, sem access keys.

Série no blog:
[parte 1](/posts/2026/2026-11-01-revisao-de-pr-de-infraestrutura-com-ia-parte-1/),
[parte 2](/posts/2026/2026-11-02-revisao-de-pr-de-infraestrutura-com-ia-parte-2/) e
[parte 3](/posts/2026/2026-11-03-revisao-de-pr-de-infraestrutura-com-ia-parte-3/).
{{< /projeto >}}

{{< projeto titulo="Este blog" repo="jjasonhenrique1986.github.io" tags="Hugo, Blowfish, GitHub Pages, GitHub Actions" >}}
O código deste blog: Hugo com o tema Blowfish, publicado no GitHub Pages por um
workflow do GitHub Actions.
{{< /projeto >}}

{{< projeto titulo="API no AWS App Runner" repo="apprunner-example" tags="Python, FastAPI, App Runner" >}}
API de exemplo em Python com FastAPI, usada para testar o deploy no AWS App Runner.

Post: [Deployando uma API usando AWS App Runner](/posts/2023/2023-12-23-deployando-uma-api-usando-aws-app-runner/).
{{< /projeto >}}

{{< projeto titulo="Subnets com for_each" repo="terraform-blog-jason" tags="Terraform, VPC" >}}
VPC e subnets na AWS com Terraform, criando as subnets com `for_each`.

Post: [Como usar a expressão for_each para criar subnets na AWS com Terraform](/posts/2024/2024-08-18-como-usar-a-expressao-for_each-para-criar-subnets-na-aws-com-terraform/).
{{< /projeto >}}

{{< projeto titulo="Anotações da CKA" repo="Anotacoes-CKA" tags="Kubernetes, CKA" >}}
Minhas anotações de estudo para a CKA (Certified Kubernetes Administrator):
pods, deployments, services, ingress, RBAC, volumes, troubleshooting, upgrade de
cluster, JSONPath e dicas de Vim.
{{< /projeto >}}

{{< projeto titulo="nginx no Kubernetes" repo="nginx-example" tags="Kubernetes" >}}
Deployment de um nginx no Kubernetes.
{{< /projeto >}}

{{< projeto titulo="Estudos de Python" repo="learn-python" tags="Python" >}}
Exercícios de estudo de Python.
{{< /projeto >}}

{{< /projetos >}}

## Curso da LINUXtips: Arquitetura de Containers na AWS

Cada parte do laboratório tem o seu repositório, quase todos com Terraform.

### Fundação e ECS

{{< projetos >}}

{{< projeto titulo="VPC e cluster ECS" repo="curso-containers-linuxtips" tags="Terraform, VPC, ECS" >}}
Primeiras aulas: a VPC (subnets públicas, privadas e de banco, NAT Gateway) e o
cluster ECS.
{{< /projeto >}}

{{< projeto titulo="Cluster ECS com Spot e On-Demand" repo="curso-containers-linuxtips-ecs-cluster-fargate" tags="Terraform, ECS, ALB, Spot" >}}
Cluster ECS com Auto Scaling Groups e capacity providers (Spot e On-Demand),
Application Load Balancer e security groups.
{{< /projeto >}}

{{< projeto titulo="Módulo de serviço ECS" repo="curso-containers-linuxtips-module-service" tags="Terraform, ECS, CloudWatch" >}}
Módulo Terraform de serviço ECS, com task definition, target group no ALB,
autoscaling por CPU e por requisições e alarmes no CloudWatch.
{{< /projeto >}}

{{< projeto titulo="Módulos de VPC e ECS" repo="curso-containers-linuxtips-aws-modules" tags="Terraform, VPC, ECS, Route 53" >}}
Módulos de VPC e de cluster ECS, com load balancers, service discovery,
Route 53 e VPC Link.
{{< /projeto >}}

{{< projeto titulo="Aplicação no ECS" repo="curso-containers-linuxtips-ecs-app" tags="Terraform, ECS, Docker" >}}
Deploy de uma aplicação no ECS usando o módulo de serviço.
{{< /projeto >}}

{{< projeto titulo="Health API" repo="curso-containers-linuxtips-ecs-api-health" tags="Terraform, ECS, API Gateway" >}}
Lab de microserviços do Health API no ECS, com roteamento interno e externo e,
na v2, API Gateway com VPC Link.
{{< /projeto >}}

{{< projeto titulo="API Gateway com domínio próprio" repo="curso-containers-linuxtips-ecs-api-gateway" tags="Terraform, API Gateway, ACM" >}}
API Gateway com domínio próprio, certificado no ACM e API keys.
{{< /projeto >}}

{{< /projetos >}}

### Multi-região

{{< projetos >}}

{{< projeto titulo="VPC multi-região" repo="curso-containers-linuxtips-aws-multiregion-vpc" tags="Terraform, VPC, GitHub Actions" >}}
VPC em duas regiões (us-east-1 e sa-east-1).
{{< /projeto >}}

{{< projeto titulo="ECS multi-região" repo="curso-containers-linuxtips-aws-multiregion-cluster-ecs" tags="Terraform, ECS, GitHub Actions" >}}
Cluster ECS nas duas regiões.
{{< /projeto >}}

{{< projeto titulo="Roteamento entre regiões" repo="curso-containers-linuxtips-aws-multiregion-routing" tags="Terraform, Global Accelerator, Route 53" >}}
Roteamento entre as regiões com AWS Global Accelerator e Route 53.
{{< /projeto >}}

{{< projeto titulo="Workload de vendas" repo="curso-containers-linuxtips-aws-multiregion-sales-workload" tags="Terraform, ECS, SNS, SQS, S3" >}}
Workload de vendas (API e worker) com SNS, SQS e S3.
{{< /projeto >}}

{{< projeto titulo="API Gateway de vendas" repo="curso-containers-linuxtips-aws-multiregion-sales-api-gateway" tags="Terraform, API Gateway, ACM" >}}
API Gateway da aplicação de vendas, com domínio próprio e certificado no ACM.
{{< /projeto >}}

{{< /projetos >}}

### EKS

{{< projetos >}}

{{< projeto titulo="Rede do EKS" repo="curso-containers-linuxtips-aws-eks-networking" tags="Terraform, VPC, EKS" >}}
Rede do cluster EKS: VPC, NAT Gateway, network ACLs e parâmetros no SSM.
{{< /projeto >}}

{{< projeto titulo="Cluster EKS" repo="curso-containers-linuxtips-aws-eks-vanilla" tags="Terraform, EKS, Helm" >}}
Cluster EKS com node groups (Graviton, Bottlerocket e Spot), access entries,
Cluster Autoscaler, Metrics Server, kube-state-metrics e Node Termination Handler.
{{< /projeto >}}

{{< projeto titulo="Teste de carga" repo="curso-containers-linuxtips-aws-eks-warmup" tags="k6, Docker" >}}
Teste de carga com k6.
{{< /projeto >}}

{{< /projetos >}}
