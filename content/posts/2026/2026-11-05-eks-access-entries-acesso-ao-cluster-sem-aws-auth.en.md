---
title: "EKS access entries: cluster access without editing aws-auth"
slug: "2026-11-05-eks-access-entries-cluster-access-without-aws-auth"
description: "How to grant Kubernetes access on Amazon EKS with access entries and access policies, using the AWS CLI and Terraform, and how to migrate the entries from the aws-auth ConfigMap."
images: ["/images/capa-eks-access-entries.png"]
author: Jason
date: 2026-11-05
tags: [ "AWS", "EKS", "Kubernetes", "Security" ]
type: post
---

For a long time, the only way to give an IAM role or user
access to Kubernetes on an EKS cluster was to edit the `aws-auth` ConfigMap
in the `kube-system` namespace. That's what I did in the post
[Configuring read-only permission on an EKS cluster]({{< relref "/posts/2023/2023-11-11-configurando-permissao-readonly-em-um-cluster-eks" >}}).

This model has a few well-known problems:

- The access configuration lives **inside** the cluster. To change who
  can access the cluster, you already need access to it.
- A YAML mistake in `aws-auth` can lock everyone out, including
  the nodes.
- Changes go through the Kubernetes API, not the AWS API. You can't
  control who can change access with IAM policies, and the
  changes don't show up in CloudTrail.

**Access entries** fix this: access is configured through the
EKS API, like any other AWS resource. In this post we'll look at the
concepts, the step by step with the AWS CLI and with Terraform, and how to migrate
what is already in `aws-auth`.

## Concepts

An **access entry** links an IAM identity (usually a role) to a
cluster. On its own, it only says that this identity can authenticate to the
cluster. Permissions inside Kubernetes come in one of two ways:

1. **Access policies**: sets of Kubernetes permissions maintained
   by AWS, which you associate with the access entry.
2. **Kubernetes groups**: you put the access entry in one or more
   groups and create the Roles, ClusterRoles and bindings with RBAC, as in the post
   [Managing access in Kubernetes with RBAC]({{< relref "/posts/2024/2024-11-15-gerenciando-acesso-no-kubernetes-com-rbac" >}}).

Each access policy association has a **scope**: the whole cluster
(`cluster`) or a list of namespaces (`namespace`).

Access entries also have a **type**. For people and pipelines, the type
is `STANDARD`, which is the default. The other types, such as `EC2_LINUX` and
`FARGATE_LINUX`, are for node roles and Fargate profiles.

### The most used access policies

| Access policy | What it allows |
|---|---|
| `AmazonEKSClusterAdminPolicy` | Administrator of the whole cluster |
| `AmazonEKSAdminPolicy` | Almost everything, usually scoped to namespaces |
| `AmazonEKSEditPolicy` | Edit most resources |
| `AmazonEKSViewPolicy` | View most resources, **without** Secrets |
| `AmazonEKSAdminViewPolicy` | View all resources, **including** Secrets |

The ARN follows the format
`arn:aws:eks::aws:cluster-access-policy/NAME`, for example
`arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy`. The full
list, with the resources and verbs of each one, is in the
[access policies documentation](https://docs.aws.amazon.com/eks/latest/userguide/access-policy-permissions.html).

Notice the difference between `AmazonEKSViewPolicy` and
`AmazonEKSAdminViewPolicy`: for someone who only needs to follow the workloads,
the first one is the right choice, because it doesn't allow reading Secrets.

## Cluster authentication mode

The cluster has to be in an authentication mode that accepts the access
entries API:

- `CONFIG_MAP`: only `aws-auth`. This is the mode of older clusters.
- `API_AND_CONFIG_MAP`: access entries and `aws-auth` at the same time. This is the
  mode for the migration.
- `API`: access entries only.

To see the current mode:

```bash
aws eks describe-cluster --name my-cluster \
    --query cluster.accessConfig.authenticationMode --output text
```

And to switch to the migration mode:

```bash
aws eks update-cluster-config --name my-cluster \
    --access-config authenticationMode=API_AND_CONFIG_MAP
```

The cluster stays in `UPDATING` for a few minutes. Beware: the change is one-way.
Once the cluster accepts the API, you can't go back to
`CONFIG_MAP`.

Older clusters need a minimum platform version to
support access entries. The table is on the
[Change authentication mode](https://docs.aws.amazon.com/eks/latest/userguide/setting-up-access-entries.html) page.

## Giving a role read-only access

Let's redo the example from the old post: an AWS role with read-only
access to the cluster, now without touching `aws-auth`.

First, create the role's access entry:

```bash
aws eks create-access-entry --cluster-name my-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-readonly \
    --type STANDARD
```

Then associate the read access policy scoped to the whole cluster:

```bash
aws eks associate-access-policy --cluster-name my-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-readonly \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
    --access-scope type=cluster
```

If the role should only see some namespaces, change the scope:

```bash
aws eks associate-access-policy --cluster-name my-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-readonly \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
    --access-scope type=namespace,namespaces=app1,app2
```

To check what was configured:

```bash
aws eks list-access-entries --cluster-name my-cluster

aws eks list-associated-access-policies --cluster-name my-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-readonly
```

### Testing the access

Whoever is going to use the role generates the kubeconfig already assuming the role:

```bash
aws eks update-kubeconfig --name my-cluster \
    --role-arn arn:aws:iam::111122223333:role/eks-readonly
```

And `kubectl auth can-i` shows right away what is allowed:

```bash
kubectl auth can-i list pods --all-namespaces
kubectl auth can-i create deployments -n app1
kubectl auth can-i get secrets -n app1
```

With `AmazonEKSViewPolicy`, the first question should answer `yes` and the
other two, `no`.

## Using Kubernetes groups instead of an access policy

If the access policies don't fit, for example because you want a
ClusterRole with a very specific list of resources, create the access entry
with a group and don't associate an access policy:

```bash
aws eks create-access-entry --cluster-name my-cluster \
    --principal-arn arn:aws:iam::111122223333:role/eks-readonly \
    --kubernetes-groups restricted-read
```

Then create in the cluster the ClusterRole and the ClusterRoleBinding that use the
`restricted-read` group as the `subject`. The two approaches can be combined: if the
access entry has access policies and groups, the identity gets the sum of the
permissions.

## With Terraform

In Terraform, the authentication mode goes in the `access_config` block of
`aws_eks_cluster`:

```hcl
resource "aws_eks_cluster" "this" {
  name     = "my-cluster"
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

`bootstrap_cluster_creator_admin_permissions` controls whether the identity
that created the cluster automatically gets administrator access. The default
is `true`. With `false`, nobody gets implicit access and every access is
declared in code, like below:

```hcl
resource "aws_eks_access_entry" "readonly" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = aws_iam_role.readonly.arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "readonly" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = aws_iam_role.readonly.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"

  access_scope {
    type = "cluster"
  }

  depends_on = [aws_eks_access_entry.readonly]
}
```

`depends_on` guarantees that the access entry exists before the policy
association.

## Migrating aws-auth

For a cluster that already uses `aws-auth`, the path is:

1. Switch the cluster to `API_AND_CONFIG_MAP`.
2. List what exists in the ConfigMap:

   ```bash
   eksctl get iamidentitymapping --cluster my-cluster
   ```

3. Create an access entry for each role or user that **you** added,
   with the equivalent permissions. The entries for the managed node group roles
   and the Fargate profiles were created by EKS itself:
   check whether their access entries already exist before touching them, or
   the nodes will no longer be able to join the cluster.
4. Remove from the ConfigMap the entries that are now access entries:

   ```bash
   eksctl delete iamidentitymapping --cluster my-cluster \
       --arn arn:aws:iam::111122223333:role/eks-readonly
   ```

5. When the ConfigMap has nothing left that you use, switch the cluster to
   `API`.

During the migration, if the same role is in `aws-auth` and in an access
entry, the access entry wins.

## Conclusion

With access entries, cluster access becomes an AWS configuration like
any other: you can create it with the CLI or Terraform, audit it through
CloudTrail and recover access even if the configuration inside
Kubernetes is wrong. For new clusters, the `API` mode with
`bootstrap_cluster_creator_admin_permissions = false` keeps every access
explicit in code.

Access entries take care of **people and pipelines** accessing Kubernetes.
To give AWS permissions to **pods**, see
[EKS Pod Identity]({{< relref "/posts/2025/2025-10-18-eks-pod-identity-permissoes-aws-para-pods" >}}).

## References

- [Grant IAM users access to Kubernetes with EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Review access policy permissions](https://docs.aws.amazon.com/eks/latest/userguide/access-policy-permissions.html)
- [Migrating existing aws-auth ConfigMap entries to access entries](https://docs.aws.amazon.com/eks/latest/userguide/migrating-access-entries.html)
- [aws_eks_access_entry on the Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_access_entry)
