---
title: "Configurando permissão somente leitura em um cluster EKS"
description: "Como configurar uma role da AWS com permissão somente leitura em um cluster EKS usando o kubectl."
image: "/images/eks.png"
author: Jason
date: 2023-11-11
tags: [ "Kubernetes", "AWS", "Containers", "EKS" ]
type: post
showTableOfContents: true
---

![Configurando permissão somente leitura em um cluster EKS](/images/eks.png)

> **Atualização (novembro de 2026):** este post usa o ConfigMap `aws-auth`,
> que a AWS considera obsoleto. Hoje o caminho recomendado são as **EKS
> access entries**, que dispensam editar o ConfigMap. Veja a seção
> [Alternativa atual: EKS access entries](#alternativa-atual-eks-access-entries)
> no fim do post. Para permissões de pods (e não de pessoas), veja
> [EKS Pod Identity](/posts/2025/2025-10-18-eks-pod-identity-permissoes-aws-para-pods/).

O objetivo desse post é demonstrar como fazer a configuração para que
uma determinada role da AWS tenha permissão de somente leitura dentro de
um cluster EKS usando o kubectl que é um utilitário para administrar o
Kubernetes através de linha de comando.

## Pré-requisitos 

Para seguir esta demonstração, são necessários os seguintes pré-requisitos:

-   Conta na AWS.
-   Um cluster EKS já criado. Para fazer a criação de um cluster EKS
    você pode usar esse
    [artigo](/posts/2022/2023-09-07-usando-eksctl-para-criacao-e-gerenciamento-de-um-cluster-eks/)
    que falei sobre o eksctl.
-   Kubectl instalado. Mais informações nesse
    [link](https://kubernetes.io/pt-br/docs/tasks/tools/install-kubectl-linux/).
-   AWS CLI instalado. Mais informações nesse
    [link](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/getting-started-install.html).

**Lembrando que essa demonstração terá custo na AWS.**

## Criação de user e role AWS 

Primeiramente, devemos criar um usuário IAM sem nenhuma permissão, conforme esse
[link](https://docs.aws.amazon.com/pt_br/IAM/latest/UserGuide/id_users_create.html)
lembrando que esse usuário deve ter acesso via CLI.

Antes de criar a role, é necessário criar uma policy que tenha
apenas permissão de list e read no EKS conforme abaixo:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "eks:ListNodegroups",
                "eks:ListEksAnywhereSubscriptions",
                "eks:DescribeFargateProfile",
                "eks:DescribeAddonConfiguration",
                "eks:ListTagsForResource",
                "eks:ListAddons",
                "eks:DescribeEksAnywhereSubscription",
                "eks:DescribeAddon",
                "eks:ListFargateProfiles",
                "eks:DescribeNodegroup",
                "eks:DescribeIdentityProviderConfig",
                "eks:ListUpdates",
                "eks:DescribeUpdate",
                "eks:AccessKubernetesApi",
                "eks:DescribeCluster",
                "eks:ListClusters",
                "eks:DescribeAddonVersions",
                "eks:ListIdentityProviderConfigs"
            ],
            "Resource": "*"
        }
    ]
}
```

Agora vamos criar uma role com a policy acima anexada. Outro ponto de
atenção é que devemos criar uma relação de confiança para que o usuário
IAM criado anteriormente possa assumir a role. Abaixo, como deve ser
configurada essa relação de confiança:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::id-conta:user/jason"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

Onde **id-conta** é o ID da sua conta.

Para mais informações sobre essas configurações acessar esse
[link](https://docs.aws.amazon.com/pt_br/IAM/latest/UserGuide/id_roles_create_for-user.html).

## Alteração arquivo aws-auth 

O primeiro passo que devemos fazer é a inclusão da role AWS no configmap
aws-auth dentro do cluster EKS. Para isso, usando uma role com
permissões administrativas, vamos executar o seguinte comando para
editar esse configmap:

``` bash
eksctl create iamidentitymapping --cluster nome-cluster --region=us-east-1 \
    --arn arn:aws:iam::id-conta:role/eks --username {{SessionName}} --group system:masters
```

Para verificação podemos executar o comando abaixo:

``` bash
eksctl get iamidentitymapping --cluster nome-cluster --region=us-east-1
```

Essa configuração também pode ser feita manualmente conforme informado
nesse
[link](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/add-user-role.html).

## Configuração de Profile AWS 

Nesse passo vamos fazer a configuração das credenciais do usuário IAM,
criação de um novo profile referenciando a role criada e acertar a
configuração do kubeconfig.

Primeiramente, vamos configurar o usuário IAM criado usando o comando
abaixo onde será solicitado o access key, secret access key, region e
output format:

``` bash
aws configure
```

Essas informações podem ser consultadas no arquivo .aws/credentials, com
um conteúdo similar ao abaixo:

``` bash
[default]
aws_access_key_id = XXXXXX
aws_secret_access_key = XXXXXX
```

Depois vamos configurar um profile no arquivo .aws/config

``` bash
[profile eks]
role_arn = arn:aws:iam::id-conta:role/eks
source_profile = default
```

Onde **source_profile** deverá ser informado o nome cadastrado no
arquivo .aws/credentials

Agora vamos acertar o kubeconfig referenciando o profile criado. Na
seção users do arquivo kubeconfig, deve ser incluído o argumento
--profile
eks conforme abaixo:

``` bash
users:
- name: eksctl@cluster-demoeksctl.us-east-1.eksctl.io
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - eks
      - get-token
      - --output
      - json
      - --cluster-name
      - cluster-demoeksctl
      - --region
      - us-east-1
      - --profile
      - eks
      command: aws
      env:
      - name: AWS_STS_REGIONAL_ENDPOINTS
        value: regional
      provideClusterInfo: false
```

## Configuração de permissão somente leitura no Kubernetes 

Após a configuração do item anterior foi verificado que, mesmo usando
uma role AWS com perfil apenas de leitura, ainda era possível alterar
recursos usando o kubectl.

Foi necessário criar um ClusterRole, um ClusterRoleBinding e um group
com permissões de list, get e watch em todos os recursos do
Kubernetes, conforme abaixo:

``` bash
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developers-readonly
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developers-readonly
subjects:
- kind: Group
  name: developers-readonly
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developers-readonly
  apiGroup: rbac.authorization.k8s.io
```

Aplicar o conteúdo acima criando um arquivo chamado readonly.yaml e
executar o comando abaixo:

``` bash
kubectl apply -f readonly.yaml
```

Após isso, é necessário remover a entrada abaixo no configmap
aws-auth:

``` bash
eksctl delete iamidentitymapping --cluster nome-cluster --region=us-east-1 --arn arn:aws:iam::id-conta:role/eks --all
```

Com um usuário com permissão administrativa verificar se a role AWS foi
removida:

``` bash
eksctl get iamidentitymapping --cluster nome-cluster --region=us-east-1
```

Depois adicionar a role AWS com o Group criado anteriormente junto com
clusterrole e clusterrolebinding executando o comando abaixo:

``` bash
eksctl create iamidentitymapping --cluster nome-cluster --region=us-east-1 --arn arn:aws:iam::id-conta:role/eks --username {{SessionName}} --group developers-readonly
```

## Validação de acesso 

Agora para validar se a role AWS está realmente com acesso somente
leitura tentamos criar um secret no cluster usando o comando abaixo:

``` bash
kubectl create secret generic teste --from-literal=user=jason
```

Deve ser apresentado o seguinte erro conforme imagem abaixo:

![Erro de permissão ao tentar criar um secret usando a role somente leitura](/images/eks-2.webp)

Depois tentamos listar todos os pods executando no cluster e obtivemos
sucesso conforme imagem abaixo:

![Listagem dos pods do cluster funcionando com a role somente leitura](/images/eks-3.webp)

Com isso garantimos que o acesso está somente leitura aos objetos do
Kubernetes.

Um cuidado: como o ClusterRole acima libera `get` e `list` em todos os
recursos (`resources: ["*"]`), essa role também consegue ler os
**Secrets** do cluster. Se isso não for desejado, liste explicitamente os
recursos em vez de usar `*`. Para entender melhor Roles, ClusterRoles e
bindings, veja o post sobre
[RBAC no Kubernetes](/posts/2024/2024-11-15-gerenciando-acesso-no-kubernetes-com-rbac/).

## Alternativa atual: EKS access entries

Com as access entries, o acesso ao cluster é configurado pela API do EKS,
sem editar o `aws-auth`. Primeiro, o modo de autenticação do cluster
precisa aceitar a API. O modo `API_AND_CONFIG_MAP` mantém o `aws-auth`
funcionando durante a migração:

``` bash
aws eks update-cluster-config --name nome-cluster --region us-east-1 \
    --access-config authenticationMode=API_AND_CONFIG_MAP
```

Depois, crie uma access entry para a role:

``` bash
aws eks create-access-entry --cluster-name nome-cluster --region us-east-1 \
    --principal-arn arn:aws:iam::id-conta:role/eks --type STANDARD
```

E associe a ela a access policy gerenciada de leitura,
**AmazonEKSViewPolicy**, no escopo do cluster inteiro:

``` bash
aws eks associate-access-policy --cluster-name nome-cluster --region us-east-1 \
    --principal-arn arn:aws:iam::id-conta:role/eks \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
    --access-scope type=cluster
```

Para limitar a alguns namespaces, use
`--access-scope type=namespace,namespaces=app1,app2`. Para conferir o que
está associado à role:

``` bash
aws eks list-associated-access-policies --cluster-name nome-cluster --region us-east-1 \
    --principal-arn arn:aws:iam::id-conta:role/eks
```

Com isso não é preciso criar o ClusterRole e o ClusterRoleBinding da seção
anterior. Se preferir manter um ClusterRole próprio, crie a access entry
com `--kubernetes-groups developers-readonly` e não associe access policy.
Mais detalhes na
[documentação de access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html).

## Conclusão 

O objetivo desse artigo foi demonstrar como conceder permissão somente
leitura aos recursos do Kubernetes vinculando com uma role AWS.

Outra forma de conceder acesso somente leitura é usar uma interface
gráfica que mostre todos os recursos do cluster EKS, como o Kubernetes
Dashboard (que foi
arquivado; hoje o [Headlamp](https://github.com/kubernetes-sigs/headlamp)
é uma alternativa mantida).
