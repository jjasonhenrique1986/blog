---
title: "Configuring read-only permission on an EKS cluster"
slug: "2023-11-11-configuring-read-only-permission-on-an-eks-cluster"
description: "How to configure an AWS role with read-only permission on an EKS cluster using kubectl."
images: ["/images/capa-configurando-permissao-readonly-em-um-cluster-eks.png"]
author: Jason
date: 2023-11-11
lastmod: 2026-11-03
tags: [ "Kubernetes", "AWS", "Containers", "EKS" ]
type: post
---

> **Update (November 2026):** this post uses the `aws-auth` ConfigMap,
> which AWS considers deprecated. Today the recommended approach is **EKS
> access entries**, which don't require editing the ConfigMap. See the
> [Current alternative: EKS access entries](#current-alternative-eks-access-entries)
> section at the end of the post. For pod permissions (not people's), see
> [EKS Pod Identity]({{< relref "/posts/2025/2025-10-18-eks-pod-identity-permissoes-aws-para-pods" >}}).

The goal of this post is to show how to configure
an AWS role to have read-only permission inside
an EKS cluster using kubectl, the utility to manage
Kubernetes from the command line.

## Prerequisites

To follow this demo, you'll need the following prerequisites:

-   An AWS account.
-   An EKS cluster already created. To create an EKS cluster
    you can follow the post [Using eksctl to create and manage
    an EKS
    cluster]({{< relref "/posts/2022/2023-09-07-usando-eksctl-para-criacao-e-gerenciamento-de-um-cluster-eks" >}}).
-   [kubectl
    installed](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).
-   [AWS CLI
    installed](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

**Keep in mind that this demo will incur costs on AWS.**

## Creating the AWS user and role

First, we need to create an IAM user without any permissions,
following the [IAM user creation
documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html).
This user must have CLI access.

Before creating the role, we need to create a policy that has
only list and read permissions on EKS, as below:

``` json
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

Now let's create a role with the policy above attached. Another point of
attention is that we need to create a trust relationship so the IAM
user created earlier can assume the role. Below is how this trust
relationship should be configured:

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::account-id:user/jason"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

Where **account-id** is your account ID.

For more information, see the [documentation on roles for IAM
users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user.html).

## Changing the aws-auth file

The first thing we need to do is add the AWS role to the
aws-auth configmap inside the EKS cluster. To do that, using a role with
administrative permissions, we'll run the following command to
edit this configmap:

``` bash
eksctl create iamidentitymapping --cluster cluster-name --region=us-east-1 \
    --arn arn:aws:iam::account-id:role/eks --username {{SessionName}} --group system:masters
```

To check it, we can run the command below:

``` bash
eksctl get iamidentitymapping --cluster cluster-name --region=us-east-1
```

This configuration can also be done manually, following the
[EKS documentation about
aws-auth](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html).

## Configuring the AWS profile

In this step we'll configure the IAM user's credentials,
create a new profile referencing the role we created and adjust the
kubeconfig.

First, let's configure the IAM user we created with the command
below, which will ask for the access key, secret access key, region and
output format:

``` bash
aws configure
```

This information can be found in the .aws/credentials file, with
content similar to the one below:

``` ini
[default]
aws_access_key_id = XXXXXX
aws_secret_access_key = XXXXXX
```

Then let's configure a profile in the .aws/config file

``` ini
[profile eks]
role_arn = arn:aws:iam::account-id:role/eks
source_profile = default
```

In **source_profile**, use the name registered in the
.aws/credentials file.

Now let's adjust the kubeconfig to reference the profile we created. In the
users section of the kubeconfig file, add the
--profile
eks argument, as below:

``` yaml
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

## Configuring read-only permission in Kubernetes

After configuring the previous item, I found that, even using
an AWS role with a read-only profile, it was still possible to change
resources with kubectl.

I had to create a ClusterRole, a ClusterRoleBinding and a group
with list, get and watch permissions on all Kubernetes
resources, as below:

``` yaml
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

Apply the content above by creating a file called readonly.yaml and
running the command below:

``` bash
kubectl apply -f readonly.yaml
```

After that, remove the entry below from the aws-auth
configmap:

``` bash
eksctl delete iamidentitymapping --cluster cluster-name --region=us-east-1 --arn arn:aws:iam::account-id:role/eks --all
```

With a user with administrative permissions, check that the AWS role was
removed:

``` bash
eksctl get iamidentitymapping --cluster cluster-name --region=us-east-1
```

Then add the AWS role with the group created earlier, together with the
clusterrole and clusterrolebinding, by running the command below:

``` bash
eksctl create iamidentitymapping --cluster cluster-name --region=us-east-1 --arn arn:aws:iam::account-id:role/eks --username {{SessionName}} --group developers-readonly
```

## Validating the access

Now, to validate that the AWS role really has read-only
access, let's try to create a secret in the cluster with the command below:

``` bash
kubectl create secret generic teste --from-literal=user=jason
```

It should show the following error, as in the image below:

![Permission error when trying to create a secret with the read-only role](/images/eks-2.webp)

Then we try to list all pods running in the cluster and it
works, as in the image below:

![Listing the cluster's pods working with the read-only role](/images/eks-3.webp)

With this we make sure access to Kubernetes objects is
read-only.

A word of caution: since the ClusterRole above allows `get` and `list` on all
resources (`resources: ["*"]`), this role can also read the cluster's
**Secrets**. If that isn't what you want, list the resources explicitly
instead of using `*`. To better understand Roles, ClusterRoles and
bindings, see the post about
[RBAC in Kubernetes]({{< relref "/posts/2024/2024-11-15-gerenciando-acesso-no-kubernetes-com-rbac" >}}).

## Current alternative: EKS access entries

With access entries, cluster access is configured through the EKS API,
without editing `aws-auth`. First, the cluster's authentication mode
needs to accept the API. The `API_AND_CONFIG_MAP` mode keeps `aws-auth`
working during the migration:

``` bash
aws eks update-cluster-config --name cluster-name --region us-east-1 \
    --access-config authenticationMode=API_AND_CONFIG_MAP
```

Then create an access entry for the role:

``` bash
aws eks create-access-entry --cluster-name cluster-name --region us-east-1 \
    --principal-arn arn:aws:iam::account-id:role/eks --type STANDARD
```

And associate the managed read access policy,
**AmazonEKSViewPolicy**, with it, scoped to the whole cluster:

``` bash
aws eks associate-access-policy --cluster-name cluster-name --region us-east-1 \
    --principal-arn arn:aws:iam::account-id:role/eks \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
    --access-scope type=cluster
```

To limit it to some namespaces, use
`--access-scope type=namespace,namespaces=app1,app2`. To check what
is associated with the role:

``` bash
aws eks list-associated-access-policies --cluster-name cluster-name --region us-east-1 \
    --principal-arn arn:aws:iam::account-id:role/eks
```

With this you don't need to create the ClusterRole and ClusterRoleBinding from the
previous section. If you prefer to keep your own ClusterRole, create the access entry
with `--kubernetes-groups developers-readonly` and don't associate an access policy.
More details in the
[access entries documentation](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html).

## Conclusion

The goal of this article was to show how to grant read-only
permission to Kubernetes resources by linking them to an AWS role.

Another way to grant read-only access is to use a graphical
interface that shows all the EKS cluster's resources, such as the Kubernetes
Dashboard (which was
archived; today [Headlamp](https://github.com/kubernetes-sigs/headlamp)
is a maintained alternative).
