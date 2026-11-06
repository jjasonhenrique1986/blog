---
title: "Using eksctl to create and manage an EKS cluster"
slug: "2022-09-07-using-eksctl-to-create-and-manage-an-eks-cluster"
description: "How to use eksctl, the Weaveworks command-line tool, to create and manage an Amazon EKS cluster."
images: ["/images/capa-usando-eksctl-para-criacao-e-gerenciamento-de-um-cluster-eks.png"]
author: Jason
date: 2022-09-07
lastmod: 2026-11-03
tags: [ "AWS", "EKS", "Kubernetes" ]
type: post
---
> **Update (November 2026):** Weaveworks shut down
> in 2024 and eksctl is now maintained by AWS, in the
> [eksctl-io/eksctl](https://github.com/eksctl-io/eksctl) repository. The Linux
> installation command was updated to the one in the current documentation.

In this article I'll show how to use the command-line
tool called eksctl, written in Golang and created by
Weaveworks, to create and manage an EKS cluster.

With this command-line tool you can create an EKS cluster
and also manage its node groups, even if the cluster wasn't
created with eksctl (for example, a cluster provisioned with
Terraform).

**Keep in mind that EKS isn't part of the AWS free tier. So, if you**
**go ahead with this lab, you'll be charged for the time the**
**EKS cluster is running.**

## Installation

eksctl can be installed on Windows, Linux and
Mac machines, or run through Docker.

### On Linux

``` bash
ARCH=$(uname -m | sed 's/x86_64/amd64/' | sed 's/aarch64/arm64/')
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify the checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

### On Mac

Mac users can also use the following command line:

``` bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

### On Windows

``` bash
chocolatey install eksctl
```

On Windows you can also use PowerShell.

In my case I used a Linux machine. To check whether the
installation succeeded, just run the command below:

``` bash
eksctl version
```

Or typing just eksctl should return the output below:

![Output of the eksctl command without arguments, listing the available subcommands](/images/eksctl-2.webp)

For more details, see the [eksctl installation
documentation](https://eksctl.io/introduction/#installation).

## Prerequisites

- An AWS user with the permissions below:

![Table with eksctl's minimum permissions per service: CloudFormation, EC2, EC2 Auto Scaling, EKS, IAM and Systems Manager](/images/eksctl-3.webp)

For more information about the permissions eksctl needs
to work, see the [eksctl minimum IAM
policies](https://eksctl.io/usage/minimum-iam-policies/). Below is how
my IAM policies ended up:

![IAM console with the four policies attached to the user: AmazonEC2FullAccess, AWSCloudFormationFullAccess, IamLimitedAccess and EksAllAccess](/images/eksctl-4.webp)

To create a user on AWS, follow the [IAM user creation
documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html).

- AWS CLI installed. Below is the installation procedure:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Depending on the Linux distribution, you'll need to install unzip.

For more information, see the [AWS CLI installation
guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

- kubectl installed. Below is the installation procedure:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/
```

For more information, see the [kubectl installation guide for
Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

## Creating your first EKS cluster with eksctl

After installing the prerequisites from the previous section you're ready
to create your first cluster. eksctl is a simpler
way to bring up an EKS cluster: in the background, it provisions all
the infrastructure resources using **CloudFormation**.

By default eksctl launches m5.large instances. To save money,
I chose to create the cluster with the file below, which sets the
instance type to t3.medium.

``` yaml
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

Create a file named **eksctl.yml** with the content above and run the
command below:

``` bash
eksctl create cluster -f eksctl.yml
```

Below is the command running and the output of the cluster creation.

![Start of the eksctl create cluster output creating the CloudFormation stacks for the cluster-demoeksctl cluster](/images/eksctl-5.webp)

The output below shows the two Nodegroups being provisioned

![End of the eksctl create cluster output with nodegroups ng-1 and ng-2 ready and the cluster-demoeksctl cluster created](/images/eksctl-6.webp)

The two nodegroups were created, with one node each, as in the images below.

![kubectl get nodes output with the cluster's two nodes in Ready status](/images/eksctl-7.png)

![EKS console showing the active node groups ng-1 and ng-2, with one node each](/images/eksctl-8.webp)

## Creating a new Nodegroup with eksctl

With eksctl you can also manage your cluster, whether or not it was created
with eksctl. Some tasks we can do with eksctl:

- Creating a new Nodegroup;
- Creating a Fargate Profile;
- Upgrading the cluster version;

For more information, see the official eksctl documentation,
which I list in the references section.

In my case I'll show how to create a new nodegroup in the EKS cluster
created earlier. Again I'll create it declaratively
using the file below:

``` yaml
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

Create a file named **nodegroup.yml** with the content above and run
the command below:

``` bash
eksctl create nodegroup -f nodegroup.yml
```

![eksctl create nodegroup output creating the ng3 nodegroup with two nodes](/images/eksctl-9.webp)

![kubectl get nodes output with four Ready nodes after creating the ng3 nodegroup](/images/eksctl-10.png)

## Deleting the Nodegroups and the EKS cluster

To delete only the nodegroup, just run the command below:

``` bash
eksctl delete nodegroup -f nodegroup.yml --approve
```

![eksctl delete nodegroup output draining and removing the ng3 nodegroup](/images/eksctl-11.webp)

We can see in the image below that we only have the 2 nodes created along with the cluster.

![kubectl get nodes output with only the two original nodes after removing the ng3 nodegroup](/images/eksctl-12.png)

Now, to delete the EKS cluster, just run the command below:

``` bash
eksctl delete cluster -f eksctl.yml
```

![eksctl delete cluster output removing the nodegroups and the cluster-demoeksctl cluster](/images/eksctl-13.webp)

## References

- [eksctl on GitHub](https://github.com/eksctl-io/eksctl)
- [Official documentation](https://eksctl.io/)
- [Official EKS documentation](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
- [eksctl usage examples](https://github.com/eksctl-io/eksctl/tree/main/examples)
