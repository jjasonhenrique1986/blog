---
title: "EKS Pod Identity: AWS permissions for pods without access keys"
slug: "2025-10-18-eks-pod-identity-aws-permissions-for-pods"
description: "How to use EKS Pod Identity to give IAM permissions to pods on Amazon EKS without access keys, and how it differs from IRSA."
images: ["/images/capa-eks-pod-identity-permissoes-aws-para-pods.png"]
author: Jason
date: 2025-10-18
tags: [ "AWS", "EKS", "Kubernetes", "Security" ]
type: post
featured: true
---

The goal of this article is to show how to use **EKS Pod Identity**
so that a pod running on Amazon EKS can access AWS services, such as S3,
**without putting access keys** inside the cluster.

Putting `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in a Secret
works, but they're long-lived credentials that can leak and
need to be rotated. With Pod Identity the pod receives **temporary**
credentials from an IAM role, and which role each application
uses is defined by the association with the Kubernetes ServiceAccount.

## Prerequisites

-   An EKS cluster. If you need to create one, see the article about
    [eksctl]({{< relref "/posts/2022/2023-09-07-usando-eksctl-para-criacao-e-gerenciamento-de-um-cluster-eks" >}}).
-   EC2 nodes (managed node groups or self-managed). Pod
    Identity is **not supported for pods on Fargate**.
-   A recent version of AWS CLI v2, which has the
    `aws eks create-pod-identity-association` command.
-   kubectl configured for the cluster.

In the examples the cluster is called `my-cluster` and the account ID is
`111122223333`. Replace them with your own values.

## How it works

1.  The **EKS Pod Identity Agent** add-on runs as a DaemonSet on each
    node.
2.  You create an IAM role that trusts the
    `pods.eks.amazonaws.com` service.
3.  You create an **association** between the role and a ServiceAccount in a
    namespace.
4.  When a pod with that ServiceAccount is created, EKS injects
    environment variables that make the AWS SDKs fetch credentials
    from the local agent, which in turn gets the role's temporary
    credentials.

## Installing the add-on

```bash
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name eks-pod-identity-agent
```

Check that the agent pods are running:

```bash
kubectl get pods -n kube-system | grep eks-pod-identity-agent
```

## Creating the IAM role

Create a file called trust-pod-identity.json. Notice the two actions:
Pod Identity needs `sts:AssumeRole` and also `sts:TagSession`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "pods.eks.amazonaws.com" },
      "Action": [ "sts:AssumeRole", "sts:TagSession" ]
    }
  ]
}
```

Now create the role and, for the example, attach the managed S3
read policy:

```bash
aws iam create-role \
  --role-name eks-s3-reader \
  --assume-role-policy-document file://trust-pod-identity.json

aws iam attach-role-policy \
  --role-name eks-s3-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

In production, prefer your own policy that allows only the buckets and
actions the application really needs.

## Creating the ServiceAccount and the association

```bash
kubectl create namespace app
kubectl create serviceaccount s3-reader -n app
```

Unlike IRSA, **you don't need to annotate the ServiceAccount**. The link
is made by the association:

```bash
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace app \
  --service-account s3-reader \
  --role-arn arn:aws:iam::111122223333:role/eks-s3-reader
```

To list the cluster's associations:

```bash
aws eks list-pod-identity-associations --cluster-name my-cluster
```

## Testing

Create a file called aws-cli.yaml with a pod using the official
AWS CLI image and the ServiceAccount we created:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  namespace: app
spec:
  serviceAccountName: s3-reader
  containers:
  - name: aws-cli
    image: amazon/aws-cli:latest
    command: ["sleep", "infinity"]
```

```bash
kubectl apply -f aws-cli.yaml
```

First check the pod's identity:

```bash
kubectl exec -n app aws-cli -- aws sts get-caller-identity
```

The `Arn` field should show an `assumed-role/eks-s3-reader/...`. Now
list the account's buckets:

```bash
kubectl exec -n app aws-cli -- aws s3 ls
```

If you try an action the role doesn't allow, such as creating a bucket, the
response will be `AccessDenied`, which confirms the pod is using only
the role's permissions.

The credentials are injected **when the pod is created**. If you create the
association after the pod is already running, you need to recreate the pod
(for example with `kubectl rollout restart` on a Deployment).

## Pod Identity or IRSA?

**IRSA** (IAM Roles for Service Accounts) was the recommended approach
before Pod Identity and still works. The main differences:

| | IRSA | EKS Pod Identity |
|---|---|---|
| OIDC provider | You need to create one per cluster in IAM | Not needed |
| Trust policy | References a specific cluster's OIDC | Trusts `pods.eks.amazonaws.com`; the same role works for several clusters |
| ServiceAccount | Needs the `eks.amazonaws.com/role-arn` annotation | No annotation; the association is made through the EKS API |
| Fargate | Supported | Not supported |
| Outside EKS | Works on other clusters with OIDC | EKS only |

Pod Identity also adds **session tags** with information such as the
cluster name, the namespace and the ServiceAccount. They can be used in
policies for attribute-based access control (ABAC).

## Cleanup

```bash
kubectl delete namespace app

aws eks delete-pod-identity-association \
  --cluster-name my-cluster \
  --association-id <id-returned-by-list>

aws iam detach-role-policy \
  --role-name eks-s3-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name eks-s3-reader
```

## Conclusion

With EKS Pod Identity, applications in the cluster access AWS with
temporary credentials and the principle of least privilege, without
access keys scattered around in Secrets. The setup is simpler than
IRSA's, especially when you have several clusters.

To go further, also see how to
[configure read-only permission on an EKS cluster]({{< relref "/posts/2023/2023-11-11-configurando-permissao-readonly-em-um-cluster-eks" >}}),
which covers the opposite direction: giving an AWS role access to Kubernetes.

## References

-   [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
-   [Set up the EKS Pod Identity Agent](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-agent-setup.html)
