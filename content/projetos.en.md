---
title: "Projects"
slug: "projects"
description: "Projects I build on GitHub, such as the action that reviews infrastructure pull requests with AI, and the repositories from the LINUXtips Container Architecture on AWS course (ECS, EKS and multi-region)."
type: page
showDate: false
showDateUpdated: false
showReadingTime: false
showWordCount: false
showViews: false
showLikes: false
---

These are the projects I maintain on [GitHub](https://github.com/jjasonhenrique1986)
and the lab repositories from the Container Architecture on AWS course by
[LINUXtips](https://linuxtips.io/), a Brazilian training company. When a project
has a blog post, the link is on its card.

## Projects

{{< projetos >}}

{{< projeto titulo="Infrastructure PR review with AI" repo="github-actions" pasta="iac-ai-review" tags="GitHub Actions, Amazon Bedrock, Terraform, Kubernetes, OIDC" >}}
An action that reviews the Terraform and Kubernetes changes in a pull request
with AI on Amazon Bedrock (Claude or gpt-oss) and posts the review as a PR
comment. It runs checkov, Trivy, kube-linter and kubeconform before the AI and
reaches AWS only through OIDC, with no access keys.

Blog series:
[part 1]({{< relref "/posts/2026/2026-11-01-revisao-de-pr-de-infraestrutura-com-ia-parte-1" >}}),
[part 2]({{< relref "/posts/2026/2026-11-02-revisao-de-pr-de-infraestrutura-com-ia-parte-2" >}}) and
[part 3]({{< relref "/posts/2026/2026-11-03-revisao-de-pr-de-infraestrutura-com-ia-parte-3" >}}).
{{< /projeto >}}

{{< projeto titulo="This blog" repo="jjasonhenrique1986.github.io" tags="Hugo, Blowfish, GitHub Pages, GitHub Actions" >}}
The source of this blog: Hugo with the Blowfish theme, published to GitHub
Pages by a GitHub Actions workflow.
{{< /projeto >}}

{{< projeto titulo="API on AWS App Runner" repo="apprunner-example" tags="Python, FastAPI, App Runner" >}}
A sample Python API built with FastAPI, used to test deployments on AWS App Runner.

Post: [Deploying an API with AWS App Runner]({{< relref "/posts/2023/2023-12-23-deployando-uma-api-usando-aws-app-runner" >}}).
{{< /projeto >}}

{{< projeto titulo="Subnets with for_each" repo="terraform-blog-jason" tags="Terraform, VPC" >}}
A VPC and subnets on AWS with Terraform, creating the subnets with `for_each`.

Post: [How to use the for_each expression to create subnets on AWS with Terraform]({{< relref "/posts/2024/2024-08-18-como-usar-a-expressao-for_each-para-criar-subnets-na-aws-com-terraform" >}}).
{{< /projeto >}}

{{< projeto titulo="CKA notes" repo="Anotacoes-CKA" tags="Kubernetes, CKA" >}}
My study notes for the CKA (Certified Kubernetes Administrator), in Portuguese:
pods, deployments, services, ingress, RBAC, volumes, troubleshooting, cluster
upgrades, JSONPath and Vim tips.
{{< /projeto >}}

{{< projeto titulo="nginx on Kubernetes" repo="nginx-example" tags="Kubernetes" >}}
An nginx Deployment on Kubernetes.
{{< /projeto >}}

{{< projeto titulo="Learning Python" repo="learn-python" tags="Python" >}}
Python study exercises.
{{< /projeto >}}

{{< /projetos >}}

## LINUXtips course: Container Architecture on AWS

Each part of the lab has its own repository, almost all of them with Terraform.

### Foundation and ECS

{{< projetos >}}

{{< projeto titulo="VPC and ECS cluster" repo="curso-containers-linuxtips" tags="Terraform, VPC, ECS" >}}
The first classes: the VPC (public, private and database subnets, NAT Gateway)
and the ECS cluster.
{{< /projeto >}}

{{< projeto titulo="ECS cluster with Spot and On-Demand" repo="curso-containers-linuxtips-ecs-cluster-fargate" tags="Terraform, ECS, ALB, Spot" >}}
An ECS cluster with Auto Scaling Groups and capacity providers (Spot and
On-Demand), an Application Load Balancer and security groups.
{{< /projeto >}}

{{< projeto titulo="ECS service module" repo="curso-containers-linuxtips-module-service" tags="Terraform, ECS, CloudWatch" >}}
A Terraform module for ECS services, with the task definition, an ALB target
group, autoscaling on CPU and on requests, and CloudWatch alarms.
{{< /projeto >}}

{{< projeto titulo="VPC and ECS modules" repo="curso-containers-linuxtips-aws-modules" tags="Terraform, VPC, ECS, Route 53" >}}
VPC and ECS cluster modules, with load balancers, service discovery, Route 53
and VPC Link.
{{< /projeto >}}

{{< projeto titulo="Application on ECS" repo="curso-containers-linuxtips-ecs-app" tags="Terraform, ECS, Docker" >}}
Deploying an application on ECS with the service module.
{{< /projeto >}}

{{< projeto titulo="Health API" repo="curso-containers-linuxtips-ecs-api-health" tags="Terraform, ECS, API Gateway" >}}
The Health API microservices lab on ECS, with internal and external routing
and, in v2, API Gateway with VPC Link.
{{< /projeto >}}

{{< projeto titulo="API Gateway with a custom domain" repo="curso-containers-linuxtips-ecs-api-gateway" tags="Terraform, API Gateway, ACM" >}}
API Gateway with a custom domain, an ACM certificate and API keys.
{{< /projeto >}}

{{< /projetos >}}

### Multi-region

{{< projetos >}}

{{< projeto titulo="Multi-region VPC" repo="curso-containers-linuxtips-aws-multiregion-vpc" tags="Terraform, VPC, GitHub Actions" >}}
A VPC in two regions (us-east-1 and sa-east-1).
{{< /projeto >}}

{{< projeto titulo="Multi-region ECS" repo="curso-containers-linuxtips-aws-multiregion-cluster-ecs" tags="Terraform, ECS, GitHub Actions" >}}
The ECS cluster in both regions.
{{< /projeto >}}

{{< projeto titulo="Routing between regions" repo="curso-containers-linuxtips-aws-multiregion-routing" tags="Terraform, Global Accelerator, Route 53" >}}
Routing between the regions with AWS Global Accelerator and Route 53.
{{< /projeto >}}

{{< projeto titulo="Sales workload" repo="curso-containers-linuxtips-aws-multiregion-sales-workload" tags="Terraform, ECS, SNS, SQS, S3" >}}
The sales workload (API and worker) with SNS, SQS and S3.
{{< /projeto >}}

{{< projeto titulo="Sales API Gateway" repo="curso-containers-linuxtips-aws-multiregion-sales-api-gateway" tags="Terraform, API Gateway, ACM" >}}
API Gateway for the sales application, with a custom domain and an ACM certificate.
{{< /projeto >}}

{{< /projetos >}}

### EKS

{{< projetos >}}

{{< projeto titulo="EKS networking" repo="curso-containers-linuxtips-aws-eks-networking" tags="Terraform, VPC, EKS" >}}
The EKS cluster network: VPC, NAT Gateway, network ACLs and SSM parameters.
{{< /projeto >}}

{{< projeto titulo="EKS cluster" repo="curso-containers-linuxtips-aws-eks-vanilla" tags="Terraform, EKS, Helm" >}}
An EKS cluster with node groups (Graviton, Bottlerocket and Spot), access
entries, Cluster Autoscaler, Metrics Server, kube-state-metrics and Node
Termination Handler.
{{< /projeto >}}

{{< projeto titulo="Load testing" repo="curso-containers-linuxtips-aws-eks-warmup" tags="k6, Docker" >}}
Load testing with k6.
{{< /projeto >}}

{{< /projetos >}}
