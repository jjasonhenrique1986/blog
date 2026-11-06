---
title: "Creating a Mutation policy with Kyverno"
slug: "2023-11-02-creating-a-mutation-policy-with-kyverno"
description: "Continuing the Kyverno series: how to create a Mutation policy in a Kubernetes cluster."
images: ["/images/capa-criando-uma-policy-do-tipo-mutation-com-o-kyverno.png"]
author: Jason
date: 2023-11-02
tags: [ "Kubernetes", "Security", "Containers" ]
type: post
---

The goal of this article is to create a Mutation policy in
Kyverno. This article continues the first one, where I showed
how to install Kyverno and created a validation policy in
Kyverno. The first article is [Getting to know
Kyverno]({{< relref "/posts/2023/2023-10-29-conhecendo-um-pouco-sobre-o-kyverno" >}}).

## Prerequisites

To create a Mutation policy in Kyverno you'll
need the following prerequisites:

-   A Kubernetes cluster
-   kubectl installed
-   Kyverno already installed in your cluster

To install Kyverno, you can also check the first
article I wrote about Kyverno.

## Testing a Mutation policy

To test a mutation policy in Kyverno, let's create a
clusterpolicy that enforces the **environment: develop** label,
as below:

``` yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-labels
spec:
  rules:
  - name: add-labels
    match:
      any:
      - resources:
          kinds:
          - Deployment
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            environment: develop
        spec:
          template:
            metadata:
              labels:
                environment: develop
  validationFailureAction: Enforce
```

To apply the clusterpolicy, create a file called add-labels.yaml
with the content above and run the command below:

``` bash
kubectl apply -f add-labels.yaml
```

To validate that the cluster policy was created, run the command below:

``` bash
kubectl get clusterpolicy
```

Now let's create a deployment without the **environment: develop** label,
as below:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app
  name: app
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
        owner: jason
    spec:
      containers:
      - image: nginx
        name: nginx
```

To apply the deployment, create a file called nginx.yaml and run
the command below:

``` bash
kubectl apply -f nginx.yaml
```

To see whether the label was applied, you can run the commands below:

``` bash
kubectl get deploy -o jsonpath='{.items[0].metadata.labels}' -n app

kubectl get deploy -o jsonpath='{.items[0].spec.template.metadata.labels}' -n app
```

They should return the labels as below:

``` text
{"app":"app","environment":"develop"}

{"app":"app","environment":"develop","owner":"jason"}
```

## Conclusion

The goal of this article was to show how to use a
Mutation policy in Kyverno. There are other policies you can create
in the [Kyverno policy library](https://kyverno.io/policies/).
