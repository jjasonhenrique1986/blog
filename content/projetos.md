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
e os repositórios dos laboratórios que fiz no curso de Arquitetura de Containers na AWS da
[LINUXtips](https://linuxtips.io/). Quando um projeto tem post no blog, o link
está junto.

## Projetos

### github-actions: revisão de PR de infraestrutura com IA

Actions reutilizáveis do GitHub. A principal é a **`iac-ai-review`**, que revisa
o Terraform e o Kubernetes de um Pull Request com IA no Amazon Bedrock (Claude
ou gpt-oss) e comenta no PR. Acesso à AWS só por OIDC, sem access keys, e
desenvolvimento guiado por specs com o GitHub Spec Kit.

- Repositório: [jjasonhenrique1986/github-actions](https://github.com/jjasonhenrique1986/github-actions)
- Série no blog: [parte 1](/posts/2026/2026-11-01-revisao-de-pr-de-infraestrutura-com-ia-parte-1/),
  [parte 2](/posts/2026/2026-11-02-revisao-de-pr-de-infraestrutura-com-ia-parte-2/) e
  [parte 3](/posts/2026/2026-11-03-revisao-de-pr-de-infraestrutura-com-ia-parte-3/)
- O projeto nasceu no repositório [iac-ai-reviewer](https://github.com/jjasonhenrique1986/iac-ai-reviewer),
  hoje arquivado: o código, com todo o histórico, mudou para a action.

### Este blog

O código deste blog: Hugo com o tema Blowfish, publicado no GitHub Pages por um
workflow do GitHub Actions.

- Repositório: [jjasonhenrique1986.github.io](https://github.com/jjasonhenrique1986/jjasonhenrique1986.github.io)

### apprunner-example

API de exemplo em Python com FastAPI, usada para testar o deploy no AWS App Runner.

- Repositório: [apprunner-example](https://github.com/jjasonhenrique1986/apprunner-example)
- Post: [Deployando uma API usando AWS App Runner](/posts/2023/2023-12-23-deployando-uma-api-usando-aws-app-runner/)

### terraform-blog-jason

VPC e subnets na AWS com Terraform, criando as subnets com `for_each`.

- Repositório: [terraform-blog-jason](https://github.com/jjasonhenrique1986/terraform-blog-jason)
- Post: [Como usar a expressão for_each para criar subnets na AWS com Terraform](/posts/2024/2024-08-18-como-usar-a-expressao-for_each-para-criar-subnets-na-aws-com-terraform/)

### Anotações da CKA

Minhas anotações de estudo para a certificação CKA (Certified Kubernetes
Administrator): pods, deployments, services, ingress, RBAC, volumes,
troubleshooting, upgrade de cluster, JSONPath e dicas de Vim.

- Repositório: [Anotacoes-CKA](https://github.com/jjasonhenrique1986/Anotacoes-CKA)

### Outros exemplos

- [nginx-example](https://github.com/jjasonhenrique1986/nginx-example): Deployment de um nginx no Kubernetes.
- [learn-python](https://github.com/jjasonhenrique1986/learn-python): exercícios de estudo de Python.

## Curso da LINUXtips

### Arquitetura de Containers na AWS

Cada parte do laboratório tem o seu repositório, quase todos com Terraform.

**Fundação e ECS**

- [curso-containers-linuxtips](https://github.com/jjasonhenrique1986/curso-containers-linuxtips): primeiras aulas, com a VPC (subnets públicas, privadas e de banco, NAT Gateway) e o cluster ECS.
- [curso-containers-linuxtips-ecs-cluster-fargate](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-ecs-cluster-fargate): cluster ECS com Auto Scaling Groups e capacity providers (Spot e On-Demand), Application Load Balancer e security groups.
- [curso-containers-linuxtips-module-service](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-module-service): módulo Terraform de serviço ECS, com task definition, target group no ALB, autoscaling por CPU e por requisições e alarmes no CloudWatch.
- [curso-containers-linuxtips-aws-modules](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-aws-modules): módulos de VPC e de cluster ECS (load balancers, service discovery, Route 53 e VPC Link).
- [curso-containers-linuxtips-ecs-app](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-ecs-app): deploy de uma aplicação no ECS com o módulo de serviço.
- [curso-containers-linuxtips-ecs-api-health](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-ecs-api-health): lab de microserviços do Health API no ECS, com roteamento interno e externo e, na v2, API Gateway com VPC Link.
- [curso-containers-linuxtips-ecs-api-gateway](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-ecs-api-gateway): API Gateway com domínio próprio, certificado no ACM e API keys.

**Multi-região**

- [curso-containers-linuxtips-aws-multiregion-vpc](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-aws-multiregion-vpc): VPC em duas regiões (us-east-1 e sa-east-1).
- [curso-containers-linuxtips-aws-multiregion-cluster-ecs](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-aws-multiregion-cluster-ecs): cluster ECS nas duas regiões.
- [curso-containers-linuxtips-aws-multiregion-routing](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-aws-multiregion-routing): roteamento entre as regiões com AWS Global Accelerator e Route 53.
- [curso-containers-linuxtips-aws-multiregion-sales-workload](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-aws-multiregion-sales-workload): workload de vendas (API e worker) com SNS, SQS e S3.
- [curso-containers-linuxtips-aws-multiregion-sales-api-gateway](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-aws-multiregion-sales-api-gateway): API Gateway da aplicação de vendas, com domínio próprio e certificado no ACM.

**EKS**

- [curso-containers-linuxtips-aws-eks-networking](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-aws-eks-networking): rede do cluster EKS (VPC, NAT Gateway, network ACLs e parâmetros no SSM).
- [curso-containers-linuxtips-aws-eks-vanilla](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-aws-eks-vanilla): cluster EKS com node groups (Graviton, Bottlerocket e Spot), access entries, Cluster Autoscaler, Metrics Server, kube-state-metrics e Node Termination Handler.
- [curso-containers-linuxtips-aws-eks-warmup](https://github.com/jjasonhenrique1986/curso-containers-linuxtips-aws-eks-warmup): teste de carga com k6.
