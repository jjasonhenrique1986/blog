---
title: "Managing access in Kubernetes with RBAC"
slug: "2024-11-15-managing-access-in-kubernetes-with-rbac"
description: "How to manage access control in Kubernetes with RBAC: Roles, ClusterRoles and their bindings."
images: ["/images/capa-gerenciando-acesso-no-kubernetes-com-rbac.png"]
author: Jason
date: 2024-11-15
tags: [ "Kubernetes", "Security", "Containers" ]
type: post
featured: true
---

In Kubernetes, security and access control are critical aspects
to make sure only the right people and processes have the
permissions they need to interact with the cluster. To manage
these permissions, Kubernetes uses a system based on **RBAC
(Role-Based Access Control)**.

In this post we'll explore the Kubernetes objects whose
main job is this access
control.

## Prerequisites

To follow this article, you'll need:

-   A Kubernetes cluster; in our case, we'll use kind. To
    create one, see [how to create a local cluster with kind]({{< relref "/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind" >}}).
-   kubectl installed.

For the examples we'll create a namespace where our tests
will run. To create this namespace, run the command below:

``` bash
kubectl create ns app
```

## Service Account

A **Service Account** in Kubernetes is used to
represent the identity of a process or pod running inside the
cluster. It's needed when a pod or application has to interact
with the Kubernetes API, whether to list pods, create objects or
perform other operations.

By default, pods use the `default` service account of the namespace
they're in, but it's good practice to create separate
service accounts for different applications or pods with specific
needs. This helps isolate permissions and follow the principle of
least privilege.

### Example

In this example we'll create a file called serviceaccount.yaml with the
content below:

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: app
```

To apply the example above, run the command below:

``` bash
kubectl apply -f serviceaccount.yaml
```

To check that the serviceaccount was created, run the command below:

``` bash
kubectl get serviceaccount -n app
```

To see more details about the serviceaccount you created, you can also run the
commands below:

```bash
kubectl get serviceaccount serviceaccountname -n app -o yaml

kubectl describe serviceaccount serviceaccountname -n app
```

With the serviceaccount you can also interact with AWS resources
using annotations, as in the example below:

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: example-serviceaccount
  namespace: app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ExampleIAMRole
```

## Role

A **Role** is a set of permissions that applies to a specific
namespace. You can use it to define what a user, a group or
a service account can do inside that namespace, such as create,
list or delete resources.

### Example

In this example we'll create a file called role.yaml with the content
below:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: app
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

In this example, the `pod-reader` role grants read permission
(`get`, `list`, `watch`) on pods and deployments inside the app
namespace.

To apply the example above, run the command below:

```bash
kubectl apply -f role.yaml
```

To see the role that was created, run the command below:

``` bash
kubectl get role -n app
```

To see more details about your role, run the command below:

``` bash
kubectl describe role rolename -n app
```

## ClusterRole

A **ClusterRole** is similar to a Role, with the difference that
it isn't restricted to a single namespace. A ClusterRole can define
permissions for resources outside namespaces, such as nodes and
volumes, or even define global permissions across multiple
namespaces.

### Example

In this example we'll create a file called clusterrole.yaml with the
content below:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: teste-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
```

In the example above, the teste-clusterrole ClusterRole grants read access to
pods and nodes across the whole cluster. Notice that in this example
we don't need to set the namespace field.

To apply the example above, run the command below:

``` bash
kubectl apply -f clusterrole.yaml
```

To see the clusterroles in the cluster, run the command below:

``` bash
kubectl get clusterrole
```

All of the cluster's ClusterRoles will be shown. To see
information about the clusterrole we created, we can run the command
below:

``` bash
kubectl get clusterrole teste-clusterrole
```

To show more detailed information, run one of the commands below:

``` bash
kubectl get clusterrole clusterrolename -o yaml

kubectl describe clusterrole clusterrolename
```

## RoleBinding

A **RoleBinding** associates a Role with a user, group or service
account inside a specific namespace. The RoleBinding is the mechanism
that actually grants the permissions defined in the Role to the target
entity, in our case our serviceaccount.

### Example

In this example we'll create a file called rolebinding.yaml with the
content below:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly
  namespace: app
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: app
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Here, the `RoleBinding` associates the `Role` called `pod-reader` with the
`ServiceAccount` called `my-service-account`, making sure this service
account has permission to read pods and deployments inside the
app namespace.

To apply the example above, run the command below:

```bash
kubectl apply -f rolebinding.yaml
```

To see the rolebinding that was created, run the command below:

``` bash
kubectl get rolebinding -n app
```

To see more information, run the command below:

``` bash
kubectl describe rolebinding -n app
```

## ClusterRoleBinding

Similar to the RoleBinding, a **ClusterRoleBinding** associates a
**ClusterRole** with a user, group or service account, but with
cluster-wide scope. This means the permissions granted by the
ClusterRole will be available in all namespaces.

### Example

In this example we'll create a file called clusterrolebinding.yaml with
the content below:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: teste-clusterrole-binding
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: app
roleRef:
  kind: ClusterRole
  name: teste-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

In this example, the `teste-clusterrole-binding` ClusterRoleBinding
associates the `teste-clusterrole` ClusterRole with the `ServiceAccount`
`my-service-account`, giving the service account permission to read
pods and nodes across the whole cluster.

To apply the example above, run the command below:

``` bash
kubectl apply -f clusterrolebinding.yaml 
```

To see the clusterrolebindings in the cluster, run the command
below:

``` bash
kubectl get clusterrolebinding
```

To show more detailed information, run one of the commands below:

``` bash
kubectl get clusterrolebinding clusterrolebindingname -o yaml

kubectl describe clusterrolebinding clusterrolebindingname
```

## Validating the permissions

To validate our serviceaccount's permissions we can run the
command below:

```bash
sudo kubectl auth can-i list pods --as=system:serviceaccount:app:my-service-account -n app
```

If the result is yes, your ServiceAccount has permission to perform
that action.

You can run the command below to confirm that your
ServiceAccount doesn't have permission to perform another action:

``` bash
sudo kubectl auth can-i delete deployments --as=system:serviceaccount:app:my-service-account -n app
```

## Using our ServiceAccount

Now that we know all about RBAC, it's time to create our
deployment with our new ServiceAccount, making our
Kubernetes cluster more secure.

Below is an example deployment using the serviceaccount we
created:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: app
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: my-service-account
      containers:
      - name: nginx
        image: nginx:1.25.2
        ports:
        - containerPort: 80
```

Create a file called deploy.yaml with the content above and run the
command below:

``` bash
kubectl apply -f deploy.yaml
```

To confirm the ServiceAccount is linked to the deployment, run the
command below:

``` bash
kubectl get deploy -n app -o yaml | grep serviceAccount
```

## Conclusion

The goal of this post was to demonstrate how to use RBAC in your
cluster, making sure users and processes have controlled
and secure access to the resources they need, helping to prevent
unwanted access and security flaws.

On EKS, RBAC is also used to give an AWS role read-only
access to the cluster, as I showed in
[Configuring read-only permission on an EKS cluster]({{< relref "/posts/2023/2023-11-11-configurando-permissao-readonly-em-um-cluster-eks" >}}).
And to restrict who can read Secrets, see
[ConfigMaps and Secrets in practice]({{< relref "/posts/2025/2025-08-16-kubernetes-configmaps-e-secrets-na-pratica" >}}).

Below are some links to the Kubernetes documentation about RBAC:

-   <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
-   <https://kubernetes.io/docs/concepts/security/rbac-good-practices/>
