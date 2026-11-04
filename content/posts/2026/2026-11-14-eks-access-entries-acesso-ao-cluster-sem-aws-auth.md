---
title: "EKS access entries: acesso ao cluster sem editar o aws-auth"
description: "Como dar acesso ao Kubernetes no Amazon EKS com access entries e access policies, pela AWS CLI e pelo Terraform, e como migrar as entradas do ConfigMap aws-auth."
images: ["/images/capa-eks-access-entries.png"]
author: Jason
date: 2026-11-14
tags: [ "AWS", "EKS", "Kubernetes", "Segurança" ]
type: post
showTableOfContents: true
---

Durante muito tempo, a única forma de dar a uma role ou usuário do IAM
acesso ao Kubernetes de um cluster EKS era editar o ConfigMap `aws-auth`,
no namespace `kube-system`. Foi o que fiz no post
[Configurando permissão somente leitura em um cluster EKS](/posts/2023/2023-11-11-configurando-permissao-readonly-em-um-cluster-eks/).

Esse modelo tem alguns problemas conhecidos:

- A configuração de acesso fica **dentro** do cluster. Para mudar quem
  acessa o cluster, você precisa já ter acesso a ele.
- Um erro de YAML no `aws-auth` pode tirar o acesso de todo mundo, inclusive
  dos nodes.
- As mudanças são feitas pela API do Kubernetes, não pela API da AWS. Não
  dá para controlar quem pode alterar o acesso com policies do IAM, e as
  alterações não aparecem no CloudTrail.

As **access entries** resolvem isso: o acesso passa a ser configurado pela
API do EKS, como qualquer outro recurso da AWS. Neste post vamos ver os
conceitos, o passo a passo pela AWS CLI e pelo Terraform, e como migrar o
que já está no `aws-auth`.

## Conceitos

Uma **access entry** liga uma identidade do IAM (normalmente uma role) a um
cluster. Sozinha, ela só diz que aquela identidade pode se autenticar no
cluster. As permissões dentro do Kubernetes vêm de uma de duas formas:

1. **Access policies**: conjuntos de permissões do Kubernetes mantidos
   pela AWS, que você associa à access entry.
2. **Grupos do Kubernetes**: você coloca a access entry em um ou mais
   grupos e cria as Roles, ClusterRoles e bindings com RBAC, como no post
   [Gerenciando acesso no Kubernetes com RBAC](/posts/2024/2024-11-15-gerenciando-acesso-no-kubernetes-com-rbac/).

Cada associação de access policy tem um **escopo**: o cluster inteiro
(`cluster`) ou uma lista de namespaces (`namespace`).

As access entries também têm um **tipo**. Para pessoas e pipelines, o tipo
é `STANDARD`, que é o padrão. Os outros tipos, como `EC2_LINUX` e
`FARGATE_LINUX`, são para as roles dos nodes e dos perfis do Fargate.

### As access policies mais usadas

| Access policy | O que permite |
|---|---|
| `AmazonEKSClusterAdminPolicy` | Administrador do cluster inteiro |
| `AmazonEKSAdminPolicy` | Quase tudo, normalmente com escopo de namespaces |
| `AmazonEKSEditPolicy` | Editar a maioria dos recursos |
| `AmazonEKSViewPolicy` | Ver a maioria dos recursos, **sem** Secrets |
| `AmazonEKSAdminViewPolicy` | Ver todos os recursos, **incluindo** Secrets |

O ARN segue o formato
`arn:aws:eks::aws:cluster-access-policy/NOME`, por exemplo
`arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy`. A lista
completa, com os recursos e verbos de cada uma, está na
[documentação das access policies](https://docs.aws.amazon.com/eks/latest/userguide/access-policy-permissions.html).

Repare na diferença entre a `AmazonEKSViewPolicy` e a
`AmazonEKSAdminViewPolicy`: para quem só precisa acompanhar os workloads,
a primeira é a escolha certa, porque não deixa ler Secrets.

## Modo de autenticação do cluster

O cluster precisa estar num modo de autenticação que aceite a API de access
entries:

- `CONFIG_MAP`: só o `aws-auth`. É o modo dos clusters antigos.
- `API_AND_CONFIG_MAP`: access entries e `aws-auth` ao mesmo tempo. É o
  modo para fazer a migração.
- `API`: só access entries.

Para ver o modo atual:

```bash
aws eks describe-cluster --name meu-cluster \
    --query cluster.accessConfig.authenticationMode --output text
```

E para mudar para o modo de migração:

```bash
aws eks update-cluster-config --name meu-cluster \
    --access-config authenticationMode=API_AND_CONFIG_MAP
```

O cluster fica em `UPDATING` por alguns minutos. Atenção: a mudança é de mão
única. Depois que o cluster aceita a API, não é possível voltar para
`CONFIG_MAP`.

Clusters mais antigos precisam de uma versão mínima de plataforma para
suportar access entries. A tabela fica na página
[Change authentication mode](https://docs.aws.amazon.com/eks/latest/userguide/setting-up-access-entries.html).

## Dando acesso somente leitura a uma role

Vamos refazer o exemplo do post antigo: uma role da AWS com acesso
somente leitura ao cluster, agora sem tocar no `aws-auth`.

Primeiro, crie a access entry da role:

```bash
aws eks create-access-entry --cluster-name meu-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-leitura \
    --type STANDARD
```

Depois, associe a access policy de leitura com escopo do cluster inteiro:

```bash
aws eks associate-access-policy --cluster-name meu-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-leitura \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
    --access-scope type=cluster
```

Se a role só deve ver alguns namespaces, troque o escopo:

```bash
aws eks associate-access-policy --cluster-name meu-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-leitura \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
    --access-scope type=namespace,namespaces=app1,app2
```

Para conferir o que foi configurado:

```bash
aws eks list-access-entries --cluster-name meu-cluster

aws eks list-associated-access-policies --cluster-name meu-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-leitura
```

### Testando o acesso

Quem vai usar a role gera o kubeconfig já assumindo a role:

```bash
aws eks update-kubeconfig --name meu-cluster \
    --role-arn arn:aws:iam::111122223333:role/eks-leitura
```

E o `kubectl auth can-i` mostra na hora o que é permitido:

```bash
kubectl auth can-i list pods --all-namespaces
kubectl auth can-i create deployments -n app1
kubectl auth can-i get secrets -n app1
```

Com a `AmazonEKSViewPolicy`, a primeira pergunta deve responder `yes` e as
outras duas, `no`.

## Usando grupos do Kubernetes em vez de access policy

Se as access policies não atendem, por exemplo porque você quer uma
ClusterRole com uma lista bem específica de recursos, crie a access entry
com um grupo e não associe access policy:

```bash
aws eks create-access-entry --cluster-name meu-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-leitura \
    --kubernetes-groups leitura-restrita
```

E crie no cluster o ClusterRole e o ClusterRoleBinding que usam o grupo
`leitura-restrita` como `subject`. As duas formas podem ser combinadas: se a
access entry tiver access policies e grupos, a identidade recebe a soma das
permissões.

## Com Terraform

No Terraform, o modo de autenticação fica no bloco `access_config` do
`aws_eks_cluster`:

```hcl
resource "aws_eks_cluster" "this" {
  name     = "meu-cluster"
  role_arn = aws_iam_role.cluster.arn

  access_config {
    authentication_mode                         = "API"
    bootstrap_cluster_creator_admin_permissions = false
  }

  vpc_config {
    subnet_ids = var.subnet_ids
  }
}
```

O `bootstrap_cluster_creator_admin_permissions` controla se a identidade
que criou o cluster ganha automaticamente acesso de administrador. O padrão
é `true`. Com `false`, ninguém ganha acesso implícito e todo acesso fica
declarado no código, como abaixo:

```hcl
resource "aws_eks_access_entry" "leitura" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = aws_iam_role.leitura.arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "leitura" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = aws_iam_role.leitura.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"

  access_scope {
    type = "cluster"
  }

  depends_on = [aws_eks_access_entry.leitura]
}
```

O `depends_on` garante que a access entry exista antes da associação da
policy.

## Migrando o aws-auth

Para um cluster que já usa o `aws-auth`, o caminho é:

1. Mudar o cluster para `API_AND_CONFIG_MAP`.
2. Listar o que existe no ConfigMap:

   ```bash
   eksctl get iamidentitymapping --cluster meu-cluster
   ```

3. Criar uma access entry para cada role ou usuário que **você** adicionou,
   com as permissões equivalentes. As entradas das roles dos node groups
   gerenciados e dos perfis do Fargate foram criadas pelo próprio EKS:
   confira se as access entries delas já existem antes de mexer nelas, ou
   os nodes param de conseguir entrar no cluster.
4. Remover do ConfigMap as entradas que já viraram access entries:

   ```bash
   eksctl delete iamidentitymapping --cluster meu-cluster \
       --arn arn:aws:iam::111122223333:role/eks-leitura
   ```

5. Quando o ConfigMap não tiver mais nada que você use, mudar o cluster para
   `API`.

Durante a migração, se a mesma role estiver no `aws-auth` e numa access
entry, vale a access entry.

## Conclusão

Com access entries, o acesso ao cluster vira uma configuração da AWS como
qualquer outra: dá para criar pela CLI ou pelo Terraform, auditar pelo
CloudTrail e recuperar o acesso mesmo que a configuração dentro do
Kubernetes esteja errada. Para clusters novos, o modo `API` com
`bootstrap_cluster_creator_admin_permissions = false` deixa todo acesso
explícito no código.

Access entries cuidam do acesso de **pessoas e pipelines** ao Kubernetes.
Para dar permissões da AWS aos **pods**, veja
[EKS Pod Identity](/posts/2025/2025-10-18-eks-pod-identity-permissoes-aws-para-pods/).

## Referências

- [Grant IAM users access to Kubernetes with EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Review access policy permissions](https://docs.aws.amazon.com/eks/latest/userguide/access-policy-permissions.html)
- [Migrating existing aws-auth ConfigMap entries to access entries](https://docs.aws.amazon.com/eks/latest/userguide/migrating-access-entries.html)
- [aws_eks_access_entry no Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_access_entry)
