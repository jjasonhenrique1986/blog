---
title: "Usando eksctl para criação e gerenciamento de um cluster EKS"
description: "Como usar o eksctl, ferramenta de linha de comando da Weaveworks, para criar e gerenciar um cluster Amazon EKS."
image: "/images/eksctl.png"
author: Jason
date: 2022-09-07
tags: [ "AWS", "EKS", "Kubernetes" ]
type: post
showTableOfContents: true
---
![Usando eksctl para criação e gerenciamento de um cluster EKS](/images/eksctl.png)

> **Atualização (novembro de 2026):** a Weaveworks encerrou as atividades
> em 2024 e o eksctl passou a ser mantido pela AWS, no repositório
> [eksctl-io/eksctl](https://github.com/eksctl-io/eksctl). O comando de
> instalação no Linux foi atualizado para o que está na documentação atual.

Nesse artigo irei demonstrar a utilização da ferramenta de linha de
comando chamada eksctl, feita em Golang e criada pela empresa
Weaveworks, para criação e gerenciamento de um cluster EKS.

Com essa ferramenta de linha de comando é possível criar um cluster EKS
e também gerenciar seus node groups, mesmo que o cluster não tenha sido
criado com o eksctl (por exemplo, um cluster provisionado pelo
Terraform).

**Lembrando que o EKS não está na oferta de free tier da AWS. Então, caso**
**prossiga com esse lab você será cobrado pelo tempo de utilização do**
**cluster EKS**

## Instalação 

O eksctl pode ser instalado em máquinas que utilizam Windows, Linux e
Mac, ou ser executado através do Docker.

### Para Linux

``` bash
ARCH=$(uname -m | sed 's/x86_64/amd64/' | sed 's/aarch64/arm64/')
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Opcional) Verificar o checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

### Para Mac

Também pode ser utilizada a seguinte linha de comando para usuários Mac:

``` bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

### Para Windows

``` bash
chocolatey install eksctl
```

No Windows também pode ser utilizado o PowerShell.

No meu caso eu utilizei uma máquina Linux. Para verificar se a
instalação foi bem-sucedida, basta executar o comando abaixo:

``` bash
eksctl version
```

Ou digitando apenas eksctl deve retornar a saída abaixo:

![Saída do comando eksctl sem argumentos, listando os subcomandos disponíveis](/images/eksctl-2.webp)

Para mais detalhes, consulte a [documentação de instalação do
eksctl](https://eksctl.io/introduction/#installation).

## Pré-requisitos 

- Usuário AWS com as permissões abaixo:

![Tabela com as permissões mínimas do eksctl por serviço: CloudFormation, EC2, EC2 Auto Scaling, EKS, IAM e Systems Manager](/images/eksctl-3.webp)

Para mais informações sobre as permissões necessárias para o eksctl
funcionar, veja as [políticas mínimas de IAM do
eksctl](https://eksctl.io/usage/minimum-iam-policies/). Abaixo, como
ficaram as minhas políticas no IAM:

![Console do IAM com as quatro policies anexadas ao usuário: AmazonEC2FullAccess, AWSCloudFormationFullAccess, IamLimitedAccess e EksAllAccess](/images/eksctl-4.webp)

Para criar um usuário na AWS, siga a [documentação de criação de
usuários do
IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html).

- AWS CLI instalado. Segue abaixo procedimento para a instalação:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Dependendo da distribuição Linux será necessário instalar o unzip.

Para mais informações, consulte o [guia de instalação do AWS
CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

- Kubectl instalado. Segue abaixo procedimento para a instalação:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"chmod +x ./kubectlmv ./kubectl /usr/local/bin/ 
```

Para mais informações, consulte o [guia de instalação do kubectl no
Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

## Criando seu primeiro cluster EKS com eksctl 

Após a instalação dos pré-requisitos da seção anterior você está pronto
para a criação do seu primeiro cluster. O eksctl é uma forma mais
simples de subir um cluster EKS: em background, ele provisiona todos
os recursos de infraestrutura utilizando **CloudFormation**.

Por padrão o eksctl sobe instâncias do tipo m5.large. Para economizar,
optei por criar o cluster com o arquivo abaixo, que define o tipo de
instância t3.medium.

``` bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: cluster-demoeksctl              
  region: us-east-1                       

nodeGroups:
  - name: ng-1                
    instanceType: t3.medium   
    desiredCapacity: 1        
  - name: ng-2
    instanceType: t3.medium 
    desiredCapacity: 1
```

Criar um arquivo com o nome **eksctl.yml** com conteúdo acima e executar o
comando abaixo:

``` bash
eksctl create cluster -f eksctl.yml
```

Abaixo a execução do comando e a saída para criação do cluster.

![Início da saída do eksctl create cluster criando as stacks do CloudFormation do cluster cluster-demoeksctl](/images/eksctl-5.webp)

Na saída abaixo é mostrado o provisionamento dos dois Nodegroups

![Fim da saída do eksctl create cluster com os nodegroups ng-1 e ng-2 prontos e o cluster cluster-demoeksctl criado](/images/eksctl-6.webp)

Foram criados os dois nodegroups, com um node cada, conforme as imagens abaixo.

![Saída do kubectl get nodes com os dois nodes do cluster no status Ready](/images/eksctl-7.png)

![Console do EKS mostrando os node groups ng-1 e ng-2 ativos, com um node cada](/images/eksctl-8.webp)

## Criando um novo Nodegroup com eksctl 

Com o eksctl você consegue também gerenciar seu cluster criado ou não
pelo eksctl. Algumas tarefas que conseguimos executar com o eksctl:

- Criação de um novo Nodegroup;
- Criação de um Fargate Profile;
- Update de versão do cluster;

Para mais informações, consulte a documentação oficial do eksctl
que vou deixar na seção de referências.

No meu caso irei demonstrar a criação de um novo nodegroup no cluster EKS
criado anteriormente. Novamente vou fazer a criação de forma declarativa
usando o arquivo abaixo:

``` bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: cluster-demoeksctl            
  region: us-east-1 

nodeGroups:
  - name: ng-3               
    instanceType: t3.medium     
    desiredCapacity: 1  
```

Criar um arquivo com o nome **nodegroup.yml** com conteúdo acima e executar
o comando abaixo:

``` bash
eksctl create nodegroup -f nodegroup.yml
```

![Saída do eksctl create nodegroup criando o nodegroup ng3 com dois nodes](/images/eksctl-9.webp)

![Saída do kubectl get nodes com quatro nodes Ready depois da criação do nodegroup ng3](/images/eksctl-10.png)

## Deletando os Nodegroups e o cluster EKS

Para deletar apenas o nodegroup basta executar o comando abaixo:

``` bash
eksctl delete nodegroup -f nodegroup.yml --approve
```

![Saída do eksctl delete nodegroup drenando e removendo o nodegroup ng3](/images/eksctl-11.webp)

Podemos perceber na imagem abaixo que temos apenas 2 nodes que foram criados juntamente com o cluster.

![Saída do kubectl get nodes com apenas os dois nodes originais depois de remover o nodegroup ng3](/images/eksctl-12.png)

Agora para deletar o cluster EKS criado basta executar o comando abaixo:

``` bash
eksctl delete cluster -f eksctl.yml
```

![Saída do eksctl delete cluster removendo os nodegroups e o cluster cluster-demoeksctl](/images/eksctl-13.webp)

## Referências 

- [GitHub eksctl](https://github.com/eksctl-io/eksctl)
- [Documentação Oficial](https://eksctl.io/)
- [Documentação Oficial EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
- [Exemplos de utilização do eksctl](https://github.com/eksctl-io/eksctl/tree/main/examples)
